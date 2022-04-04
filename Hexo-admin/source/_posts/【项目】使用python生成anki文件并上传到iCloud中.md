---
title: 【项目】使用python生成anki文件并上传到iCloud中
date: 2022-03-11 16:45:55
categories: 个人项目
tags: [genanki, anki, pyicloud]
top: true
---

这周开始看 `java-design-partterns`的时候经常在例子中看到很多不认识的单词，就想着把每次遇到的不认识的单词记录下来发送到手机里，可以随时复习。经过调研，发现python都有现成的库可以使用，使用`genanki`生成anki文件，再使用`pyicloud`登录自己的icloud账号，上传文件到icloud drive中，最后你就能通过anki 备忘录之类的软件从icloud drive中导入你的文件了。

<!-- more --> 

### 使用pyicloud上传文件

从配置文件读取你的apple账号密码，在初始化的方法中登录。对外暴露一个上传文件的接口，默认使用anki文件夹存储。

```python
from pyicloud import PyiCloudService


class Cloud(object):
    def __init__(self):
        username = ''
        password = ''
        for line in open('account.txt', 'r', encoding='utf-8'):
            lines = line.split(' ')
            username = lines[0]
            password = lines[1]
        self.api = PyiCloudService(username, password)
        
    def upload_file(self, file_path):
        with open(file_path, 'rb') as file_in:
            self.api.drive['anki'].upload(file_in)
```



### 使用genanki生成anki文件

比较简单，先定义模板model，然后再把单词的中英文填入模板生成一个个的note，最后再使用deck写入文件（也可以写入数据库）。

### 效果图

上传的效果

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/ea1031963c18781be9686d2a3efe0e4.jpg)

导入之后的效果

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/62ce72db22fb8cfd06561bafe9f056f.jpg)

### 源码地址

https://github.com/Alexhuihui/anki-icloud.git

