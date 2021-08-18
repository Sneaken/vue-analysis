## 作用

检查是不是用 yarn 来安装的

必须要用 yarn 来安装， 因为用到了 yarn workspaces 的概念

monorepo

> 在版本控制系统中，monorepo 是一种软件开发策略，其中许多项目的代码存储在同一存储库中。

## 题外话

### script.preinstall 的作用

```bash
# 克隆项目
git clone https://github.com/vuejs/vue-next.git
cd vue-next
# 安装包
yarn
## 此时 你会发现运行了 node ./script/checkYarn.js 的命令
```

检查发现 package.json 中 的 script 中定义了一下

```json5
{
  "script": {
    // 这个是 npm 钩子 在 install 之前执行
    // 如果想要跳过这个钩子 可以使用 yarn --ignore-scripts
    "preinstall": "node ./scripts/checkYarn.js"
  }
}
```
