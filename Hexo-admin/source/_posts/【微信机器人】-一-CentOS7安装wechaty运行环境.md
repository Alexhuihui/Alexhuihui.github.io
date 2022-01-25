---
title: 【微信机器人】(一)CentOS7安装wechaty运行环境
date: 2022-01-24 19:08:45
categories: 微信机器人
tags: [node,微信机器人,gcc]
---

### 安装Node14.x

1. 运行Node.js安装程序脚本

   ```bash
   $ sudo yum -y install curl
   
   $ curl -sL https://rpm.nodesource.com/setup_14.x | sudo bash -
   ```

   <!-- more --> 

2. 在CentOS7上安装Node.js 14版本

   ```bash
   $ sudo yum install -y nodejs
   ```

3. 查看版本

   ```
   $ node -v
   $ npm -v
   ```

### 创建和初始化你的机器人项目（npm install）

因为 PadLocal 使用的某些依赖（特别是 better-sqlite3）是 node native module。所以 npm install 会通过 gyp 编译项目，这个过程中不同平台上可能出现错误。

1. centos安装 Build Tools

   ```bash
   $ sudo yum groupinstall "Development Tools"
   ```

2. [centos升级gcc版本](https://www.cnblogs.com/jixiaohua/p/11732225.html)

3. 安装依赖

   ```bash
   mkdir node-wechat && cd node-wechat
   npm init -y
   npm install ts-node typescript -g --registry=https://r.npm.taobao.org
   tsc --init --target ES6
   ```

4. 编写demo

   ```javascript
   // bot.ts
   
   import {PuppetPadlocal} from "wechaty-puppet-padlocal";
   import {Contact, Message, ScanStatus, Wechaty} from "wechaty";
   
   const token: string = ""            // padlocal token
   const puppet = new PuppetPadlocal({ token })
   
   const bot = new Wechaty({
       name: "TestBot",
       puppet,
   })
   
   bot
   .on("scan", (qrcode: string, status: ScanStatus) => {
       if (status === ScanStatus.Waiting && qrcode) {
           const qrcodeImageUrl = ["https://api.qrserver.com/v1/create-qr-code/?data=", encodeURIComponent(qrcode)].join("");
           console.log(`onScan: ${ScanStatus[status]}(${status}) - ${qrcodeImageUrl}`);
       } else {
           console.log(`onScan: ${ScanStatus[status]}(${status})`);
       }
   })
   
   .on("login", (user: Contact) => {
       console.log(`${user} login`);
   })
   
   .on("logout", (user: Contact) => {
       console.log(`${user} logout`);
   })
   
   .on("message", async (message: Message) => {
       console.log(`on message: ${message.toString()}`);
   })
   
   .start()
   
   console.log("TestBot", "started");
   ```

5. 运行demo

   ```bash
   ts-node bot.ts
   ```

   

