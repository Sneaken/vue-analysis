## 作用

打包文件

### 前情提示

1. `fs` 不是原生模块，而是从 `fs-extra` 导出的

### 分块解析

#### argv

```js
// 模块解析包
const argv = require('minimist')(process.argv.slice(2));
```

[minimist npm link](https://www.npmjs.com/package/minimist)

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

#### run

> 该文件主函数

```js
async function run() {
  if (isRelease) {
    // remove build cache for release builds to avoid outdated enum values
    // 如果是发版需要清理缓存
    await fs.remove(path.resolve(__dirname, '../node_modules/.rts2_cache'));
  }
  // 判断有没有指定的模块
  if (!targets.length) {
    await buildAll(allTargets);
    checkAllSizes(allTargets);
  } else {
    await buildAll(fuzzyMatchTarget(targets, buildAllMatching));
    checkAllSizes(fuzzyMatchTarget(targets, buildAllMatching));
  }
}
```

#### build

> 打包主函数

```js
async function build(target) {
  // 由于项目是 node scripts/build.js
  // 所以 path 的基地址是项目的根目录
  const pkgDir = path.resolve(`packages/${target}`);
  // 所以 pkgDir 是完整的模块地址
  const pkg = require(`${pkgDir}/package.json`);

  // if this is a full build (no specific targets), ignore private packages
  if ((isRelease || !targets.length) && pkg.private) {
    return;
  }

  // if building a specific format, do not remove dist.
  // 是否打指定格式的包
  if (!formats) {
    // 不指定格式的话就删除旧的打包目录，准备重新打包
    await fs.remove(`${pkgDir}/dist`);
  }

  const env = (pkg.buildOptions && pkg.buildOptions.env) || (devOnly ? 'development' : 'production');
  // execa 用法 (执行的程序, [给程序传的参数...])
  await execa(
    'rollup',
    [
      '-c',
      '--environment',
      [
        `COMMIT:${commit}`,
        `NODE_ENV:${env}`,
        `TARGET:${target}`,
        formats ? `FORMATS:${formats}` : ``,
        buildTypes ? `TYPES:true` : ``,
        prodOnly ? `PROD_ONLY:true` : ``,
        sourceMap ? `SOURCE_MAP:true` : ``,
      ]
        .filter(Boolean)
        .join(','),
    ],
    // 子进程使用父类的 stdio (标准输入输出)
    { stdio: 'inherit' },
  );

  if (buildTypes && pkg.types) {
    console.log();
    console.log(chalk.bold(chalk.yellow(`Rolling up type definitions for ${target}...`)));

    // build types
    const { Extractor, ExtractorConfig } = require('@microsoft/api-extractor');

    const extractorConfigPath = path.resolve(pkgDir, `api-extractor.json`);
    const extractorConfig = ExtractorConfig.loadFileAndPrepare(extractorConfigPath);
    const extractorResult = Extractor.invoke(extractorConfig, {
      localBuild: true,
      showVerboseMessages: true,
    });

    if (extractorResult.succeeded) {
      // concat additional d.ts to rolled-up dts
      const typesDir = path.resolve(pkgDir, 'types');
      if (await fs.exists(typesDir)) {
        const dtsPath = path.resolve(pkgDir, pkg.types);
        const existing = await fs.readFile(dtsPath, 'utf-8');
        const typeFiles = await fs.readdir(typesDir);
        const toAdd = await Promise.all(
          typeFiles.map((file) => {
            return fs.readFile(path.resolve(typesDir, file), 'utf-8');
          }),
        );
        await fs.writeFile(dtsPath, existing + '\n' + toAdd.join('\n'));
      }
      console.log(chalk.bold(chalk.green(`API Extractor completed successfully.`)));
    } else {
      console.error(
        `API Extractor completed with ${extractorResult.errorCount} errors` +
          ` and ${extractorResult.warningCount} warnings`,
      );
      process.exitCode = 1;
    }

    await fs.remove(`${pkgDir}/dist/packages`);
  }
}
```

#### checkAllSizes

> 输出全部模块的大小

```js
function checkAllSizes(targets) {
  if (devOnly) {
    return;
  }
  console.log();
  for (const target of targets) {
    checkSize(target);
  }
  console.log();
}
```

#### checkSize

> 输出当前模块的大小

```js
function checkSize(target) {
  const pkgDir = path.resolve(`packages/${target}`);
  checkFileSize(`${pkgDir}/dist/${target}.global.prod.js`);
  checkFileSize(`${pkgDir}/dist/${target}.runtime.global.prod.js`);
}
```

### checkFileSize

> 输出文件大小

```js
function checkFileSize(filePath) {
  if (!fs.existsSync(filePath)) {
    return;
  }
  const file = fs.readFileSync(filePath);
  const minSize = (file.length / 1024).toFixed(2) + 'kb';
  const gzipped = gzipSync(file);
  const gzippedSize = (gzipped.length / 1024).toFixed(2) + 'kb';
  const compressed = compress(file);
  const compressedSize = (compressed.length / 1024).toFixed(2) + 'kb';
  console.log(
    `${chalk.gray(
      chalk.bold(path.basename(filePath)),
    )} min:${minSize} / gzip:${gzippedSize} / brotli:${compressedSize}`,
  );
}
```
