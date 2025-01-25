---
title: 【并发】Java和Go的并发编程对比
date: 2023-06-16 19:54:06
categories: 并发
tags: 并发

---

Go使用的是`CSP`并发模型，而JAVA使用的是基于传统的内存访问控制的并发模型。他们有以下区别。我们通过示例代码来比对一下Go和 Java的并发编程

<!-- more --> 

“Communicating Sequential Processes” (CSP) 是一种并发计算的数学模型，最早由计算机科学家Tony Hoare于1978年提出。CSP 的主要目标是描述并发系统中进程之间的交互和通信方式，而不涉及共享内存的概念。与传统的内存访问控制有很大的区别，下面是一些主要的区别：

1. **通信方式：**
   - **CSP：** 使用进程之间的明确通信来实现协同工作。进程通过发送和接收消息进行通信，但它们并不共享内存空间。
   - **传统内存访问控制：** 多个进程可能在同一块内存中进行读写操作，通过共享内存来实现通信。这可能导致诸如竞态条件和死锁等问题。
2. **并发模型：**
   - **CSP：** 采用事件驱动的方式，进程之间通过消息传递进行通信，以实现并发。并发在这里是通过协作和通信而非共享状态来实现的。
   - **传统内存访问控制：** 并发通常是通过多个进程或线程共享同一块内存来实现的。这可能引入一系列并发控制问题，如锁和同步。
3. **数据共享：**
   - **CSP：** 鼓励避免共享数据，而是通过消息传递来传递必要的信息。这样设计有助于减少竞态条件和提高系统的可靠性。
   - **传统内存访问控制：** 通常涉及多个进程或线程共享相同的内存区域，需要使用锁或其他同步机制来确保数据一致性。
4. **同步和互斥：**
   - **CSP：** 使用通信机制来进行同步，进程之间通过消息传递协调各自的动作。
   - **传统内存访问控制：** 常常需要使用锁或信号量等机制来进行同步和互斥，以防止多个进程同时访问共享的内存区域。

总的来说，CSP 提供了一种不同于传统共享内存的并发模型，强调通过明确的通信来实现进程之间的协同工作，以减少并发问题的出现。这种方式更容易推理和调试，并且有助于构建可靠的并发系统。

### Go

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	type Button struct {
		Clicked *sync.Cond
	}
	button := Button{Clicked: sync.NewCond(&sync.Mutex{})}
	subscribe := func(c *sync.Cond, fn func()) {
		var goroutineRunning sync.WaitGroup
		goroutineRunning.Add(1)
		go func() {
			defer c.L.Unlock()
			goroutineRunning.Done()
			c.L.Lock()
			c.Wait()
			fn()
		}()
		goroutineRunning.Wait()
	}
	var clickRegistered sync.WaitGroup
	clickRegistered.Add(3)
	subscribe(button.Clicked, func() {
		fmt.Println("Maximizing window.")
		clickRegistered.Done()
	})
	subscribe(button.Clicked, func() {
		fmt.Println("Displaying annoying dialog box!")
		clickRegistered.Done()
	})
	subscribe(button.Clicked, func() {
		fmt.Println("Mouse clicked.")
		clickRegistered.Done()
	})
	button.Clicked.Broadcast()
	clickRegistered.Wait()
}
```

这段代码演示了使用条件变量（`sync.Cond`）实现订阅和发布模式的示例。

首先，代码定义了一个名为`Button`的结构体，其中包含一个指向条件变量的指针`Clicked`。通过`sync.NewCond`函数，我们创建了一个与互斥锁关联的条件变量。

接下来，代码定义了一个`subscribe`函数，用于订阅事件。该函数接收一个条件变量`c`和一个回调函数`fn`作为参数。在函数内部，它创建了一个`sync.WaitGroup`类型的变量`goroutineRunning`，用于等待goroutine的启动。然后，它启动一个新的goroutine，在其中等待条件变量的信号。一旦接收到信号，它会执行回调函数`fn`。在等待和执行过程中，它使用互斥锁来保护共享资源。最后，通过`goroutineRunning.Wait()`确保goroutine已经启动。这样做是为了避免出现竞争条件，确保回调函数在订阅完成之后才会执行。

在`main`函数中，我们创建了一个`Button`实例`button`，并调用`subscribe`函数三次，每次传递不同的回调函数。这些回调函数分别打印不同的消息。

随后，我们创建了一个`sync.WaitGroup`类型的变量`clickRegistered`，用于等待所有订阅的回调函数执行完成。通过`clickRegistered.Add(3)`将等待计数设置为3，因为我们有三个订阅的回调函数。

最后，我们调用`button.Clicked.Broadcast()`发送广播信号，通知所有订阅者事件已发生。这将触发所有等待中的`subscribe`函数中的条件变量的信号，并执行对应的回调函数。

通过使用条件变量和互斥锁，代码实现了一个简单的订阅和发布模式。当事件发生时，订阅者收到信号并执行相应的回调函数。这种模式可以在并发环境中实现解耦和事件驱动的编程

### Java

```
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Main {
    public static void main(String[] args) {
        Button button = new Button();

        Thread thread1 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Maximizing window.");
            });
        });

        Thread thread2 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Displaying annoying dialog box!");
            });
        });

        Thread thread3 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Mouse clicked.");
            });
        });

        thread1.start();
        thread2.start();
        thread3.start();

        // 模拟按钮点击事件
        button.click();

        try {
            thread1.join();
            thread2.join();
            thread3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Button {
    private Lock lock;
    private Condition condition;

    public Button() {
        lock = new ReentrantLock();
        condition = lock.newCondition();
    }

    public void click() {
        lock.lock();
        try {
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void subscribe(Runnable callback) {
        Thread thread = new Thread(() -> {
            lock.lock();
            try {
                condition.await();
                callback.run();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        });
        thread.start();
    }
}
```

在上述代码中，使用了`Condition`、`Lock`和`ReentrantLock`来实现线程间的同步和通信。

1. `Lock`接口是Java提供的用于多线程同步的机制之一。`ReentrantLock`是`Lock`接口的一个具体实现类，它提供了独占锁的功能。在示例代码中，我们创建了一个`ReentrantLock`实例，用于保护共享资源的访问。
2. `Condition`接口是与锁关联的条件，可以用于实现线程间的等待和通知机制。在示例代码中，我们使用`lock.newCondition()`创建了一个`Condition`实例，用于实现订阅者线程的等待和主线程的通知。
3. `lock.lock()`和`lock.unlock()`用于获取和释放锁。通过使用`lock.lock()`获取锁，可以确保只有一个线程可以执行被保护的代码块，其他线程将被阻塞。一旦线程完成了对共享资源的操作，使用`lock.unlock()`释放锁，以便其他线程可以获取锁并执行。
4. `condition.await()`用于使当前线程进入等待状态，直到其他线程通过调用`condition.signal()`或`condition.signalAll()`发出信号。在示例代码中，订阅者线程在收到信号前会调用`condition.await()`进入等待状态。
5. `condition.signalAll()`用于唤醒所有等待在该条件上的线程。在示例代码中，主线程调用`button.click()`后会调用`condition.signalAll()`，以通知所有等待的订阅者线程。

通过使用`Condition`、`Lock`和`ReentrantLock`，我们可以实现更精细的线程同步和通信。它们提供了更灵活的控制机制，使得线程之间的交互更加可控和高效。

可以使用`Object`类中的`wait()`和`notifyAll()`方法来实现线程间的等待和通知机制，用于替代`Condition`接口和`Lock`机制。

在使用`wait()`和`notifyAll()`时，需要注意以下几点：

1. `wait()`方法用于使当前线程进入等待状态，直到其他线程调用相同对象的`notify()`或`notifyAll()`方法。在等待期间，当前线程会释放对象的锁。
2. `notifyAll()`方法用于唤醒所有等待在相同对象上的线程。它会通知所有等待的线程继续执行，但只有在获取到对象的锁之后才能真正执行。
3. 在使用`wait()`和`notifyAll()`时，必须在同步代码块或同步方法中调用，以确保对对象的锁的正确使用。
4. 通常，你需要使用一个共享的对象作为通信的锁，类似于示例代码中的`button`对象。

下面是修改后的示例代码，使用`wait()`和`notifyAll()`实现线程间的等待和通知：

```
class Button {
    private final Object lock = new Object();
  
    public void click() {
        synchronized (lock) {
            lock.notifyAll();
        }
    }
  
    public void subscribe(Runnable task) {
        synchronized (lock) {
            try {
                lock.wait();
                task.run();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Button button = new Button();
        Thread subscriber1 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Maximizing window.");
            });
        });
        Thread subscriber2 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Displaying annoying dialog box!");
            });
        });
        Thread subscriber3 = new Thread(() -> {
            button.subscribe(() -> {
                System.out.println("Mouse clicked.");
            });
        });
        subscriber1.start();
        subscriber2.start();
        subscriber3.start();
      
        // 主线程等待订阅者线程完成
        try {
            subscriber1.join();
            subscriber2.join();
            subscriber3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
      
        button.click();
    }
}
```

在这个示例中，`Button`类使用了一个共享的锁对象`lock`。在订阅者线程中，调用`lock.wait()`使线程进入等待状态，在主线程中调用`lock.notifyAll()`唤醒所有等待的线程。主线程使用`join()`方法等待订阅者线程完成后再调用`button.click()`。

使用`wait()`和`notifyAll()`可以实现基本的线程间等待和通知机制，但相比`Condition`和`Lock`，它们的使用更加基础和低级，需要手动处理锁的获取和释放，并且可能存在更多的风险，如死锁和竞态条件。因此，在实际开发中，建议使用`Condition`和`Lock`机制，因为它们提供了更灵活、更可靠的线程同步和通信方式。相比于使用`wait()`和`notifyAll()`，`Condition`和`Lock`具有以下优势：

1. 精确的通知机制：`Condition`接口提供了更细粒度的通知机制，可以选择性地通知等待线程。你可以创建多个`Condition`实例来控制不同的等待条件，并使用`signal()`或`signalAll()`方法通知特定的等待线程。
2. 更灵活的锁控制：`Lock`接口提供了更灵活的锁控制机制。它支持可重入锁（ReentrantLock）和读写锁（ReentrantReadWriteLock），以及各种锁的高级功能，如公平性、超时等待和中断响应。
3. 更安全的并发控制：`Condition`和`Lock`提供了更安全的并发控制机制，避免了可能导致死锁、竞态条件和线程饥饿等问题。它们通过显示地获取和释放锁来确保线程的正确同步和协调。
4. 可扩展性和性能优化：`Condition`和`Lock`机制提供了更高级的线程同步功能，可以更好地满足复杂的并发需求。它们支持更多的高级操作，如条件等待、多个等待队列、可中断的等待等，并提供了更好的性能优化选项。

综上所述，尽管可以使用`wait()`和`notifyAll()`来实现简单的线程同步和通信，但在更复杂的并发场景下，使用`Condition`和`Lock`会更加可靠和灵活。它们提供了更多的功能和性能优化选项，可以更好地管理线程的状态、等待和唤醒，确保线程间的正确同步和协作。