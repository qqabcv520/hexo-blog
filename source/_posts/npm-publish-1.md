---
title: 使用node.js开发命令行工具（一）创建与发布
date: 2019-03-31 14:27:10
tags: ["node.js"]
---
在前端工程化的大环境下，合理运用node和npm script，可以大大提高我们的开发效率，那么怎么才能使自己开发的nodejs代码通过npm安装，就可以直接使用命令行进行调用呢？
<!--more-->

## 准备工作
确保自己有node环境，并且node已经配置到环境变量。没安装的可以到[官网下载](https://nodejs.org)或者[国内镜像下载](http://nodejs.cn/)。

## 第一行node代码
* 新建一个js文件，比如: `hello.js`。
* 在js文件中键入`console.log('hello world!')`，保存。
* 在打开控制台，切换到`hello.js`所在的目录，执行`node ./hello.js`命令中。

执行完上面的最后一步命令后，就可以看到控制台执行了我们的js文件，输出`hello world`了。

## 强大的nodejs
我们已经写出了我们的第一个node应用，想要做出更复杂更强大的应用，也只是时间问题了。不过，nodejs除了语法和浏览器端的一样，api和浏览器端是完全不一样的，nodejs没有浏览器端的bom和dom对象，取而代之的是操作系统api和一些工具包，详细的api文档可以查看[英文文档](https://nodejs.org/en/docs/)或者[中文文档](http://nodejs.cn/api/)。

## 命令行工具
当我们开发好node程序之后，能不能不经过node，直接像vue/cli这种cli工具那样，输入`hello`执行我们刚刚的程序呢呢？当然可以。
* 创建一个新文件夹作为项目目录。
* 使用控制台进入项目目录，执行`npm init`初始化一个npm项目。
* 将之前的hello.js放入项目目录。
* 打开`package.json`文件，在底部添加一项配置`"bin"`，`bin`对象里的key就是命令名称，value就是要执行的js文件。
    ```json
    {
      "name": "test",
      "version": "1.0.0",
      "description": "",
      "main": "index.js",
      "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
      },
      "author": "",
      "license": "MIT",
      "bin": {
        "hello": "hello.js"
      }
    }
    ```
* 打开`hello.js`文件，在文件顶部添加一句`#!/usr/bin/env node`并保存，用以指定使用node.js运行当前js文件。

这个时候，我们的程序已经完成了，只需要发布，使用的时候安装就可以了。那么怎么发布呢，有两种方式，一种方式是本地发布`npm link`，这种方式只有本地能进行安装，另一种方式是`npm publish`发布到npm中央仓库，任何人都能够使用npm安装你的应用。

## 测试
当我们怎么样进行本地测试运行呢？我们可以使用`npm link`命令，这时候我们的当前项目就会被发布到本地并全局安装了，我们可以直接使用`hello`命令运行刚刚的js文件了。

除此之外，如果我们想局部安装，我们可以切刀需要局部安装的项目中，使用`npm link <packageNmae>`替代`npm install <packageNmae>`命令，进行局部安装。被局部安装的包，不会添加到全局变量，但是可以使用npm script进行调用。

## 发布
* 在[npm官网](https://www.npmjs.com/)注册一个自己的账号，用于发布和管理自己的npm包。
* 给你的npm包起个名字，`package.json`中的`name`字段就是你的npm包的名字，`name`在官网查询下是不是重复，重复的包名不能提交。
* 给你的npm包定义版本号，`package.json`中的`version`字段就是你的npm包的版本号，`version`应该比之前的版本递增，推荐使用[语义化版本](https://semver.org/lang/zh-CN/)规范。
* 使用`npm publish`命令，根据提示先登录npm账号，然后发布npm包。

按步骤执行到这里，整个npm包就发布完成了，我们可以在其他的npm项目下面用`npm install`命令安装我们发布的包了。
