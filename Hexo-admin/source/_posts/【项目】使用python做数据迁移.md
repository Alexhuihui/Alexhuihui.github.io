---
title: 【项目】使用python做数据迁移
date: 2022-10-28 19:45:55
categories: 个人项目
tags: python

---

工作中经常会遇到重构老项目，所以需要迁移之前的数据库到新的数据库中，为此我使用`python`来做数据迁移脚本。

<!-- more --> 

### 初始版本

第一个版本的数据迁移脚本，主要是使用单条`sql`的插入，效率偏低。

先获取原始数据读取到内存中

```python
def get_original():
    connect = Connect.get_original_connect()
    select_sql = '''select * from table'''
    cursor = connect.cursor()
    cursor.execute(select_sql)
    # 获取所有记录列表
    return cursor.fetchall()
```

再在`main`函数中处理原始数据，调用插入函数，插入到新数据库中

```python
def main():
    original_list = get_original()
    a = 0
    for original in original_list:
        print(original)
        insert_into_target(original)
        print('插入数据成功' + str(original[0]))
        a = a + 1
    print(a)
```

插入数据

```python
def insert_into_usr(mbr):
    connect = Connect.get_user_connect()
    cursor = connect.cursor()
    id = mbr[0]
    name = mbr[1]
    value_lo = ((id, name))
    insert_sql = '''insert into usr_user(id, name) 
             values (%s, %s)'''
    cursor.execute(insert_sql, value_lo)
    connect.commit()
    cursor.close()
```

第一个版本的脚本易于理解，代码简单，但是由于是单条插入，速度很慢。所以接下来我们改进一下脚本。

### 改进版

改进思路就是由单条插入改成批量插入`executemany`

第一步先获取原始数据读取到内存中

```python
def get_original():
    connect = Connect.get_original_connect()
    select_sql = '''select * from table'''
    cursor = connect.cursor()
    cursor.execute(select_sql)
    # 获取所有记录列表
    return cursor.fetchall()
```

定义全局数组变量，然后在`main`函数中调用插入函数拼接好数据往这个全局变量中插入，然后到一定次数后调用数据库批量插入数据

```python
target_data = []

def main():
    original_list = get_original()
    a = 0
    for i in range(len(original_list)):
        original = original_list[i]
        insert_into_target(original)
        
        if i == (len(original_list) - 1):
            do_insert_target()
            a = a + 1
        if i != 0 and i % 1000 == 0:
            do_insert_target()
            a = a + 1
    print(a)
    
def do_insert_target():
    connect = Connect.get_user_connect()
    cursor = connect.cursor()
    insert_sql = '''insert into target(id, name) 
             values (%s, %s)'''
    cursor.executemany(insert_sql, target_data)
    connect.commit()
    cursor.close()
    print('插入数据成功' + str(len(target_data)))
    user_data.clear()


def insert_into_target(mbr):
    id = mbr[0]
    name = mbr[1]

    value_lo = (id,
                name)
    target_data.append(value_lo)
```

### 总结

遇到数据量很大的情况，使用批量插入能极大程度提高数据迁移的速度，但要做好数据防丢失的措施。