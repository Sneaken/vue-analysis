## 作用

1. 运行测试用例
2. 更新版本

### 前情提示

#### semver

> [semver](https://www.npmjs.com/package/semver)

1.  semver.prerelease 获取 预发布版本的 `希腊字母版本号`(这个叫啥？？？) 和 `版本号`

    如 semver.prerelease('3.2.2-alpha.3') 返回 ['alpha', 3]

2.  semver.inc 接受一个额外的 identifier 字符串参数，该参数将附加字符串的值作为预发布标识符

    如 semver.inc ('1.2.3', 'prerelease', 'beta') => '1.2.4-beta.0'

3.  semver.valid Return the parsed version, or null if it's not valid.

    ```js
    const semver = require('semver');

    semver.valid('1.2.3'); // '1.2.3'
    semver.valid('1.2.4-beta.0'); // '1.2.4-beta.0'
    semver.valid('a.b.c'); // null
    semver.valid(semver.coerce('v2')); // '2.0.0'
    semver.valid(semver.coerce('42.6.7.9.3-alpha')); // '42.6.7'
    ```

#### enquirer

> [enquirer](https://www.npmjs.com/package/enquirer)

美化 cli

### 分块解析

#### 1. main

```js
async function main() {
  // 命令行的第一个参数 指定的版本号
  let targetVersion = args._[0];

  if (!targetVersion) {
    // 没有明确的版本号的时候， 提供一些建议
    const { release } = await prompt({
      type: 'select',
      name: 'release',
      message: 'Select release type',
      // 这边的版本号被括号包裹住了
      choices: versionIncrements.map((i) => `${i} (${inc(i)})`).concat(['custom']),
    });

    // 选择了 custom 后
    if (release === 'custom') {
      targetVersion = (
        await prompt({
          type: 'input',
          name: 'version',
          message: 'Input custom version',
          // 默认选项
          initial: currentVersion,
        })
      ).version;
    } else {
      // 从括号中间取版本号
      targetVersion = release.match(/\((.*)\)/)[1];
    }
  }

  // 检验提取的版本号是否正确
  if (!semver.valid(targetVersion)) {
    throw new Error(`invalid target version: ${targetVersion}`);
  }

  const { yes } = await prompt({
    type: 'confirm',
    name: 'yes',
    message: `Releasing v${targetVersion}. Confirm?`,
  });

  if (!yes) {
    return;
  }
  // 上面已经确定了版本号， 但是还没修改到 package.json 中去
  // 确认过版本号以后开始跑测试用例

  // run tests before release
  step('\nRunning tests...');
  if (!skipTests && !isDryRun) {
    await run(bin('jest'), ['--clearCache']);
    await run('yarn', ['test', '--bail']);
  } else {
    console.log(`(skipped)`);
  }

  // 开始更新版本号
  // update all package versions and inter-dependencies
  step('\nUpdating cross dependencies...');
  updateVersions(targetVersion);

  // 开始打包
  // build all packages with types
  step('\nBuilding all packages...');
  if (!skipBuild && !isDryRun) {
    await run('yarn', ['build', '--release']);
    // test generated dts files
    step('\nVerifying type declarations...');
    await run('yarn', ['test-dts-only']);
  } else {
    console.log(`(skipped)`);
  }

  // 更新 CHANGELOG.md
  // 这边可以借鉴一下（前提是你 commit 的消息是规范的， 所以 commit 的时候还做了校验）
  // generate changelog
  await run(`yarn`, ['changelog']);

  // 干货干货
  // 因为修改完了 package.json 和 CHANGELOG.md，所以这边 git 是有修改记录的
  const { stdout } = await run('git', ['diff'], { stdio: 'pipe' });
  if (stdout) {
    // 有东西提交的时候 git add -A && git commit -m 'release: v${版本号}'
    step('\nCommitting changes...');
    await runIfNotDry('git', ['add', '-A']);
    await runIfNotDry('git', ['commit', '-m', `release: v${targetVersion}`]);
  } else {
    console.log('No changes to commit.');
  }

  // publish packages
  step('\nPublishing packages...');
  // for of 或者 for 都能异步遍历
  for (const pkg of packages) {
    await publishPackage(pkg, targetVersion, runIfNotDry);
  }

  // 开始推代码
  // push to GitHub
  step('\nPushing to GitHub...');
  // 打标签
  await runIfNotDry('git', ['tag', `v${targetVersion}`]);
  // 推标签
  await runIfNotDry('git', ['push', 'origin', `refs/tags/v${targetVersion}`]);
  // 推代码
  await runIfNotDry('git', ['push']);

  if (isDryRun) {
    console.log(`\nDry run finished - run git diff to see package changes.`);
  }

  if (skippedPackages.length) {
    console.log(
      chalk.yellow(`The following packages are skipped and NOT published:\n- ${skippedPackages.join('\n- ')}`),
    );
  }
  console.log();
}
```

#### updateVersions 根据指定版本号 更新 version 字段

```js
function updateVersions(version) {
  // 1. update root package.json
  updatePackage(path.resolve(__dirname, '..'), version);
  // 2. update all packages
  packages.forEach((p) => updatePackage(getPkgRoot(p), version));
}
```

#### updatePackage 更新 package.json

```js
function updatePackage(pkgRoot, version) {
  const pkgPath = path.resolve(pkgRoot, 'package.json');
  const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8'));
  // 更新 vue 的版本
  pkg.version = version;
  // 更新符合要求的依赖包的版本
  updateDeps(pkg, 'dependencies', version);
  updateDeps(pkg, 'peerDependencies', version);
  // 这边还格式化了一下
  fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2) + '\n');
}
```

#### updateDeps 更新指定字段的 version

```js
function updateDeps(pkg, depType, version) {
  const deps = pkg[depType];
  if (!deps) return;
  Object.keys(deps).forEach((dep) => {
    // 这边的 `packages` 指的是 packages/* 下面的包
    if (dep === 'vue' || (dep.startsWith('@vue') && packages.includes(dep.replace(/^@vue\//, '')))) {
      // 符合要求就更新版本
      // 从这里我们可以知道 vue 和 其子包的版本是一致的
      console.log(chalk.yellow(`${pkg.name} -> ${depType} -> ${dep}@${version}`));
      deps[dep] = version;
    }
  });
}
```

#### publishPackage 发布包

```js
async function publishPackage(pkgName, version, runIfNotDry) {
  if (skippedPackages.includes(pkgName)) {
    return;
  }
  const pkgRoot = getPkgRoot(pkgName);
  const pkgPath = path.resolve(pkgRoot, 'package.json');
  const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8'));
  if (pkg.private) {
    return;
  }

  // For now, all 3.x packages except "vue" can be published as
  // `latest`, whereas "vue" will be published under the "next" tag.
  let releaseTag = null;
  if (args.tag) {
    releaseTag = args.tag;
  } else if (version.includes('alpha')) {
    releaseTag = 'alpha';
  } else if (version.includes('beta')) {
    releaseTag = 'beta';
  } else if (version.includes('rc')) {
    releaseTag = 'rc';
  } else if (pkgName === 'vue') {
    // TODO remove when 3.x becomes default
    releaseTag = 'next';
  }

  // TODO use inferred release channel after official 3.0 release
  // const releaseTag = semver.prerelease(version)[0] || null

  step(`Publishing ${pkgName}...`);
  try {
    await runIfNotDry(
      'yarn',
      ['publish', '--new-version', version, ...(releaseTag ? ['--tag', releaseTag] : []), '--access', 'public'],
      {
        cwd: pkgRoot,
        stdio: 'pipe',
      },
    );
    console.log(chalk.green(`Successfully published ${pkgName}@${version}`));
  } catch (e) {
    if (e.stderr.match(/previously published/)) {
      console.log(chalk.red(`Skipping already published: ${pkgName}`));
    } else {
      throw e;
    }
  }
}
```

#### runIfNotDry --dry 根据该参数来确实是否只是展示，还是要实际执行

```js
const isDryRun = args.dry;
const run = (bin, args, opts = {}) => execa(bin, args, { stdio: 'inherit', ...opts });
// 可以理解为 预检
const dryRun = (bin, args, opts = {}) => console.log(chalk.blue(`[dryrun] ${bin} ${args.join(' ')}`), opts);
const runIfNotDry = isDryRun ? dryRun : run;
```

### 总结

1. 整个流程很清楚，甚至可以直接照搬到项目中去

2. 整个流程

   确认发布的版本 -> 跑测试用例（如果有的话） -> -> 更新 package.json 的版本 -> 可能需要打包 -> 更新 changelog 内容 -> 提交变更 -> 打标签 -> 推标签 -> 推代码 -> 发包
