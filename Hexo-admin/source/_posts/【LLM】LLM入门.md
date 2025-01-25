---
title: 【LLM】LLM入门
date: 2023-04-14 19:54:06
categories: LLM
tags: LLM
---

### `LLM`介绍

语言模型在最近几年内迅速提高，大型语言模型（LLMs）如GPT-3和GPT-4成为中心。这些模型因其能够以惊人的技巧执行各种任务而变得流行。此外，随着这些模型的参数数量（数十亿！）增加，它们不可预测地获得了新的能力。

在本文中，我们将探讨LLMs、它们可以执行的任务、它们的缺点以及各种提示工程策略。

<!-- more --> 

#### 什么是LLMs？

LLMs是神经网络，在大量文本数据上进行了训练。训练过程使模型学习文本中的模式，包括语法、句法和词汇关联。这些学习到的模式被用于生成类似人类文字，使其非常适合自然语言处理（NLP）任务。

#### 哪些LLM可用？

有几个LLM可用，其中GPT-4最受欢迎。其他模型包括 LLaMA、PaLM、BERT 和 T5 等等。每个模型都有其优点和缺点，并且一些是开放性使用而其他则只能通过API使用。

#### LLM 的局限性

尽管表现出色, 但 LLMS 也存在一些限制. 其中一个重要缺陷是无法超越提示中提供的信息进行推理。此外，LLMs 可以根据它们所训练的数据生成有偏见的文本。控制 LLMs 的输出也很具挑战性，因此需要使用提示工程策略来实现所需的输出。。

### 你能执行哪些任务？

我们可以通过为每个案例格式化提示来指导LLMs执行特定的任务。以下是这些任务和提示模板以及每个示例的列表。

#### 文本摘要

LLMs可以生成长文本的摘要，使内容更易于理解和消化。我们可以使用以下提示模板请求对一段文字进行摘要：

```
<Full text>
Summarize the text above: / Explain the text above in <N> sentences:
```

例如，我们可以要求按照以下提示对这段文本进行总结：

```
In a mechanical watch the watch's gears are turned by a spiral spring called a mainspring. In a manual watch energy is stored in the mainspring by turning a knob, the crown on the side of the watch. Then the energy from the mainspring powers the watch movement until it runs down, requiring the spring to be wound again.
A self-winding watch movement has a mechanism which winds the mainspring using the natural motions of the wearer's body. The watch contains an oscillating weight that turns on a pivot. The normal movements of the watch in the user's pocket (for a pocketwatch) or on the user's arm (for a wristwatch) cause the rotor to pivot on its staff, which is attached to a ratcheted winding mechanism. The motion of the watch is thereby translated into circular motion of the weight which, through a series of reverser and reducing gears, eventually winds the mainspring. There are many different designs for modern self-winding mechanisms. Some designs allow winding of the watch to take place while the weight swings in only one direction while other, more advanced, mechanisms have two ratchets and wind the mainspring during both clockwise and anti-clockwise weight motions.
Explain the text above in 2 sentences:
```

这将产生以下输出：

```
A mechanical watch uses a mainspring to power the movement of the watch's gears. In a manual watch, the mainspring is wound by turning a knob, while a self-winding watch uses the natural motion of the wearer's body to wind the mainspring through an oscillating weight and ratcheted winding mechanism.
```

#### 信息提取

LLMs可以从大量文本中识别和提取特定信息。这在新闻和商业情报等领域非常有用。以下模板指导模型提取信息：

```
<Full text>
Mention X in the text above: / List X in the text above:
```

例如，我们可以要求从以下文本中提取地点名称：

```
The earliest credible evidence for a successful design is the watch made by the Swiss watchmaker Abraham-Louis Perrelet, who lived in Le Locle. In late 1776 or early 1777, he invented a self-winding mechanism for pocket watches using an oscillating weight inside the watch that moved up and down. The Geneva Society of Arts, reporting on this watch in 1777, stated that 15 minutes walking was necessary to fully wind the watch.
List the names of places in the text above:
```

这将产生以下输出：

```
Le Locle, Geneva.
```

### 问答

LLMs还可以根据提示提供的信息回答问题。以下模板可用于指导LLM回答问题：

模板

```
基于下面的上下文回答问题。<其他说明>。如果不确定，请回复“不确定”。
上下文：<Context>
Q：<Question>
A：
```

例如，使用以下提示：

提示

```
基于下面的上下文回答问题。用一句话回答。如果不确定，请回复“不确定”。
上下文：自动表也称为自动绕线表或简单地称为自动表，是一种机械表，在佩戴者的自然运动中提供能量来绕制主发条，使手工绕线变得不必要。它与手动表有所区别，因为手动表必须定期手摇其主发条。
Q：自动腕表需要手动绕吗？
A：
```

ChatGPT 回应道：

输出

```
否，如果足够佩戴，则无需手动缠绕自动腕表，因为佩戴者的自然运动会提供能量来维持其正常运转。
```

#### 文本分类

LLMs可以将文本分类为预定义的类别，例如情感分析或垃圾邮件检测。您可以按以下方式格式化文本分类指令：

模板

```
将文本分类为<选择列表>：
文本：<完整文本>
分类：
```

例如，我们可以尝试对一段文字进行情感分析，判断其情绪是积极的、中性的还是消极的：

提示

```
将该段文字归类为积极、中性或消极：
文本：那个投篮选择太糟糕了。
分类：
```

输出

```
消极
```

#### 对话

LLM可以生成类似人类的对话，使它们在聊天机器人和客户服务应用程序中非常有用。我们可以指示LLM按照特定方式跟随对话行动。

模板

```
以下是与<Actor>的对话。 <Actor>应该回答<Further instructions>。
个人：<Conversation>
<Actor>：<Response>
个人：<Conversation>
<Actor>：
```

LLM将填写我们指示其类型演员的响应，并且我们可以通过再次将整个会话及新响应传递给它来继续会话。例如：

提示

```
以下是与律师的对话。 律师应简单明了地回答。
个人：您好，我想知道如何争议停车罚单
律师：
```

输出

```
当然，我能够提供帮助。第一步是检查罚单是否存在任何错误或失误。如果发现任何问题，则可以使用这些作为争议罚款的理由。如果没有错误，则仍然可以通过提供证据支持您的案件（例如照片或证言）来争议罚款。您还可以请求听证会以亲自陈述您的案件。
```

提示

```
以下是与律师的对话。 律师应简单明了地回答。
个人：您好，我想知道如何争议停车罚单
律师：当然，我能够提供帮助。第一步是检查罚单是否存在任何错误或失误。如果发现任何问题，则可以使用这些作为争议罚款的理由。如果没有错误，则仍然可以通过提供证据支持您的案件（例如照片或证言）来争议罚款。您还可以请求听证会以亲自陈述您的案件。
个人：我如何请求听证会？
律师：
```

输出

```
要请求听证会，您需要按照票据背面或网站上的说明进行操作
```

#### 代码生成

LLMs可以生成代码，使开发人员更容易创建复杂的软件程序。我们可以编写一个代码注释来描述我们希望代码执行的操作，LLM将能够尽力完成它。我们希望代码执行的操作越复杂，我们在提示注释中需要提供的信息就越具体和详细。

模板可能因所需生成的编程语言而异，但我建议大致遵循以下模板：

模板

```
<您选择的语言的注释符号（/*或'''或#）>
方法：<方法名称>
语言：<编程语言>
说明：<方法必须执行什么操作>
参数：<方法将接收哪些参数>
上下文：<实现该方法所需进一步信息（URL、数据库等）>
输出：<该方法输出内容>
<您选择的语言的注释符号（/*或'''或#）>
```

例如，如果我们想要获取分页API，则可以编写以下评论：

提示

```
/* Method: fetchFirst5Pages
 * Language: Javascript
 * Description: Fetch the first 5 pages from this paginated API: http://example.com/pages/?n=x
 * Context: This API accepts the following URL parameters:
 *   n: the number of the page to fetch
 * Output: the content of the first 5 pages
 */
```

输出

```
这是JavaScript中fetchFirst5Pages函数实现示例，假设使用了fetch函数：

async function fetchFirst5Pages() {
  const baseUrl = 'http://example.com/pages/?n=';
  const pages = [];

  for (let i = 1; i <= 5; i++) {
    const url = baseUrl + i;
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`Error fetching page ${i}`);
    }
    
    const pageData = await response.json();
    pages.push(pageData);

  }

  return pages;
}

该函数使用循环通过将页码附加到基本URL来获取API的前5个页面。它检查服务器的响应是否正常，如果不正常则抛出错误。然后，它从响应中解析JSON数据并将其添加到页面数组中。最后，它返回页面数组。
```

#### 推理

法学硕士可能会在一些基本任务上遇到困难，比如算术问题。然而，如果我们要求LLM对其回答进行推理，我们可以帮助它得出正确的答案。这是我推荐的模板：

模板

```
问： <问题>
答： 让我们逐步解决这个问题，确保我们有正确的答案。首先，<说明>。
```

在这个例子中，如果没有推理就提出一个简单的算术问题，则无法成功：

提示

```
回答以下问题。 如果不确定答案，请回复“不确定”。
问：以下列表中质数之和是3的倍数：[2, 4, 6, 7, 9, 14]
A: [是或否]
```

输出

不是。 列表[2、4、6、7、9、14]中质数之和不是3的倍数。 因此，答案为“否”。

但是，如果指示LLM对其回答进行推理，则会得到非常不同的结果：

提示

```
问：以下列表中质数之和是否为3的倍数？ [2、4、6、7、9、14]
A: 让我们逐步解决这个问题以确保我们有正确的答案。 首先找到列表中所有质数并将它们加起来。 然后检查所得数字是否可以被3整除。
```

输出

```
好吧，请让我们逐步解决这个问题。

列表中的质数是2和7。 所以我们可以将它们加在一起：

2 + 7 = 9

结果为9。 现在我们需要检查它是否是3的倍数。 我们可以通过检查9除以3的余数是否等于0来做到这一点：

9％3 = 0

由于余数为0，因此我们知道9确实是3的倍数。

因此，答案是“是”，列表[2、4、6、7、9、14]中质数之和是3的倍数。
```

#### 策略

虽然这些模板可以帮助您挤出LLMs的潜力以满足您的需求，但有时它们仍然可能在某些复杂任务上遇到困难。已经开发了一些技术来帮助LLMs得出令人满意的答案。到目前为止，我们只使用了零-shot提示，现在我们将对其进行定义。但是，我们可以使用更复杂的技术。

##### 零-shot提示

在本文中的所有示例中，我们都使用了零-shot提示。零-shot提示意味着向LLM传递一个提示，其中只包含我们想要它回答的问题，并没有任何其他情况下应该如何响应的示例。例如，在早期示例中：

Prompt

```
将文本分类为积极、中性或消极：
Text: That shot selection was awful
Classification:
```

Output

```
Negative
```

这可能不足以让模型准确地回答我们想要它回答的问题。让我们看看还有哪些替代方案。

##### 少量样本提示

与零-shot提示相反，在少量样本提示中，在提问之前给LLM提供一些如何响应的示例。例如，假设我们想像先前示例那样对文本进行情感分类。但是每次都希望以非常特定格式获得答案。我们可以事先给它一些示例，如下所示：

Prompt

```
将文本分类为积极、中性或消极：
Text: Today the weather is fantastic
Classification: Pos
Text: The furniture is small.
Classification: Neu
Text: I don't like your attitude
Classification: Neg
Text: That shot selection was awful
Classification:
```

Output

```
Neg
```

##### 思维链提示

对于更复杂的任务，我们可以将少量示例提示与使模型推理其答案相结合。 我们可以使用之前的相同示例来查看推理和少量提示如何影响结果：

```
Q: The sum of prime numbers in the following list are a multiple of 3: [2, 4, 6, 7, 9, 14]
A: The prime numbers in the list are 2 and 7. Their sum is 9. 9 % 3 = 0. The answer is yes.
Q: The sum of prime numbers in the following list are a multiple of 3: [1, 2, 4, 7, 8, 10]
A: The prime numbers in the list are 1, 2 and 7. Their sum is 10. 10 % 3 = 1. The answer is no.
Q: The sum of prime numbers in the following list are a multiple of 3: [1, 2, 3, 4]
A: The prime numbers in the list are 1, 2 and 3. Their sum is 6. 6 % 3 = 0. The answer is yes.
Q: The sum of prime numbers in the following list are a multiple of 3: [5, 6, 7, 8]
A:
The prime numbers in the list are 5 and 7. Their sum is 12. 12 % 3 = 0. The answer is yes.
```

### 最后的注释

LLM已经彻底改变了自然语言处理领域，但为了最大化其潜力，提示工程至关重要。通过理解LLM可以执行的任务、它们的缺点以及各种提示工程策略，开发人员可以利用LLM的能力来创建创新和有效的解决方案。未来可能会开发更多的策略和技术，因此请继续关注这个领域的进展，以进一步最大化LLM潜力。此外，随着LLMs参数数量不断增加到数十亿级别，我们现在甚至无法想象更多任务将有可能被实现。想到使用这些新工具所能实现什么样子以及它们将为我们服务于哪些用例是非常令人兴奋的事情。