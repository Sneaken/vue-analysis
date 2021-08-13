# vue3/packages/shared 模块

## /shared/src/index

### `__DEV__`

这里的 `__DEV__` 不能说是环境变量，这是 rollup 的标记位, 根据打包格式的不同会被替换为不同的值

```
// rollup.config.ts
const replacements = {
    // ...ignore
    __DEV__: isBundlerESMBuild
      ? // preserve to be handled by bundlers
        `(process.env.NODE_ENV !== 'production')`
      : // hard coded dev/prod builds
        !isProduction,
  }
  // allow inline overrides like
  //__RUNTIME_COMPILE__=true yarn build runtime-core
  Object.keys(replacements).forEach(key => {
    if (key in process.env) {
      replacements[key] = process.env[key]
    }
  })
  return replace({
    values: replacements,
    preventAssignment: true
  })
```

### EMPTY_OBJ 空对象

```ts
// 开发环境的 EMPTY_OBJ 是个冻结的对象，无法修改
// 冻结的对象只是最外层不能无法修改
// 但是修改了不会报错
const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__ ? Object.freeze({}) : {};

// 例子
const EMPTY_OBJ_1 = Object.freeze({});
EMPTY_OBJ_1.name = 'xx';
console.log(EMPTY_OBJ_1.name); // undefined

const EMPTY_OBJ_2 = Object.freeze({ props: { mp: 'xxxx' } });
EMPTY_OBJ_2.props.name = 'xx';
EMPTY_OBJ_2.props2 = 'props2';
console.log(EMPTY_OBJ_2.props.name); // 'xx'
console.log(EMPTY_OBJ_2.props2); // undefined
console.log(EMPTY_OBJ_2);
// {
//   "props": {
//     "mp": "xxxx",
//     "name": "xx"
//   }
// }
```

### EMPTY_ARR 空数组

```ts
// 开发环境是个冻结的数组, 无法修改
// 冻结的数组只是最外层不能无法修改
// 冻结的数组直接修改（意思是不用数组方法）是不会报错的，不会成功
// 但是用了数组方法的话，会报错
const EMPTY_ARR = __DEV__ ? Object.freeze([]) : [];
// 例子：
EMPTY_ARR.push(2); // 报错 Uncaught TypeError: Cannot add property 1, object is not extensible
EMPTY_ARR.length = 3;
console.log(EMPTY_ARR.length); // 0
```

### NOOP 空函数

```ts
const NOOP = () => {};

// 很多库的源码中都有这样的定义函数，比如 jQuery、underscore、lodash 等
// 使用场景：1. 方便判断， 2. 方便压缩
// 1. 比如：
const instance = {
  render: NOOP,
};

// 条件
const dev = true;
if (dev) {
  instance.render = function () {
    console.log('render');
  };
}

// 可以用作判断。
if (instance.render === NOOP) {
  console.log('i');
}
// 2. 再比如：
// 方便压缩代码
// 如果是 function(){} ，不方便压缩代码
```

### NO always return false

```ts
// 可能是因为用的地方比较多， 所以抽成了一个函数？
const NO = () => false;
```

### isOn 判断字符串是不是 on 开头，并且 on 后首字母不是小写字母

```ts
// ^符号在开头，则表示是什么开头。而在其他地方是指非。
// [^a-z] 匹配不是小写字母
const onRE = /^on[^a-z]/;
const isOn = (key) => onRE.test(key);

// 例子：
isOn('onChange'); // true
isOn('onchange'); // false
isOn('on3change'); // true
```

### isModelListener 判断字符串是不是 'onUpdate:' 开头

```ts
// es6 语法
const isModelListener = (key: string) => key.startsWith('onUpdate:');
```

### extend 合并

```ts
// 猜测是用的地方比较多，提取出来，节省查找原型链的时间？
const extend = Object.assign;
```

### remove 从数组中删除指定元素

```ts
const remove = <T>(arr: T[], el: T) => {
  const i = arr.indexOf(el);
  if (i > -1) {
    arr.splice(i, 1);
  }
};
// 例子：
const arr = ['i', 'love', 'you'];
remove(arr, 'love');
console.log(arr); // ['i', 'you']
```

splice 其实是一个很耗性能的方法。删除数组中的一项，其他元素都要移动位置。

### hasOwn 是不是自己本身所拥有的属性(不是来自原型链上面的)

```ts
const hasOwnProperty = Object.prototype.hasOwnProperty;
const hasOwn = (val: object, key: string | symbol): key is keyof typeof val => hasOwnProperty.call(val, key);

// 例子：
// 特别提醒：__proto__ 是浏览器实现的原型写法，后面还会用到
// 现在已经有提供好几个原型相关的API
// Object.getPrototypeOf
// Object.setPrototypeOf
// Object.isPrototypeOf

// .call 则是函数里 this 显示指定以为第一个参数，并执行函数。

hasOwn({ __proto__: { a: 1 } }, 'a'); // false
hasOwn({ a: undefined }, 'a'); // true
hasOwn({}, 'a'); // false
hasOwn({}, 'hasOwnProperty'); // false
hasOwn({}, 'toString'); // false
```

### isArray 判断是不是数组

```ts
const isArray = Array.isArray;
```

### objectToString

```ts
// 猜测：用的地方计算量比较大，节省查找原型链的时间
const objectToString = Object.prototype.toString;
```

### toTypeString 返回 [object *]

```ts
const toTypeString = (value: unknown): string => objectToString.call(value);
```

#### 引申

顶级类型 unknown

> unknown 类型是 any 的类型安全版本。每当你想使用 any 时，应该先试着用 unknown。

在 any 允许我们做任何事的地方，unknown 的限制则大得多。
在对 unknown 类型的值执行任何操作之前，必须先限定其类型

### toRawType 转换得到原始类型

```ts
const toRawType = (value: unknown): string => {
  // extract "RawType" from strings like "[object RawType]"
  return toTypeString(value).slice(8, -1);
};
```

### isMap 判断是不是 Map 对象

```ts
const isMap = (val: unknown): val is Map<any, any> => toTypeString(val) === '[object Map]';
// 例子：
const map = new Map();
const o = { p: 'Hello World' };

map.set(o, 'content');
map.get(o); // 'content'
isMap(map); // true
```

### isSet 判断是不是 Set 对象

```ts
const isSet = (val: unknown): val is Set<any> => toTypeString(val) === '[object Set]';

// 例子：
const set = new Set();
isSet(set); // true
```

### isDate 判断是不是 Date 对象

```ts
const isDate = (val: unknown): val is Date => val instanceof Date;
// 不是很准 无法防止骚操作
isDate({__proto__ : new Date()); // true
```

### isFunction 判断是不是函数

> 这个是比较常用,相对来说兼容性比较好的

```ts
const isFunction = (val: unknown): val is Function => typeof val === 'function';
```

### isString 判断是不是字符串

```ts
const isString = (val: unknown): val is string => typeof val === 'string';
```

### isSymbol 判断是不是 Symbol 类型的数据

```ts
const isSymbol = (val: unknown): val is symbol => typeof val === 'symbol';
```

### isObject 判断是不是对象（不包括 null）

```ts
const isObject = (val: unknown): val is Record<any, any> => val !== null && typeof val === 'object';
```

### isPromise 判断是不是 Promise 对象

> [promise/A+ 规范](https://promisesaplus.com/)

```ts
// Promise 就是这样定义的似乎
// 如果一个对象 包含 then 方法 和 catch 方法 可以认为是 Promise 对象
const isPromise = <T = any>(val: unknown): val is Promise<T> => {
  return isObject(val) && isFunction(val.then) && isFunction(val.catch);
};
```

### isPlainObject 判断是不是普通对象

```ts
const isPlainObject = (val: unknown): val is object => toTypeString(val) === '[object Object]';
```

### isIntegerKey 判断 Key 是不是 字符串类型的数字

```ts
const isIntegerKey = (key: unknown) =>
  isString(key) && key !== 'NaN' && key[0] !== '-' && '' + parseInt(key, 10) === key;
```

### isReservedProp

```ts
const isReservedProp = /*#__PURE__*/ makeMap(
  // the leading comma is intentional so empty string "" is also included
  ',key,ref,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted',
);
```

### cacheStringFunction 高性能场景下的函数缓存实战

```ts
const cacheStringFunction = <T extends (str: string) => string>(fn: T): T => {
  // 一个纯粹的空对象
  const cache: Record<string, string> = Object.create(null);
  return ((str: string) => {
    const hit = cache[str];
    return hit || (cache[str] = fn(str));
  }) as any;
};

const camelizeRE = /-(\w)/g;
/**
 * 中划线命名转小驼峰
 * @private
 */
export const camelize = cacheStringFunction((str: string): string => str.replace(camelizeRE, (_, c) => (c ? c.toUpperCase() : ''))

const hyphenateRE = /\B([A-Z])/g;
/**
 * 驼峰转中划线
 * @private
 */
export const hyphenate = cacheStringFunction((str: string) => str.replace(hyphenateRE, '-$1').toLowerCase());

/**
 * 首字母大写
 * @private
 */
export const capitalize = cacheStringFunction((str: string) => str.charAt(0).toUpperCase() + str.slice(1));

/**
 * @private
 */
export const toHandlerKey = cacheStringFunction((str: string) => (str ? `on${capitalize(str)}` : ``));
```

### hasChanged 判断值是不是改变了

```ts
// compare whether a value has changed, accounting for NaN.
export const hasChanged = (value: any, oldValue: any): boolean => !Object.is(value, oldValue);

// Object.is() 方法判断两个值是否为同一个值。

// 使用场景
// 有没有发现和 watch 里面的很像？
```

#### 引申

[Object.is mdn docs](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/is)

```js
// polyfill
if (!Object.is) {
  Object.is = function (x, y) {
    // SameValue algorithm
    if (x === y) {
      // Steps 1-5, 7-10
      // Steps 6.b-6.e: +0 != -0
      return x !== 0 || 1 / x === 1 / y;
    } else {
      // Step 6.a: NaN == NaN
      return x !== x && y !== y;
    }
  };
}
```

### invokeArrayFns 执行数组里的函数

```ts
const invokeArrayFns = (fns: Function[], arg?: any) => {
  for (let i = 0; i < fns.length; i++) {
    fns[i](arg);
  }
};

// 批量调用生命周期函数
// 如批量调用 beforeDestroy hooks
// 如批量调用 updated hooks
```

### def

```ts
const def = (obj: object, key: string | symbol, value: any) => {
  Object.defineProperty(obj, key, {
    configurable: true,
    enumerable: false,
    value,
  });
};
```

#### 引申

属性描述符（特性）

1. value 当试图获取属性时所返回的值。
2. writable 该属性是否可写。
3. enumerable 该属性在 for in 循环中是否会被枚举。
4. configurable 该属性是否可被删除。
5. set() 该属性的更新操作所调用的函数。
6. get() 获取属性值时所调用的函数。

数据描述符（其中属性为：enumerable，configurable，value，writable）与存取描述符（其中属性为 enumerable，configurable，set()，get()）之间是有互斥关系的。在定义了 set()和 get()之后，描述符会认为存取操作已被定义了，其中再定义 value 和 writable 会引起错误。

### toNumber 尝试转换成数字

```ts
const toNumber = (val: any): any => {
  const n = parseFloat(val);
  // 如果 n 不是数字则原路返回，否则返回转换后的结果
  return isNaN(n) ? val : n;
};
```

#### 引申

判断是不是数字: isNaN
判断是不是 NaN: Number.isNaN

### getGlobalThis 获取 globalThis

> globalThis 不能直接使用， 毕竟不是所有的环境都实现了这个

```ts
let _globalThis: any;
export const getGlobalThis = (): any => {
  // 如果有缓存的结果就直接使用
  return (
    _globalThis ||
    (_globalThis =
      // 先判断环境 是否实现了 globalThis
      typeof globalThis !== 'undefined'
        ? globalThis
        : // 浏览器和 Web Worker 里面，self指向顶层对象, 但是 Node 没有 self
        typeof self !== 'undefined'
        ? // 确认是 浏览器 或者 web worker 环境
          self
        : // 浏览器里面，顶层对象是 window，但 Node 和 Web Worker 没有window。
        typeof window !== 'undefined'
        ? // 确认是 浏览器
          window
        : // node 环境里面， 顶层对象是 global
        typeof global !== 'undefined'
        ? // 确认是 node 环境
          global
        : {})
  );
};
```
