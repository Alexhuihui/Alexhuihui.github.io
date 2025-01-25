---
title: 【GitHub】使用Maven发布Java包
date: 2022-11-03 19:25:18
categories: Maven
tags: Maven
---

本文介绍如何通过`GitHub`的`actions`在代码推送到仓库时，将`Java`包发布到`maven`仓库的过程。

<!-- more --> 

### 修改`pom`文件

首先你要在项目的`pom`文件中添加要发布的仓库地址，我们准备发到`github package`上

```xml
<!-- 发布maven私服 -->
 <distributionManagement>
   <repository>
     <id>github</id>
     <name>GitHub OWNER Apache Maven Packages</name>
     <url>https://maven.pkg.github.com/Alexhuihui/alex-common</url>
   </repository>
 </distributionManagement>
```

### 在项目根目录下新建`.github/workflows/maven-publish.yml`

```yaml
# This workflow will build a package using Maven and then publish it to GitHub packages when a release is created
# For more information see: https://github.com/actions/setup-java/blob/main/docs/advanced-usage.md#apache-maven-with-a-settings-path

name: Maven Package

on: [push]

jobs:
  publish:
    runs-on: ubuntu-latest 
    permissions: 
      contents: read
      packages: write 
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-java@v2
        with:
          java-version: '8'
          distribution: 'temurin'
      - name: Publish package
        run: mvn --batch-mode deploy
        env:
          GITHUB_TOKEN: ${{ secrets.REPOSITORY_TOKEN }}
```

### 申请自己的`accessToken`

进入自己`github`，然后在设置下面的开发者设置中申请一个新的`personal access token`

### 配置加密机密

进入自己的仓库，在设置下面的机密中的工作流中新建一个机密。

### 参考

[发布包到 Maven 中心仓库和 GitHub Packages](https://docs.github.com/cn/actions/publishing-packages/publishing-java-packages-with-maven)