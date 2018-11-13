## 一、当前技术栈
Webpack+React+antd+dva


## 二、遇到的问题
### 1、 Load chunk fail
每次发版后，如果老的js、css等被清除，用户浏览器如果使用老版本时，切换路由时会存在 "Load chunk fail" 报错，导致用户不得不强制手动刷新


### 2、无增量更新（待实现）
https://tech.meituan.com/qrcodepayment_static_optimize.html
（欢迎写出实现方案，完成之二）

## 三、针对问题1的方案
### 1、目标：
1）用户切换路由时将无明显感觉就完成切换刷新
2）页面无报错
3）不涉及服务端进行版本号管理，前端可控


### 2、增加全局Load chunk fail的 报错拦截
1）这个报错来自 webpack的 onScriptComplete方法，而 webpack 没有对外提供抛错处理
2）dva/dynamic  没有对应的 catch 方法
3）window.onerror无法捕获 该promise错误
4）而[unhandledrejection](https://developer.mozilla.org/en-US/docs/Web/Events/unhandledrejection)可以捕获该错误


### 3、执行打包时，自动生成版本号文件
1）所谓版本就是本次发布的唯一值，可以是手写的版本值、内容的变更的hash值、时间戳均可
2）Dice在前端代码不变更时，进行打包，默认不会重新打包生成新的版本号，所以当前采用时间戳


### 4、定时轮询获取最新版本


### 5、代码如下
1）用于生成版本
在npm run build方法中执行 node version.config.js

```javascript
/**
 * version.config.js  应用版本简易生成
 * 1、所谓版本就是时间戳，因为Dice在前端不变更时进行打包，不会重新打包
 * 2、此方案并不能解决强制打包时或前端项目的非业务代码发生变更导致的Dice打包时，前端版本的变更情况
 */
const fs = require('fs');


const data = { version: Date.parse(new Date()) };


fs.writeFile('./public/version.json', JSON.stringify(data), (err) => {
  if (err) {
    console.log('version.json generated fail', err);
  } else {
    console.log('version.json generated ok.');
  }
});

```

2）nginx配置变更
因为文件会被默认的设置缓存，因此需要更改设置不被缓存

```
    location ~* version\.(json)$ {
        expires -1;
        add_header 'Cache-Control' 'no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0';
    }
```

3）报错拦截与轮询

```javascript
/**
 * version.js  版本校验，对外提供 checkVersion 方法
 * 应用版本检查
 * 局限：
 * 1、非增量更新
 * 2、浏览器需要刷新页面
 */
import agent from 'agent';
import { message } from 'antd';


function getCurrentVersion() {
  return agent.get('/version.json')
    .then(response => response.body);
}


window.addEventListener('unhandledrejection', (e) => { // 兜底方案，当用户路由变更时机恰巧在5min时
  const msg = e.reason.message;
  if (msg && msg.indexOf('Loading chunk') > -1) {
    if (navigator.onLine) {
      window.location.reload();
    } else {
      message.error('亲，检查一下网络连接', 3);
    }
  }
  return true;
});


export function checkVersion() {
  let currentVersion = null;
  let timeId = null;
  getCurrentVersion().then((response) => { // 初次获取版本号
    currentVersion = response.version;
  });
  timeId = setInterval(() => {
    getCurrentVersion().then((response) => {
      if (currentVersion === response.version) return;
      window.needReload = true; // 将会正在path-listener监听到路由变化时，被触发
      clearInterval(timeId);
      timeId = null;
    });
  }, 300000); // 5min请求一下版本号
}
```

4）监听路由变更则强制刷新
在dva的全局性的model的subscriptions中监听，或者其他感觉合适的位置

```javascript
let currentPathname = null;
...  
subscriptions: {
    setup({ dispatch, history }) {
      history.listen((location) => {
        if (window.needReload && currentPathname && currentPathname !== location.pathname) { // 当应用版本、pathname发生变化，那么跳转时就进行刷新
          window.location.reload();
        } else {
          currentPathname = location.pathname;
        }
      });
    },
  },
...
```
