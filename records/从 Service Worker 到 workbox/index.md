## 从 Service Worker 到 workbox

---

## Service Worker

---

### 解释
> Service Worker 是浏览器在后台独立于网页运行的脚本，它打开了通向不需要网页或用户交互的功能的大门。

[Service Worker：简介  |  Web Fundamentals  |  Google Developers](https://developers.google.com/web/fundamentals/primers/service-workers?hl=zh-cn)
___
![](./assets/images/service-worker-frame.png)

---

### 技术特点
+ 独立于主线程 (不可以操作 Dom)
+ 仅仅支持 https （本地调试 localhost 除外）
+ 大量使用了 Promise
+ 缓存需要结合 Cache Storage 使用
+ 使用不当相当危险

---

### 兼容性
![](./assets/images/caniuse-sw.png)
![](./assets/images/caniuse-cache.png)
___
![](./assets/images/caniuse-fetch.png)

[Can I use](https://caniuse.com/)

---

### 一个 Service Worker 的 Demo
___
```js
// index.html
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js')
    .then(function (registration) {
       console.log('Service Worker registration successfully: ');
    })
    .catch(function (err) {
       console.log('Service Worker registration failed: ');
    });
}
```
___
```js
// service-worker.js
var CACHE_NAME = 'SERVICE_WORKER_DEMO';

var preCacheList = [
  '/static/images/pig.png',
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(preCacheList))
  );
});

self.addEventListener('activate', function(event) {
  event.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (key === CACHE_NAME) return caches.delete(key);
      })
    )).then(() => {
      console.log(`[lifecycle][${id}]: activated`);
    })
  );
});

self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        if (response) return response;
        return fetch(event.request).then(function(response) {
          if (response && response.status === 200 && response.type === 'basic') {
            var responseClone = response.clone();
            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseClone);
              });
            return response;
          }
          return response;           
        });
      })
  );
});
```

---

### 生命周期
___
![](./assets/images/service-worker-flow.png)
___
![](./assets/images/service-worker-lifecycle.png)
___
#### register
```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js', {scope: '/'})
    .then(function (registration) {
       console.log('Service Worker registration successfully: ');
    })
    .catch(function (err) {
       console.log('Service Worker registration failed: ');
    });
}
```
___
<p class="content">
register 之前需要判断浏览器是否支持 Service Worker。注册事件的返回值就是一个 Promise。scope 表示 service worker 只能拦截 scope 下发出的请求。例如设置 scope 为 '/test/' 则无法拦截 '/test-a/index.html' 下发出的请求。
</p> 
___
#### install
<p class="content">
install 事件，当 Service Worker 安装完毕后会触发 install 事件。event.waitUntil 会延长 install 事件，它接受一个 promise，当这个 promise 被 resolve 结束安装过程。如果报错则进入 redundant。在这里适合做一些 precache 把一些 lazy-load 的资源在这里可以预先获取并且缓存起来。
</p> 
___
```js
var preCacheList = [
  '/static/images/pig.png',
  '/static/images/snake.png',
  '/static/images/dragon.png',
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(preCacheList))
  );
});
```
___
<p class="content">
caches.open 用来打开一个缓存空间。
caches.allAll 相当于是一个 Promise.all 。在这里它将所有的资源请求并且缓存起来。
</p>
___

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(reg => {
    });
  });
}
```
___
<p class="content">
为了防止 Service Worker 在安装 precache 的过程中占用页面首屏的带宽，我们需要在页面加载完毕后再注册 Service Worker。
</p>
___
#### activate
<p class="content">
activate 是指 Service Worker 完成了安装过程。开始激活，激活后便可以开始工作。关于 activate， 面对之前已经安装过 Service Worker 和没有安装过 Service Worker 是两种情况。
</p>
___
1. 没有安装过
   
![](./assets/images/service-worker-install.gif)
___
2. 安装过
   
![](./assets/images/service-worker-activate.gif)
___
<p class="content">
当目前已经 install 了一个 service-worker，我们更新了 Service Worker 内容时，浏览器会比对出内容的字节变化，并且 install 一个新的 Service Worker。
然后新的 service-worker 等待老的 Service Worker redundant（用户关闭或者刷新页面）。新的 Service Worker 便开始工作。
</p>
___
<p class="content">
下图是调试在浏览器会出现以下情况，新的 Service Worker  "waiting" 老的 Service Worker  交接工作。
</p>
___
![](./assets/images/chrome-devtool-skipWaiting.png)
___
<p class="content">
由于在新的 Service Worker 激活前的 install 阶段 我们已经 “培养” 了 新的 Service Worker 的 precache。在 activate 阶段我们可以 “铲除” 老的 Service Worker 的 precache，这样可以节省浏览器的存储空间。
</p>

---

### self.skipWaiting() 和 self.clients.claim() 
___
#### self.skipWaiting()
<p class="content">
当 Service Worker 更新时，新的 Service Worker 不想等待 老的 Service Worker 完成工作（关闭、刷新），而想获取控制权怎么办？下面代码中 "self.skipWaiting()" 的作用是让新的 Service Worker 跳过等待直接接管工作。
</p>
___
```js
self.addEventListener('install', function() {
  self.skipWaiting();
});
```
___
#### self.clients.claim()
<p class="content">
self.skipWaiting() 解决了 同一 scope 下 新老 Service Worker 快速掌握控制权的问题。但是当原页面没有 Service Worker 或者 是被其他 scope 的 Service Worker 控制。现在要快速获取控制权如何办?
</p>
___
```js
self.addEventListener('activate', function(){
  self.clients.claim();
});
```
---
### Fetch Event
___
<p class="content">
fetch 是 Service Worker 的核心事件。当请求时会触发 fetch 事件。 因此可以拦截到所有的 request。 并且你可以在对这些拦截到的 request 做处理，比如返回你自己构造的 response。
</p>
___

```js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        if (response) return response;
        return fetch(event.request).then(function(response) {
          if (response && response.status === 200 && response.type === 'basic') {
            var responseClone = response.clone();
            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseClone);
              });
            return response;
          }
          return response;           
        });
      })
  );
});
```

---

### 如何移除一个 service-worker
___
没有效果的方法
+ 关闭浏览器
+ 关闭电脑
+ 清除缓存
+ 删除 注册 Service Worker 的代码
___
有效的方法
___
在 Chrome dev-tool 中手动 unregister

![](./assets/images/chrome-devtool-unregister.png)
___
调用 unregister 方法
```js
navigator.serviceWorker.getRegistrations().then(function(registrations) {
  for(let registration of registrations) {
    registration.unregister()    
  }
}).catch(function(err) {
  console.log('Service Worker registration failed: ', err);
});
```
___
注册一个新的 service-worker.v2.js 代替老的 service-worker.v1.js
```js
if ('serviceWorker' in navigator) {
  // navigator.serviceWorker.register('/service-worker.v1.js', {scope: '/'})
  navigator.serviceWorker.register('/service-worker.v2.js', {scope: '/'})
    .then(function (registration) {
       console.log('Service Worker registration successfully: ');
    })
    .catch(function (err) {
       console.log('Service Worker registration failed: ');
    });
}
```
___
更新 service-worker.js 

---

### service-worker 错误使用与危险场景
___
<p class="content">
将首页 index.html 缓存策略设置为 "only cache"。
注册的 service-worker.js 使用 service-worker.v1024.js 的 contenthash 形式，每次更新 Service Worker 业务时 更改 hash。
当需要修改 Service Worker 业务时，更新 service-worker.v1024.js 为 service-worker.v2048.js。
由于 index.html 中的 hash 一直是之前的 "service-worker.v1024.js"，因此 Service Worker 永远不会刷新。
更加致命的时由于这些错误发生在客户端将永远无法修改。
</p>
___
该场景出现的问题：
+ index.html 的缓存策略为 "only cache"。
+ service-worker.js 的命名方式为 contenthash的方式。
___
<p class="content">
一个 spa 应用存在 lazy-load 的业务场景下，他所有 lazy-load 的路由都注册在 index.js。
当用户访问 index.html 后并且一直停留在该页面。此时用户打开了一个新的 tab 页，此时我们恰好更新了 service-worker.js，并且使用 self.skipWaiting() 直接接管了页面所有的 fetch 事件监听。在 activate 中也删除了之前无用的路由 cache。此时用户在老的tab页中点击一个路由会加载 'manage.v1024.js',而这个js 的 cache 已经在 activate 中被移除，在服务器也被移除。用户访问便会出现 404 的情况。
</p>
___
该场景出现的问题
+ 静态资源不是增量的部署

---
### Service Worker 工程化的问题
+ precache 的缓存文件需要手写，但是项目中一般是 webpack 打包出的文件
+ 定义路由过于复杂，没有完善的路由系统
+ 请求策略其实就只有几种，但是需要自己抽象
+ 手写 Service Worker 过于危险
---

## workbox

---

### 解释
workbox 是对service-worker 的封装。让使用 service-worker 更加简单。

[Workbox  |  Google Developers](https://developers.google.com/web/tools/workbox)
___
#### workbox 有什么好处？
___
+ 有 webpack 插件做配合
+ 封装了 precaching
+ 封装了 请求策略
+ 封装了 路由
+ 封装了 缓存过期策略
  
---

#### 自动构建 prechahing 列表
```js
// webpack.js
const WorkboxPlugin = require('workbox-webpack-plugin');

new WorkboxPlugin.InjectManifest({
  swSrc: './src/service-worker.js',
  maximumFileSizeToCacheInBytes: 4 * 1024 * 1024,
  exclude: [/\.(?:png)$/]
})
```
___
workbox-webpack-plugin 会在 swSrc 上下文注入一个数组 self.__WB_MANIFEST, 该数组为所有 webpack 构建出的文件数组。
```js
// service-worker.js
precacheAndRoute(self.__WB_MANIFEST || []);
```

---

#### 路由
___
```js
import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate, NetworkFirst} from 'workbox-strategies';

registerRoute(/(\/|\.html)$/, new NetworkFirst());
registerRoute(/\.(?:js|css)$/, new StaleWhileRevalidate());
```

---

#### 请求策略
___
根据资源的不同性质可以采用不同的请求策略，workbox 帮我们封装了 5 种缓存策略
___

<a data-request="http://localhost:9999/stale-while-revalidate-random">Stale-While-Revalidate<a/>

<img class="strategies" src="./assets/images/workbox-logo.svg" data-placeholder-src="./assets/images/workbox-logo.svg" data-test-src="./assets/images/workbox-stale-while-revalidate.png"/>

___
<a data-request="http://localhost:9999/cache-first-random">Cache First<a/>

<img class="strategies" src="./assets/images/workbox-logo.svg" data-placeholder-src="./assets/images/workbox-logo.svg" data-test-src="./assets/images/workbox-cache-first.png"/>
___
<a data-request="http://localhost:9999/network-first-random">Network First<a/>

<img class="strategies" src="./assets/images/workbox-logo.svg" data-placeholder-src="./assets/images/workbox-logo.svg" data-test-src="./assets/images/workbox-network-first.png"/>
___
<a data-request="http://localhost:9999/network-only-random">Network Only<a/>

<img class="strategies" src="./assets/images/workbox-logo.svg" data-placeholder-src="./assets/images/workbox-logo.svg" data-test-src="./assets/images/workbox-network-only.png"/>
___
Cache Only

<img class="strategies" data-src="./assets/images/workbox-logo.svg" data-placeholder-src="./assets/images/workbox-logo.svg" data-test-src="./assets/images/workbox-cache-only.png"/>

---

#### 缓存过期策略
___
```js
registerRoute(
  matchImg('workbox-cache-first.png'),
  new CacheFirst({
    cacheName: 'img-strategies-test',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 20,
        maxAgeSeconds: 7 * 24 * 60 * 60,
      }),
    ],
  })
);
```
---
### 目前使用 Service Worker 的站点
[twitter](https://twitter.com/home)

[微博移动端](https://m.weibo.cn/)

[codesandbox](https://codesandbox.io/)

[......](https://developers.google.com/web/showcase)

---
![](./assets/images/service-worker-mind.png)
