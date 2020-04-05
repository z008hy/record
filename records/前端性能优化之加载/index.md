

## 资源合理化

资源合理化即根据合理打包拆分资源，最大化的利用浏览器请求

### 资源合并
+ 合并图片，制作雪碧图，此处只适用于小图片。超过 30k 的图片则不适用。
+ 在资源 < 30k 的情况下合并小资源，减少 http 请求。

### 资源分离
+ 当资源 > 80k 的情况下最好将资源拆分。
> 由于 浏览器在 http1 下对资源的并发请求是有限制的。例如 Chrome 只支持并发 6 个请求。而此时如果有一个“鹤立鸡群”的资源，则会导致下面6个并发请求必须等待该资源加载完毕再请求~

## 使用 webpack splitChunks 做资源的合理拆分
```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async', 
      // 超过 30k 则进行资源合并
      minSize: 30000,
      // 超过 100k 则进行代码拆分
      maxSize: 100000,
      minChunks: 1,
      // 最大异步请求数
      maxAsyncRequests: 5,
      // 最大初始化请求数
      maxInitialRequests: 3,
      // 打包分割符
      automaticNameDelimiter: '~',
      name: true,
    }
  }
};
```
---
## 缓存
由于所有缓存的控制权都在客户端，使用缓存之前一定要考虑什么数据需要缓存什么数据不能缓存，例如 index.html 不做强缓存。

### 强缓存
存储在 Response Headers。浏览器识别是否进行缓存 Response Headers key 中的相关内容 决定是否读取缓存。命中强缓存的 Status Code 为 400。

#### Expires
例如：
```
Expires:Mar, 08 Apr 2020 10:47:02 GMT
```
该时间代表缓存失效，浏览器根据当前时间与其对比缓存是否可用。由于浏览器的时间可能会不准确，因此该策略很不可靠 😂，不推荐使用。

#### Cache-Control
Cahce-Control 是相对与Expires更加可靠的强缓存方案，并且它的优先级高于Expires目前强缓存采取的主流方案。

```
Cache-Control: max-age=0, s-maxage=137
```
他有常见的几类配置形态：

+ max-age=60 表示首次请求60秒内可以使用缓存
+ private 告诉 代理服务器（CDN）这个是个人信息，你不用存了，如果不标 CDN 默认会存储
+ s-maxage 表示在 代理服务器（CDN）缓存的时间（秒）
+ no-store 不进行任何缓存（包括协商缓存）
+ no-cache 表示不使用强缓存，去服务端获取 资源 然后看要不要协商缓存

### 协商缓存
协商缓存就是指是否缓存我要问问服务器，服务器大佬说了算 😎。命中协商缓存的 Status Code 为 304。

#### Last-Modify/If-Modify-Since
首次请求资源时在 Response Header 中携带一个 
```
Last-Modify:Mar, 08 Apr 2020 10:47:02 GMT
``` 
当下次请求该资源时浏览器在 Request Header 中增加一个
```
If-Modify-Since:Mar, 08 Apr 2020 10:47:02 GMT
```
服务端根据 If-Modify-Since 记录的时间做出判断是否允许读取缓存

#### Etag/If-None-Match
由于 Last-Modify 是针对资源的过期时间做的一种缓存，无法精确定位到资源是否发生过修改。而且 Last-Modify 的最小单位是秒。秒级一下的变化是无法控制的。后起之秀 Etag 出现了 🤗。

收起请求资源时，服务端会根据资源的内容生成一个唯一表示，然后把该标识放在 Response Header 中返回，就像下面这样。
```
Etag: W/"377da-SAl2Ha+YBP9IdylbxZ5fxqePZAQ"
```
再次请求时，浏览器会在 Request Header 中自动加入该标识。
```
if-none-match: W/"377da-SAl2Ha+YBP9IdylbxZ5fxqePZAQ"
```
服务器兄弟会根据该标识来判断是否允许读取缓存。

### Service Worker 缓存

Service Worker 又是另一种缓存控制手段。但是 Service Worker 存在浏览器兼容性问题和需要在前端代码中设置缓存策略。具体相关 Service Worker 的内容可以查看下面文章：

[从 Service Worker 到 Workbox](https://github.com/z008hy/record/blob/master/records/%E4%BB%8E%20Service%20Worker%20%E5%88%B0%20workbox/index.md)

### 总结

+ 一般请求下我们使用 Cache-Control + Etag 的方式进行浏览器的缓存配置足以，没必要使用 Expires 和 Last-Modify。前者存在可控性强方便使用的特点。
+ 强缓存命中 Status Code 为 200 ，协商缓存为 304。
+ 使用 Cache-Control:no-store 时 请注意。 

### 相关文章

+ [HTTP 缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)
+ [Cache-Control for Civilians](https://csswizardry.com/2019/03/cache-control-for-civilians/)
---

## 懒加载
对于用户而言，当前能在页面中看到的信息只有当前展示的。因此没有必要将所有的资源都在用户访问时加载，只需要加载用户此时需要的资源。这种按需加载的方式可以叫做懒加载。

### 图片的懒加载
图片资源的懒加载目前有三种方式，如下：

#### 监听滚动事件
不设置 img 的 src 属性，而是使用其他属性记录 src 例如 data-src，在页面加载完毕后注册 滚动 事件 当图片出现或者即将在可视区域时设置图片src。

#### Intersection Observer API
创建一个 IntersectionObserver 然后监测元素, 在监测回调中替换 src,注意兼容性

[MDN Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API)

[IntersectionObserver API 使用教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2016/11/intersectionobserver_api.html)

[IntersectionObserver’s Coming into View  |  Web  |  Google Developers](https://developers.google.com/web/updates/2016/04/intersectionobserver)

#### 原生支持的 img lazyload
功能很好，兼容性还不足

[New in Chrome 77  |  Web  |  Google Developers](https://developers.google.com/web/updates/2019/09/nic77)

#### 相关资料

[Lazy Loading Images](https://css-tricks.com/snippets/javascript/lazy-loading-images/)

[Lazy Loading Resource](https://developers.google.com/web/fundamentals/performance/lazy-loading-guidance/images-and-video)


### JS/CSS 的懒加载
JS 和 CSS 的懒加载在需要配合 webpack 来做。从内容上分，我们经常会遇到 vue 或 react 与路由配合的懒加载，当然我们也可以做更小力度的懒加载。

### 使用动态语句

```js
let module = await import("./AModule.js");
module.default();
module.andAnotherThing();
```

---

### SSR

SSR 是为了提升首屏的渲染效率。SSR 解决了首屏页面渲染的白屏状态。但是对于大多数的应用你可能不在乎首屏渲染速率。