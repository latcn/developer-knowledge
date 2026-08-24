# 🧠 JavaScript / TypeScript / Node.js 完全学习指南（后端工程师视角）

> 本指南基于“后端工程师”的知识背景，对比 Java 生态，深入剖析 JS/TS/Node.js 的核心原理、最佳实践，并提供完整可运行的示例代码。全部内容遵循“零环境依赖，开箱即用”的原则。

---

## 目录
1. [JavaScript 语言核心](#1-javascript-语言核心)  
   - 类型系统、原型链、作用域与闭包、this 绑定、事件循环  
2. [ES6+ 核心特性](#2-es6-核心特性)  
   - let/const、解构、箭头函数、Promise、async/await、class、模块、Proxy、Map/Set  
3. [TypeScript 类型系统](#3-typescript-类型系统)  
   - 类型注解、接口/类型别名、泛型、判别联合、品牌类型、工具类型  
4. [Node.js 核心](#4-nodejs-核心)  
   - 事件循环与 libuv、模块系统、流与背压、Worker Threads、子进程、错误处理、安全实践  
5. [工程化与最佳实践](#5-工程化与最佳实践)  
   - Monorepo、TurboRepo、Maven/Gradle 对比、依赖管理、代码规范  
6. [完整可运行示例](#6-完整可运行示例)  
   - 集所有概念于一体的 Node.js ESM 服务  
7. [总结与进阶路径](#7-总结与进阶路径)

---

## 1. JavaScript 语言核心

### 1.1 类型系统：动态弱类型
**第一性原理**：变量无类型，**值有类型**。原始类型（`number`、`string`、`boolean`、`null`、`undefined`、`symbol`、`bigint`）按值传递；引用类型（`object`、`array`、`function`）按引用传递。

```javascript
// 原始类型
let a = 10;
let b = a; b = 20;
console.log(a); // 10（互不影响）

// 引用类型
let obj1 = { name: 'DSH' };
let obj2 = obj1;
obj2.name = 'Node';
console.log(obj1.name); // 'Node'（相互影响）
```

### 1.2 原型链与继承
**第一性原理**：对象通过 `[[Prototype]]` 链接形成原型链。ES6 `class` 是语法糖，底层仍是原型继承。

```javascript
// ES6 class（推荐写法）
class Animal {
    constructor(name) { this.name = name; }
    speak() { console.log(`${this.name} makes a sound.`); }
}
class Dog extends Animal {
    constructor(name, breed) {
        super(name);
        this.breed = breed;
    }
    speak() {
        super.speak();
        console.log(`${this.name} barks!`);
    }
}
const d = new Dog('旺财', '金毛');
d.speak(); // 旺财 makes a sound. \n 旺财 barks!
```

**多继承**：JS 不支持直接多继承，但可通过 **Mixin（混入）** 组合行为。

```javascript
const FlyMixin = (Base) => class extends Base { fly() { console.log('Flying!'); } };
const SwimMixin = (Base) => class extends Base { swim() { console.log('Swimming!'); } };
class Duck extends SwimMixin(FlyMixin(Animal)) { }
const duck = new Duck('唐老鸭');
duck.fly(); // Flying!
duck.swim(); // Swimming!
```

### 1.3 作用域与闭包
**第一性原理**：**闭包**是函数及其词法环境的组合，内部函数持有外部变量的引用。

```javascript
function createCounter() {
    let count = 0;
    return function() { return ++count; };
}
const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

### 1.4 `this` 绑定
**第一性原理**：`this` 在函数调用时确定，规则：`new` > `call/apply/bind` > 对象方法 > 默认（严格模式 `undefined`）。箭头函数继承外层 `this`。

```javascript
function show() { console.log(this); }
const obj = { fn: show };
obj.fn(); // obj
show.call({name:'explicit'}); // {name:'explicit'}
const arrow = () => console.log(this);
arrow(); // 继承定义时的 this（严格模式下为 undefined）
```

### 1.5 事件循环（Event Loop）与任务队列
**第一性原理**：JS 单线程，通过事件循环调度**宏任务**（`setTimeout`、I/O）和**微任务**（`Promise.then`、`queueMicrotask`）。每个宏任务执行完，清空全部微任务。

```javascript
console.log('1. 同步');
setTimeout(() => console.log('4. 宏任务'), 0);
Promise.resolve().then(() => console.log('3. 微任务'));
queueMicrotask(() => console.log('2. 微任务'));
// 输出顺序：1 → 2 → 3 → 4
```

**宏任务分类**：`setTimeout`、`setInterval`、`setImmediate`（Node）、I/O、UI 渲染。  
**微任务分类**：`Promise.then`、`queueMicrotask`、`MutationObserver`（浏览器）、`process.nextTick`（Node，优先级最高）。

---

## 2. ES6+ 核心特性

### 2.1 `let` / `const` 与块级作用域
- `let` / `const` 块级作用域，存在**暂时性死区**（TDZ），不挂载到全局对象。
- `const` 必须初始化且不可重新赋值（引用类型内容可变）。

```javascript
// var 函数级作用域，变量提升
console.log(a); // undefined
var a = 1;

// let TDZ
// console.log(b); // ReferenceError
let b = 2;

// const
const c = 3;
// c = 4; // TypeError
const obj = { name: 'DSH' };
obj.name = 'Node'; // ✅ 允许
```

### 2.2 解构赋值
```javascript
const config = { host: 'localhost', port: 3000 };
const { host, port } = config;
const [first, second] = [1, 2, 3];
```

### 2.3 箭头函数
- 没有自己的 `this`、`arguments`，继承外层作用域。
- 适用于回调函数，但不适用于构造函数或需要动态 `this` 的场景。

### 2.4 Promise 与异步编程
```javascript
const fetchData = () => new Promise((resolve, reject) => {
    setTimeout(() => resolve({ data: 'DSH' }), 1000);
});
fetchData()
    .then(data => console.log(data))
    .catch(err => console.error(err))
    .finally(() => console.log('done'));
// 并发
Promise.all([fetchData(), fetchData()]).then(([r1, r2]) => console.log(r1, r2));
```

### 2.5 `async` / `await`
```javascript
async function getData() {
    try {
        const result = await fetchData();
        return result.data;
    } catch (err) {
        throw err;
    }
}
```

### 2.6 类（`class`）
```javascript
class Animal {
    #privateField = 'secret'; // 私有字段
    constructor(name) { this.name = name; }
    speak() { console.log(this.name); }
    getSecret() { return this.#privateField; }
}
class Dog extends Animal { /* ... */ }
```

### 2.7 模块（ES Module）
```javascript
// math.js
export const PI = 3.14;
export default function add(a, b) { return a + b; }

// app.js
import add, { PI } from './math.js';
```

### 2.8 迭代器与生成器
**生成器** `function*` 可暂停、恢复，自动生成迭代器。

```javascript
function* range(start, end) {
    for (let i = start; i <= end; i++) yield i;
}
const iter = range(1, 3);
console.log([...iter]); // [1, 2, 3]
```

### 2.9 `Proxy` 与 `Reflect`（元编程）
```javascript
const target = { name: 'DSH' };
const proxy = new Proxy(target, {
    get(obj, prop, receiver) {
        console.log(`读取 ${prop}`);
        return Reflect.get(obj, prop, receiver);
    },
    set(obj, prop, value, receiver) {
        console.log(`设置 ${prop}=${value}`);
        return Reflect.set(obj, prop, value, receiver);
    }
});
proxy.name; // 读取 name
```

### 2.10 `Map` / `Set` / `WeakMap` / `WeakSet`
- `Map`：任意类型键，键值对。
- `Set`：值不重复的集合。
- `WeakMap`：键为对象，弱引用，不阻止 GC。

---

## 3. TypeScript 类型系统

### 3.1 类型注解与推断
```typescript
let name: string = 'DSH';
let age = 30; // 推断为 number
let union: string | number = 'hello';
let unknownVal: unknown = 'hello';
if (typeof unknownVal === 'string') unknownVal.toUpperCase();
```

### 3.2 接口（Interface）与类型别名（Type Alias）
```typescript
interface User { id: number; name: string; email?: string; }
type ID = string | number;
type UserWithoutEmail = Omit<User, 'email'>;
interface Admin extends User { permissions: string[]; }
```

### 3.3 泛型（Generics）
```typescript
function identity<T>(value: T): T { return value; }
interface ApiResponse<T> { data: T; message: string; }
class Queue<T> { private items: T[] = []; push(item: T) { this.items.push(item); } }
```

### 3.4 判别联合（Discriminated Union）
```typescript
type State = 
    | { status: 'loading'; startTime: number }
    | { status: 'success'; data: string }
    | { status: 'error'; error: Error };
function handle(state: State) {
    switch (state.status) {
        case 'loading': console.log(state.startTime); break;
        case 'success': console.log(state.data); break;
        case 'error': console.log(state.error.message); break;
        default: const _exhaustive: never = state; break;
    }
}
```

### 3.5 品牌类型（Branded Types）
```typescript
type Brand<T, B> = T & { __brand: B };
type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
function createUserId(id: string): UserId { return id as UserId; }
```

### 3.6 工具类型
- `Partial<T>`、`Required<T>`、`Readonly<T>`
- `Pick<T, K>`、`Omit<T, K>`
- `Exclude<T, U>`、`ReturnType<T>`、`Parameters<T>`

---

## 4. Node.js 核心

### 4.1 事件循环与 libuv
Node.js 底层是 libuv（C++），事件循环阶段：timers → pending callbacks → idle/prepare → poll → check → close callbacks。`setImmediate` 在 check 阶段执行，`process.nextTick` 优先级高于微任务。

```javascript
setTimeout(() => console.log('timers'), 0);
setImmediate(() => console.log('check'));
process.nextTick(() => console.log('nextTick')); // 最先
```

### 4.2 模块系统（CommonJS vs ESM）
- **CommonJS**：`require`，动态加载，运行时解析。
- **ES Module**：`import`，静态加载，编译时解析，支持 Tree Shaking。

```javascript
// CommonJS
const fs = require('fs');

// ESM (package.json "type": "module" 或 .mjs)
import { readFile } from 'node:fs/promises';
```

### 4.3 流（Stream）与背压（Backpressure）
```javascript
import { createReadStream, createWriteStream } from 'node:fs';
const readStream = createReadStream('large.txt');
const writeStream = createWriteStream('out.txt');
readStream.pipe(writeStream); // 自动处理背压

// 手动处理背压
async function send(res, generator) {
    for await (const chunk of generator) {
        if (!res.write(chunk)) await new Promise(resolve => res.once('drain', resolve));
    }
    res.end();
}
```

### 4.4 Worker Threads（CPU 密集任务）
```javascript
import { Worker, isMainThread, parentPort, workerData } from 'node:worker_threads';
if (isMainThread) {
    const worker = new Worker(new URL(import.meta.url), {
        workerData: { n: 42 },
        env: {},          // 空环境变量
        execArgv: [],     // 不继承启动参数
        resourceLimits: { maxOldGenerationSizeMb: 64 }
    });
    worker.on('message', console.log);
    worker.on('error', console.error);
} else {
    const { n } = workerData;
    const result = fib(n);
    parentPort.postMessage(result);
}
```

### 4.5 子进程（Child Process）与进程树终结
```javascript
import { spawn } from 'node:child_process';
const child = spawn('sleep', ['60'], { detached: true, stdio: 'ignore' });
child.unref();
// 杀死进程组（POSIX）
process.kill(-child.pid, 'SIGTERM');
// Windows: spawn('taskkill', ['/PID', pid, '/T', '/F'])
```

### 4.6 错误处理与优雅退出
```javascript
process.on('uncaughtException', (err) => {
    console.error(err);
    gracefulShutdown('uncaughtException');
});
process.on('unhandledRejection', (reason) => {
    console.error(reason);
    gracefulShutdown('unhandledRejection');
});

function gracefulShutdown(signal) {
    server.close(() => {
        cleanup();
        process.exit(0);
    });
    setTimeout(() => process.exit(1), 5000);
}
```

### 4.7 安全实践
- **路径穿越防护**：`resolve(root, userInput)` 后检查是否以 `root + sep` 开头。
- **环境变量隔离**：Worker / 子进程设置 `env: {}`。
- **请求体大小限制**：检查 `content-length`，超限 `req.destroy()`。
- **依赖审计**：`npm audit`，使用 `package-lock.json`。

---

## 5. 工程化与最佳实践

### 5.1 Monorepo vs Polyrepo
| 对比维度 | Monorepo | Polyrepo |
|---------|----------|----------|
| 代码复用 | 直接引用，即时生效 | 需发布升级，周期长 |
| 变更原子性 | 一次 PR 跨项目修改 | 多仓库分步修改，易断裂 |
| 构建效率（无工具） | 慢（全量构建） | 快（独立构建） |
| 权限管理 | 困难（需精细控制） | 天然隔离 |
| 依赖版本 | 统一提升，干净 | 各自锁定，易冲突 |

**推荐**：多个强关联项目（如 BFF + 前端）用 Monorepo；完全解耦的微服务用 Polyrepo。

### 5.2 Monorepo 构建工具：TurboRepo vs Maven/Gradle
- **相似**：都处理依赖图、支持并行构建、有缓存机制。
- **差异**：
  - Maven/Gradle **内置编译能力**（`javac`），拥有完整生命周期。
  - TurboRepo **仅调度脚本**，不关心编译细节，依赖外部工具链（`tsc`、`webpack`）。
  - Maven/Gradle 仅限 JVM 生态；TurboRepo 跨语言（任何有命令行脚本的项目）。

### 5.3 编码规范（ESLint / Prettier）
- 默认使用 `const`，需重新赋值用 `let`，禁用 `var`。
- 使用 `===` 而非 `==`。
- 所有异步操作使用 `async/await` 并 `try/catch`。
- 函数参数使用对象解构实现命名参数。

---

## 6. 完整可运行示例

以下是一个 **集所有核心概念于一体的 Node.js ESM 服务**，保存为 `full-training.js`，执行 `node full-training.js`，浏览器访问 `http://localhost:3000` 即可交互。

```javascript
// ================================================================
// 文件名: full-training.js
// 描述: JS/TS/Node.js 全栈核心原理完整训练 (ESM)
// 运行: node full-training.js
// 测试: 浏览器打开 http://localhost:3000
// ================================================================

import { createServer } from 'node:http';
import { readFile, access, mkdir, writeFile } from 'node:fs/promises';
import { createReadStream } from 'node:fs';
import { resolve, join, sep } from 'node:path';
import { fileURLToPath } from 'node:url';
import { Worker } from 'node:worker_threads';
import { spawn } from 'node:child_process';
import { randomUUID } from 'node:crypto';
import { EventEmitter } from 'node:events';

const __dirname = fileURLToPath(new URL('.', import.meta.url));
const PORT = 3000;

// ==================== 1. 路由系统 (Exact + Prefix) ====================
const routes = { exact: new Map(), prefix: new Map() };

function addRoute(method, path, handler) {
    const key = `${method}:${path}`;
    if (routes.exact.has(key)) throw new Error(`路由重复: ${key}`);
    routes.exact.set(key, handler);
}

function addPrefix(method, prefix, handler) {
    const key = `${method}:${prefix}`;
    if (routes.prefix.has(key)) throw new Error(`前缀路由重复: ${key}`);
    routes.prefix.set(key, handler);
}

// ==================== 2. HTML 首页 ====================
addRoute('GET', '/', async (req, res) => {
    const html = `
    <!DOCTYPE html>
    <html>
    <head><title>JS/TS/Node.js 全栈训练</title>
    <style>
        body{font-family:system-ui;padding:20px;max-width:900px;margin:0 auto;background:#0f1117;color:#e8edf5;}
        .card{background:#1a1d27;padding:20px;border-radius:12px;margin:12px 0;border:1px solid #2a2f3c;}
        button{padding:10px 20px;background:#7cb8ff;color:#0f1117;border:none;border-radius:6px;cursor:pointer;font-weight:600;}
        button:hover{background:#5a9ae0;}
        pre{background:#0b0d12;color:#d4dcec;padding:12px;border-radius:6px;overflow-x:auto;font-family:monospace;font-size:0.85rem;}
        .tag{color:#f07178;}.attr{color:#ffcb6b;}.str{color:#c3e88d;}.kw{color:#c792ea;}
    </style>
    </head>
    <body>
        <h1>🧠 JS / TS / Node.js 全栈核心原理</h1>
        <div class="card"><h3>📡 SSE 流 (Generator + Stream)</h3><button id="sseBtn">连接 SSE</button><div id="sseLog"></div></div>
        <div class="card"><h3>🧵 Worker Threads (CPU 密集)</h3><button id="workerBtn">计算斐波那契 (n=42)</button><span id="workerResult"></span></div>
        <div class="card"><h3>🔪 子进程树终结</h3><button id="spawnBtn">启动并杀死子进程</button><span id="spawnResult"></span></div>
        <div class="card"><h3>📦 事件驱动 (EventEmitter)</h3><button id="eventBtn">触发事件</button><pre id="eventLog"></pre></div>
        <div class="card"><h3>🔒 路径穿越防护</h3><a href="/static/../../package.json" target="_blank">尝试路径穿越 (应返回 403)</a></div>
        <script>
            document.getElementById('sseBtn').onclick = () => {
                const evt = new EventSource('/api/stream');
                const log = document.getElementById('sseLog');
                evt.onmessage = e => { log.innerHTML += '<div>📩 ' + e.data + '</div>'; };
                evt.onerror = () => evt.close();
            };
            document.getElementById('workerBtn').onclick = async () => {
                const res = document.getElementById('workerResult');
                res.textContent = '⏳ 计算中...';
                const resp = await fetch('/api/worker?n=42');
                const data = await resp.json();
                res.textContent = '✅ 结果: ' + data.result + ' (耗时 ' + data.duration + 'ms, 线程: ' + data.threadId + ')';
            };
            document.getElementById('spawnBtn').onclick = async () => {
                const res = document.getElementById('spawnResult');
                res.textContent = '⏳ 启动子进程...';
                await fetch('/api/spawn', { method: 'POST' });
                res.textContent = '✅ 子进程已启动，5秒后自动终结';
                setTimeout(async () => {
                    await fetch('/api/kill', { method: 'POST' });
                    res.textContent = '🔪 子进程树已终结';
                }, 5000);
            };
            document.getElementById('eventBtn').onclick = async () => {
                const resp = await fetch('/api/event');
                const data = await resp.json();
                document.getElementById('eventLog').textContent = JSON.stringify(data, null, 2);
            };
        </script>
    </body>
    </html>
    `;
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(html);
});

// ==================== 3. SSE 流 (Generator 转 Stream) ====================
addRoute('GET', '/api/stream', async (req, res) => {
    res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive'
    });
    res.write(': connected\n\n');

    // 生成器: 产生 SSE 帧
    async function* generateEvents() {
        for (let i = 1; i <= 10; i++) {
            yield `data: ${JSON.stringify({ id: i, time: new Date().toISOString(), msg: 'DSH 下行帧' })}\n\n`;
            await new Promise(r => setTimeout(r, 800));
        }
    }

    // 背压控制: 逐帧写入
    for await (const chunk of generateEvents()) {
        const canContinue = res.write(chunk);
        if (!canContinue) {
            await new Promise(resolve => res.once('drain', resolve));
        }
    }
    res.end();

    // 客户端断连检测
    res.on('close', () => console.log('[SSE] 客户端断开'));
});

// ==================== 4. Worker Threads (CPU 密集 + 隔离) ====================
addRoute('GET', '/api/worker', async (req, res) => {
    const url = new URL(req.url, 'http://x');
    const n = parseInt(url.searchParams.get('n')) || 42;

    // 【铁律】Worker 环境完全隔离
    const worker = new Worker(
        `
        const { parentPort, threadId } = require('worker_threads');
        function fib(n) { return n <= 1 ? n : fib(n-1) + fib(n-2); }
        const start = performance.now();
        const result = fib(${n});
        parentPort.postMessage({ result, duration: performance.now() - start, threadId });
        `,
        {
            env: {},           // 空环境变量
            execArgv: [],      // 不继承启动参数
            resourceLimits: { maxOldGenerationSizeMb: 64 }
        }
    );

    const result = await new Promise((resolve, reject) => {
        const timer = setTimeout(() => {
            worker.terminate();
            reject(new Error('Worker 超时'));
        }, 10000);
        worker.on('message', (msg) => { clearTimeout(timer); resolve(msg); });
        worker.on('error', reject);
        worker.on('exit', (code) => {
            if (code !== 0) reject(new Error(`Worker 退出码: ${code}`));
        });
    });

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(result));
});

// ==================== 5. 子进程树终结 ====================
let childProcess = null;

addRoute('POST', '/api/spawn', async (req, res) => {
    if (childProcess) { res.writeHead(400); return res.end('已有子进程'); }

    const isWin = process.platform === 'win32';
    const cmd = isWin ? 'ping' : 'sleep';
    const args = isWin ? ['127.0.0.1', '-t'] : ['30'];

    childProcess = spawn(cmd, args, {
        detached: true,
        stdio: ['ignore', 'pipe', 'pipe'],
        env: {}  // 空环境
    });

    childProcess.stdout.on('data', (d) => console.log(`[子进程] ${d.toString().trim()}`));
    childProcess.unref();

    res.writeHead(200);
    res.end(JSON.stringify({ pid: childProcess.pid }));
});

addRoute('POST', '/api/kill', async (req, res) => {
    if (!childProcess) return res.end('无子进程');

    const pid = childProcess.pid;
    if (process.platform === 'win32') {
        spawn('taskkill', ['/PID', pid.toString(), '/T', '/F'], { stdio: 'ignore' });
    } else {
        try { process.kill(-pid, 'SIGTERM'); } catch (e) {}
        setTimeout(() => { try { process.kill(-pid, 'SIGKILL'); } catch (e) {} }, 1000);
    }
    childProcess = null;
    res.end(JSON.stringify({ status: 'killed' }));
});

// ==================== 6. EventEmitter (观察者模式) ====================
class AppEvents extends EventEmitter {}
const events = new AppEvents();

// 事件监听
events.on('user:login', (user) => {
    console.log(`[事件] 用户 ${user} 登录`);
});
events.on('user:action', (action) => {
    console.log(`[事件] 用户执行: ${action}`);
});

addRoute('POST', '/api/event', async (req, res) => {
    const id = randomUUID();
    events.emit('user:login', `user-${id.slice(0, 6)}`);
    events.emit('user:action', `操作-${Date.now()}`);

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        status: 'ok',
        events: ['user:login', 'user:action'],
        timestamp: new Date().toISOString()
    }));
});

// ==================== 7. 静态资源 + 路径穿越防护 ====================
addPrefix('GET', '/static', async (req, res) => {
    const url = new URL(req.url, 'http://x');
    let pathname = url.pathname.replace(/^\/static/, '') || '/index.html';

    const staticRoot = resolve(__dirname, 'public');
    const target = resolve(join(staticRoot, pathname));

    // 【铁律】路径穿越防护
    if (!target.startsWith(staticRoot + sep) && target !== staticRoot) {
        res.writeHead(403);
        return res.end('Forbidden: 路径穿越检测');
    }

    try {
        await access(target);
        const stream = createReadStream(target);
        res.writeHead(200);
        stream.pipe(res);
    } catch (e) {
        if (e.code === 'ENOENT') {
            res.writeHead(404);
            res.end('Not Found');
        } else {
            res.writeHead(500);
            res.end('Internal Error');
        }
    }
});

// ==================== 8. HTTP 服务器 (请求级错误隔离) ====================
const server = createServer(async (req, res) => {
    try {
        const url = new URL(req.url || '/', 'http://x');
        const method = req.method;
        const pathname = url.pathname;

        // 路由匹配: Exact → Prefix
        const exactKey = `${method}:${pathname}`;
        let handler = routes.exact.get(exactKey);

        if (!handler) {
            for (const [key, h] of routes.prefix) {
                const [m, prefix] = key.split(':');
                if (m === method && pathname.startsWith(prefix)) {
                    handler = h;
                    break;
                }
            }
        }

        if (!handler) {
            res.writeHead(404);
            return res.end('Not Found');
        }

        await handler(req, res);
    } catch (err) {
        // 【铁律】请求异常绝不崩溃
        console.error('[请求异常]', err.message);
        if (!res.headersSent) {
            res.writeHead(500, { 'Content-Type': 'text/plain' });
            res.end('Internal Server Error: ' + err.message);
        } else {
            res.destroy();
        }
    }
});

// ==================== 9. 进程生命周期 (优雅退出) ====================
let shuttingDown = false;

function gracefulShutdown(signal) {
    if (shuttingDown) return;
    shuttingDown = true;
    console.log(`\n[优雅退出] 收到 ${signal}`);

    server.close(() => {
        console.log('[优雅退出] HTTP 服务器已关闭');
        if (childProcess) {
            console.log('[优雅退出] 清理子进程...');
            try {
                if (process.platform === 'win32') {
                    spawn('taskkill', ['/PID', childProcess.pid.toString(), '/T', '/F']);
                } else {
                    process.kill(-childProcess.pid, 'SIGKILL');
                }
            } catch (e) {}
        }
        process.exit(0);
    });

    setTimeout(() => { console.error('[优雅退出] 超时，强制退出'); process.exit(1); }, 5000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('uncaughtException', (err) => {
    console.error('[严重] 未捕获异常:', err);
    gracefulShutdown('uncaughtException');
});
process.on('unhandledRejection', (reason) => {
    console.error('[严重] 未处理 Rejection:', reason);
    gracefulShutdown('unhandledRejection');
});

// ==================== 10. 启动 ====================
await mkdir(resolve(__dirname, 'public'), { recursive: true });
await writeFile(resolve(__dirname, 'public/index.html'), `
    <!DOCTYPE html>
    <html><head><title>静态 SPA Fallback</title></head>
    <body style="font-family:system-ui;padding:20px;background:#0f1117;color:#e8edf5;">
        <h1>📁 静态资源目录</h1>
        <p>SPA Fallback: 请求不存在的文件会回到这里。</p>
    </body>
    </html>
`);

server.listen(PORT, '0.0.0.0', () => {
    console.log(`\n✅ 全栈训练服务启动: http://localhost:${PORT}`);
    console.log(`📌 PID: ${process.pid}`);
    console.log(`📌 按 Ctrl+C 触发优雅退出\n`);
});
```

---

## 7. 总结与进阶路径

### 7.1 核心要点回顾
- **JS 核心**：原型链、闭包、事件循环（宏任务/微任务）是理解异步的基础。
- **ES6+**：`const`/`let`、`Promise`、`async/await`、`Proxy` 是现代开发标配。
- **TypeScript**：类型系统增强了可维护性，品牌类型、判别联合等模式提升安全性。
- **Node.js**：流、Worker Threads、子进程、错误处理、安全防护是服务端开发的关键。

### 7.2 对比 Java 的认知迁移
| Java 概念 | JS/Node.js 对应 |
|-----------|-----------------|
| JVM | V8 引擎 + libuv |
| 多线程 | Worker Threads (真正的 OS 线程) |
| Maven/Gradle | npm/pnpm + TurboRepo (编排) |
| Spring Boot | NestJS / Express |
| 同步 I/O | 异步 I/O (非阻塞) |
| 异常处理 | try/catch + uncaughtException 兜底 |

### 7.3 下一步学习建议
- **Web 框架**：Express（轻量）→ NestJS（企业级，类似 Spring）
- **数据库 ORM**：Prisma / TypeORM
- **测试**：Jest / Vitest
- **构建工具**：Vite / Webpack
- **容器化**：Docker + 多阶段构建
- **监控**：OpenTelemetry + Prometheus

---

> 本指南所有示例均可在 Node.js 18+ 环境中直接运行，无需任何外部依赖。祝你学习愉快，从“后端工程师”成长为“全栈架构师”！