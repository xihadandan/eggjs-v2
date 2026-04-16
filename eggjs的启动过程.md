# eggjs的启动过程

## egg-bin debug --port=7001 都发生了什么
- egg-bin 是个node可执行的命令
- 在egg-bin包中 package.json
```javascript
...
"bin": {
    "egg-bin": "bin/egg-bin.js",
    "mocha": "bin/mocha.js",
    "ets": "bin/ets.js",
    "c8": "bin/c8.js"
  },
...
```

- egg-bin\bin\egg-bin.js
```javascript
#!/usr/bin/env node

'use strict';

const Command = require('..');

// 实例化并启动，Command类指的是egg-bin\index.js中的 EggBin类
new Command().start(); 
// start方法在 common-bin\lib\command.js中的CommonBin 类
```

- egg-bin\index.js
```javascript
...
const Command = require('./lib/command');
class EggBin extends Command {
  constructor(rawArgv) {
    super(rawArgv);
    this.usage = 'Usage: egg-bin [command] [options]';

    // load directory 调用的是CommonBin中的load
    this.load(path.join(__dirname, 'lib/cmd'));
  }
}
...

```

- egg-bin\lib\command.js
```javascript
const BaseCommand = require('common-bin');
class Command extends BaseCommand {
  constructor(rawArgv) {
    super(rawArgv);
    ...
  }
}

```
- egg-bin\node_modules\common-bin\lib\command.js
```javascript
class CommonBin {
  constructor(rawArgv) {
    /**
     * original argument
     * @type {Array}
     */
    this.rawArgv = rawArgv || process.argv.slice(2);
    // process.argv
    // (4) ['D:\\nodejs\\node.exe', 'E:\\2git-work\\platform_v7\\wellpt-front-end\\wellapp-web\\node_modules\\egg-bin\\bin\\egg-bin.js', 'debug', '--port=7001']
    // this.rawArgv
    // (2) ['debug', '--port=7001']
    ...
  }
  load(fullPath) {
    ...
  }
  start() {
    // new Command().start();会进入这个方法
    co(function* () {
      ...
      yield this[DISPATCH]();
    }.bind(this)).catch(this.errorHandler.bind(this));
  }
  * [DISPATCH]() {
    // 会执行两次
    ...
    const parsed = yield this[PARSE](this.rawArgv);
    const commandName = parsed._[0]; // 得到指定的命令名称是debug
    ...
    if (this[COMMANDS].has(commandName)) {
        // 第一次进入，由EggBin实例调用start()后进入;
        ...
        // 会实例化 egg-bin\lib\cmd\debug.js中的 DebugCommand类
        const command = this.getSubCommandInstance(Command, rawArgv); // 得到bug命令实例
        yield command[DISPATCH](); 
        return;
    }
    ...
    // 第二次进入，由bug命令实例发起
    yield this.helper.callFn(this.run, [ context ], this); // 调用debug中的run
  }
}


```
### 文字描述
- 继承关系
```javascript
/*

继承关系
EggBin --> Command --> CommonBin(npm包common-bin)

DebugCommand --> DevCommand --> Command --> CommonBin(common-bin)

*/
```
- EggBin实例化
- EggBin构造方法中加载所有egg-bin\lib\cmd目录下所有命令，并添加到COMMANDS中
- EggBin实例调用自身的start方法，在 CommonBin类中有声明start方法
- CommonBin类中的start方法调用DISPATCH 方法
- 第一次进入DISPATCH 方法（EggBin实例的）实例化debu命令（DebugCommand)
- DevCommand类中 设置子进程要运行的模块 this.serverBin = path.join(__dirname, '../start-cluster')
- 第一次进入DISPATCH 方法（EggBin实例的）调用debug命令实例的DISPATCH
- 第二次进入DISPATCH 方法（debug命令实例的），并调用debug命令实例的 run方法
- run方法中格式化参数（调用DevCommand类中的 formatArgs）
- formatArgs中得到framework 调用 argv.framework = utils.getFrameworkPath （ utils.getFrameworkPath在egg-utils\lib\framework.js中）
- run方法中 fork子进程  const child = cp.fork(this.serverBin, eggArgs, options);


### debug命令会使用 child_process 模块来创建（fork）子进程 start-cluster.js
- 自动建立 IPC 通道‌：父进程与子进程可通过 send() 和 message 事件双向通信
- 一个进程至少包含一个线程，线程是进程的一部分，不能独立于进程存在 
- egg-bin\lib\cmd\debug.js

```javascript
const Command = require('./dev');
const cp = require('child_process');
// 继承dev
class DebugCommand extends Command {
  constructor(rawArgv) {
    super(rawArgv);
  }
  * run(context) {
    ...
    const eggArgs = yield this.formatArgs(context); // 调用dev中的formatArgs方法
    // this.serverBin在 \egg-bin\lib\cmd\dev.js 中
    // this.serverBin = path.join(__dirname, '../start-cluster');
    const child = cp.fork(this.serverBin, eggArgs, options);
    ...
  }
  child.on('message', msg => {
      if (msg && msg.action === 'debug' && msg.from === 'app') {
        ...
      }
  })

}

```

- egg-bin\lib\cmd\dev.js
```javascript
class DevCommand extends Command {
    constructor(rawArgv) {
        super(rawArgv);
        ...
        this.serverBin = path.join(__dirname, '../start-cluster');
        ...
    }
    * formatArgs(context) {
        const { cwd, argv } = context;
        // argv.workers 是 1
        argv.baseDir = argv.baseDir || cwd;
        ...
        argv.port = argv.port || argv.p;
        argv.framework = utils.getFrameworkPath({
            framework: argv.framework,
            baseDir: argv.baseDir,
        }); 
        // argv.framework的值是 "E:\\2git-work\\platform_v7\\wellpt-front-end\\wellapp-web\\node_modules\\wellapp-framework"
        return [ JSON.stringify(argv) ];
    }
}
```

- egg-utils\lib\framework.js
```javascript
...
function getFrameworkPath({ framework, baseDir }) {
    const pkgPath = path.join(baseDir, 'package.json');
    assert(fs.existsSync(pkgPath), `${pkgPath} should exist`);

    const moduleDir = path.join(baseDir, 'node_modules');
    const pkg = utility.readJSONSync(pkgPath);
    ...
    if (pkg.egg && pkg.egg.framework) { // pkg.egg.framework 是 "wellapp-framework"
        return assertAndReturn(pkg.egg.framework, moduleDir);
    }

}
...

```

### 启动集群
- egg-bin\lib\start-cluster
```javascript
#!/usr/bin/env node

'use strict';

const debug = require('debug')('egg-bin:start-cluster');
const options = JSON.parse(process.argv[2]);
debug('start cluster options: %j', options);
require(options.framework).startCluster(options); // 调用 egg-cluster\index.js中 startCluster 方法

```
### 集群启动流程
- egg-cluster\index.js
```javascript
const Master = require('./lib/master');
/**
 * cluster start flow:
 *
 * [startCluster] -> master -> agent_worker -> new [Agent]       -> agentWorkerLoader
 *                         `-> app_worker   -> new [Application] -> appWorkerLoader
 *
 */
exports.startCluster = function(options, callback) {
  new Master(options).ready(callback); // 实例化
};


```


