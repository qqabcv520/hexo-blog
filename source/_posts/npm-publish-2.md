---
title: 使用node.js开发命令行工具（二）命令行输入输出UI库
date: 2019-04-08 09:09:44
tags: ["Node.js"]
---
相信很多前端都听说过或者使用过@angular/cli, vue-cli, create-react-app或其他类似的命令行工具。他们能够在命令行后面跟各种复杂的参数已经交互性的命令行选项，那你知道这些功能是怎么实现的吗？
<!--more-->
## commander.js

node.js命令行开发工具开发库，使node.js开发CLI工具变得简单，允许快捷的定义形如`<command> [options]`的命令。

基础用法：
```js
const program = require('commander');

 program
  .version(require('../package.json').version, '-v, --version') // 定义版本信息
  .usage('<command> [options]'); // 定义命令用法
 
program
  .command('rm <dir>') // 定义一个rm命令
  .description('删除文件或文件夹') // 给rm命令添加描述信息，获取命令帮助信息的时候会显示
  .option('-r, --recursive', 'Remove recursively') // rm允许添加-r或者--recursive命令进行递归
  .action(function (dir, cmd) { // 对应命令的处理函数
    console.log('remove ' + dir + (cmd.recursive ? ' recursively' : ''))
  });
 
 program.parse(process.argv); // commander的入口欧，传入命令行参数执行解析
```
github仓库：https://github.com/tj/commander.js


## inquirer.js

node.js 交互式命令行界面开发库，允许方便的定义使用上下左右进行列表选择等交互式命令。


基础用法：
```js
const inquirer = require('inquirer');

inquirer.prompt(
{
    type: 'input', // 问题类型，包括input，number，confirm，list，rawlist，password
    name: 'name', 
    message: '请输入项目名称', // 问题
    default: 'unnamed' // 默认值
    validate: (input: string) => {
        if (input.length > 255) { // 输入验证：name长度不允许超过255
            return '项目名称超过限制';
        }
        return true;
    }        
},
{
    type: 'list',
    name: 'type',
    message: '请选择',
    choices: ['item1', 'item2', 'item3', 'item4'], // 可选选项
    default: 'project'
}).then(answers => {
    console.log(answers.name);
    console.log(answers.type);
});
```
{% asset_img inquirer.js.jpg inquirer.js示例 %}

github仓库：https://github.com/SBoudrias/Inquirer.js


## ora

优雅的命令行Loading动画。

```js
const ora = require('ora');

const spinner = ora('Loading unicorns').start();

setTimeout(() => {
	spinner.color = 'yellow';
	spinner.text = 'Loading rainbows';
}, 1000);

setTimeout(() => {
	spinner.stop();
}, 2000);
```
<p align="center">
	<br>
	{% asset_img ora.svg ora.js示例 %}
	<br>
</p>

github仓库：https://github.com/sindresorhus/ora
