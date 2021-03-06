## 背景信息
+ 本文对于的 vue 版本 2.6.11

## 关键角色
+ observer
+ watcher
+ dep

### observer
observer 用于处理一个 state，处理前是一个普通的 state，处理后就变成了一个 响应式的 state

### watcher
watcher 是一个监听者，watcher 的作用是监听 state 的变化，然后做相应的处理

### dep
dep 可以视为是一个中介，它把 watcher 和数据的变化关联起来，在变化发生的时候，通知所有watcher工作。 简单来说 dep 管理所有的 watcher。

## 阶段
整个响应式系统的组成分两个阶段：依赖收集、依赖触发。这两个阶段都发生在 Object.defineProperty 的 get 和 set 中。

### 依赖收集
在 vue 初始化的时候会进行依赖收集的阶段，当然通过 Vue.set 等操作也会进行依赖收集。

收集阶段是 watcher 读取 state 时触发了 state 的 get 方法。在get 方法内部 通过 dep.depend() 来把 watcher 收集到 dep 上。这里可以看到 watcher dep 以及 state 都参与到了收集的场景中。dep 像一个中介把 watcher 和 state 关联了起来。具体的方法调用如下的顺序所示：

1. new Watcher()
2. watcher.get()
3. Object.defineProperty 的 get()
4. dep.depend()
5. wather.addDep()
6. dep.addSub()

#### watcher.get()
```js
  get () {
    // 挂载当前 watcher 到 Dep.target
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      // 调用监听 state 的 get
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // 如果需要深度监听则递归所有子属性
      if (this.deep) {
        traverse(value)
      }
      // 收集依赖完毕，执行完毕恢复 挂载当前 watcher 到 Dep.target
      popTarget()
      // 收集依赖完毕，将 newDeps 存储到 deps 清空 newDeps
      this.cleanupDeps()
    }
    return value
  }
```
watcher.get 内部会执行 pushTarget 将当前 watcher (this) 放在 Dep 的静态属性 target 上。这样做的原因是 Object.defineProperty 的 get() 无法传递参数，只能将 watcher 挂载在一个地方，然后 Object.defineProperty 的 get() 中再获取。


#### dep.depend()
```js
depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
```
dep.depend() 所做的就是拿到 Dep 的静态属性 target，也就是拿到 watcher。然后调用 watcher.addDep 方法，并且把 当前的 dep（this）传递过去,将控制权交给 watcher。

#### wather.addDep()
```js
addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```
将 自己的 dep 列表中放入 dep，然后在把控制权交给 dep，执行dep.addSub(this)。在这里我们发现，其实在 dep.depend 完全可以直接可以调用 addSub 方法，但是为什么要经 watcher ，让 watcher 来控制调用该方法？我们可以观察到在 wather.addDep 中其实是对 dep 做了一系列的判断，然后符合判断后再调用该方法，防止重复添加。此处还有一点需要注意的是watcher 也记录了与 dep 的关联。这样做的目的是清除 watcher 时操作 dep 将 watcher 移除，具体调用方法如下：
```js
teardown () {
    if (this.active) {
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
```

#### dep.addSub()
```js
addSub (sub: Watcher) {
  // 收藏该 watcher
  this.subs.push(sub)
}
```
最终落地的方法 将 watcher 放入 dep 的看守列表。

### 依赖触发
在 state 发生改变时会进入到依赖触发阶段，依赖触发的起始点是在 Object.defineProperty 的 set 方法中。方法调用顺序为：

1. Object.defineProperty set
2. dep.notify()
3. watcher.update()
4. queueWatcher()
5. nextTick()
6. flushSchedulerQueue()
7. watcher.run()

#### queueWatcher()
queueWatcher 的目的是 将所有的 watcher 放到一个队列中。然后触发一次微任务。

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // 因为如果已经 flushing 那么 queue 是拍过顺序的，此处需要插入 watcher 到正确的顺序
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 在这里 保证只开启一次微任务 用 waiting 状态来加锁
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
#### nextTick()
将 flushSchedulerQueue 放入本轮微任务的 callback 队列，触发本轮微任务的执行。

#### flushSchedulerQueue()
按照顺序执行 watcher，然后还原案发现场。

1. 设置全局 flushing 状态为 true，优化在 queueWatcher 时 添加 watcher 的策略。flushing 过再插入需要按照正确的顺序插入。保证 render watcher 排在最后面。
2. 将 queue 中的 watcher 排序，让 render watcher 放在最后面
3. 循环 watcher 执行 watcher.run()
4. 清空队列，重置状态（还原案发现场）

#### watcher.run()
获取当前 value ，执行 watcher 的 callback
