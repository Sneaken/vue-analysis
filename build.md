## 作用

打包文件

### 分块解析

#### buildAll

```js
async function buildAll(targets) {
  await runParallel(require('os').cpus().length, targets, build);
}
```

#### runParallel

> 有点东西的函数， 我称之为影分身之术

```js
async function runParallel(maxConcurrency, source, iteratorFn) {
  const ret = [];
  const executing = [];

  for (const item of source) {
    // 精髓
    const p = Promise.resolve().then(() => iteratorFn(item, source));
    ret.push(p);

    if (maxConcurrency <= source.length) {
      // 打包结束了以后, 从执行队列中删除
      const e = p.then(() => executing.splice(executing.indexOf(e), 1));
      // 添加到执行队列中
      executing.push(e);

      if (executing.length >= maxConcurrency) {
        // 并发数达到上限后，开始执行队列。
        // 一旦有执行完的，继续加入到队列中。
        await Promise.race(executing);
      }
    }
  }
  // 等待全部执行完
  return Promise.all(ret);
}
```
