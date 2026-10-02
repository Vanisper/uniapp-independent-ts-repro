# UniApp 微信独立分包 TypeScript 最小复现

在官方 Vue 3 + TypeScript CLI 模板上，新增一个独立分包和一个 7 行的 `<script setup lang="ts">` 页面，即可触发微信小程序构建错误。`minimal-repro` 标签保留原始失败场景；当前 `main` 包含本地编译器补丁，构建可以成功。

## 验证修复

在项目目录执行：

```sh
npm ci
npm run build:mp-weixin
npm run type-check
```

`npm ci` 通过 `postinstall` 自动应用 `patches/` 中的补丁，构建和类型检查均应成功。

## 复现原始问题

完成依赖安装后，可以在当前版本撤销已安装的补丁进行对照：

```sh
npm exec -- patch-package --reverse --error-on-fail
npm run build:mp-weixin
```

复现发生在 CLI 编译阶段，无需填写微信 AppID 或启动微信开发者工具。

预期：构建成功，生成独立分包页面。

实际：退出码为 `1`，输出如下（文件路径已改为项目相对路径）：

```text
Compiler version: 5.26（vue3）
Compiling...
x Build failed in 94ms
[uni:mp-using-component] Unexpected token, expected "," (11:12)
file: src/packages/independent/index.vue?vue&type=script&setup=true&lang.ts&uni_mp_independent_root=packages%2Findependent:11:12
Build failed with errors.
```

构建耗时可能变化；关键错误是 `Unexpected token`，对应的脚本模块 ID 同时包含 `lang.ts` 和 `uni_mp_independent_root`。

对照完成后恢复补丁：

```sh
npm run postinstall
npm run build:mp-weixin
```

也可检出 `minimal-repro` 标签，执行 `npm ci` 后直接构建，该版本尚未引入补丁。

## 最小改动

相对 `official-template` 标签，原始复现代码只有两处改动，补丁没有修改这两处代码。

在 `src/pages.json` 新增：

```json
{
  "subPackages": [
    {
      "root": "packages/independent",
      "independent": true,
      "pages": [{ "path": "index" }]
    }
  ]
}
```

新增 `src/packages/independent/index.vue`：

```vue
<script setup lang="ts">
const message = 'Independent TS page'
</script>

<template>
  <view>{{ message }}</view>
</template>
```

源码没有类型注解。仅指定 `lang="ts"`，编译器生成的渲染函数就包含 TS 类型，因此仍会触发问题。

`vite.config.ts`、`App.vue`、`main.ts`、manifest 和主包页面保留官方模板内容。项目使用独立安装的 npm 依赖，无分享插件、uni-helper、monorepo 或自定义编译配置。

## 编译器补丁

补丁文件为 [`patches/@dcloudio+vite-plugin-uni+3.0.0-5020620260917001.patch`](patches/@dcloudio+vite-plugin-uni+3.0.0-5020620260917001.patch)，适用于本项目锁定的 5.26 Vue 3 编译器。

仅修改发布包 `dist/config/index.js` 中默认 UniApp 分支的一行 `esbuild.include`：

```diff
- : /\.(tsx?|jsx|uts)$/,
+ : /(?:\.(?:tsx?|jsx|uts)$|[?&]lang\.(?:tsx?|jsx|uts)(?:&|$))/,
```

保留原有语言后缀匹配，同时识别 `lang.ts&uni_mp_independent_root=…` 这样的语言查询参数。新增匹配限定于 `lang` 参数，避免将其他查询参数值中的 `.ts` 当作脚本语言。

补丁用 [`patch-package`](https://github.com/ds300/patch-package) 保存，作为开发依赖固定为 `8.0.1`。`postinstall` 使用 `patch-package --error-on-fail`，补丁应用失败时终止安装。

## 对照验证

以下原始未修复结果于 2026-10-03 在本项目实测：

| 场景 | 结果 |
| --- | --- |
| 官方模板基线，无独立分包 | 微信小程序构建成功 |
| 独立分包 + 上述 TS setup 页面 | 连续三次构建失败，错误一致 |
| 同一页面，将 `independent` 改为 `false` | 微信小程序构建成功 |
| 保持 `independent: true`，仅移除 `lang="ts"` | 微信小程序构建成功 |
| 失败场景执行 `npm run type-check` | 类型检查通过 |

成功的对照均检查了 `dist/build/mp-weixin/app.json` 中的分包配置，以及 `packages/independent/index.js` 页面产物；JS 对照的产物仍包含 `independent: true`。

要复测普通分包对照，将 `src/pages.json` 中的 `"independent": true` 改为 `false`，再运行 `npm run build:mp-weixin`。完成后恢复为 `true`。

要复测 JS 对照，将独立分包页面的 `<script setup lang="ts">` 改为 `<script setup>`，再运行同一构建命令。完成后恢复 `lang="ts"`。

以上验证针对这里的 TS setup 页面，不据此断言所有 TS 文件或所有 TS 页面形态都失败。

补丁验证也于 2026-10-03 完成：

| 场景 | 结果 |
| --- | --- |
| 保持同一锁文件，仅撤销补丁 | 复现原始 `Unexpected token` 错误 |
| 重新应用补丁，原始 7 行页面保持不变 | 微信小程序构建成功 |
| 独立 TS 页增加显式 `: string` 类型注解 | 微信小程序构建成功 |
| 独立 TS 页增加原生 `onShareAppMessage` 钩子 | 微信小程序构建成功 |
| 普通分包 TS、独立分包 JS 两组对照 | 均构建成功 |
| `npm ci` 重新安装依赖 | 自动应用补丁成功 |
| `npm run type-check` | 类型检查通过 |

原始复现页面的最终产物仍包含 `independent: true`，并生成独立页面与分包内的 `common/vendor.js`。验证结束后已恢复原始复现源码，现有依赖版本保持不变。

## 官方来源与版本

项目按[官方 CLI 文档](https://uniapp.dcloud.net.cn/quickstart-cli.html#创建uni-app)创建：

```sh
npx --yes degit dcloudio/uni-preset-vue#vite-ts uniapp-independent-ts-repro
cd uniapp-independent-ts-repro
npx --yes @dcloudio/uvm@latest --manager npm
```

模板来源固定于 [`vite-ts` 的 `6fb81ac3c5736b8b0a83e667b3ed90223d458dd8`](https://github.com/dcloudio/uni-preset-vue/tree/6fb81ac3c5736b8b0a83e667b3ed90223d458dd8)。该模板当时仍使用 5.24；通过官方 UVM 升级至 2026-10-03 查询时的最新稳定版 5.26。稳定版以[官方 release.json](https://download1.dcloud.net.cn/hbuilderx/release.json)为依据，HBuilderX 版本为 `5.26.2026091802`。

升级时首次 npm 安装发生网络中断，随后使用同一官方工具指定已核实的稳定版本重试，完成后检查了全部 20 个编译器相关依赖的声明版本与安装版本：

```sh
npx --yes @dcloudio/uvm@latest 3.0.0-5020620260917001 --manager npm
```

当时 `@dcloudio/uvm@latest` 解析为 `0.3.1`。交付项目已提交 `package-lock.json`，复现时使用 `npm ci`，无需再次升级。

| 项目 | 实测版本 |
| --- | --- |
| Node.js | `22.22.2` |
| npm | `10.9.7` |
| UniApp 编译器 | `5.26（vue3）` |
| DCloud 编译器相关依赖 | `3.0.0-5020620260917001` |
| Vite | `5.2.8` |
| Rollup | `4.14.3` |
| Vue / `@vue/compiler-sfc` | `3.4.21` |
| `@vue/runtime-core`（根开发依赖） | `3.5.43` |
| TypeScript | `4.9.5` |
| vue-tsc | `1.8.27` |
| patch-package（本地修复工具） | `8.0.1` |

框架版本由官方模板和 UVM 安装得到，未复制业务项目的依赖配置；修复额外添加了 patch-package。

## 已定位的编译链路

5.26 [发布源码](https://github.com/dcloudio/uni-app/commit/724a07c41c10ac214e258bf444d6b855ed491c5b)中，独立分包会在 Vue 脚本模块 ID 末尾[追加 `uni_mp_independent_root`](https://github.com/dcloudio/uni-app/blob/724a07c41c10ac214e258bf444d6b855ed491c5b/packages/uni-cli-shared/src/json/mp/subpackage.ts#L219-L224)，而 esbuild 的[语言筛选规则](https://github.com/dcloudio/uni-app/blob/724a07c41c10ac214e258bf444d6b855ed491c5b/packages/vite-plugin-uni/src/config/index.ts#L47-L54)为 `/\.(tsx?|jsx|uts)$/`。

未应用补丁时，追加参数后脚本 ID 不再以 `.ts` 结尾；Vite 去掉查询参数后又只剩 `.vue`，两种匹配都无法选中 TS 脚本。编译器生成的类型标注未被擦除，后续 JavaScript 解析阶段报错。上面的撤销补丁步骤可直接验证这一问题。

## 版本管理

Git 历史保留官方模板与锁文件、最小复现代码、复现说明三个原始提交，编译器补丁另行提交。

- `official-template`：已验证能够构建微信小程序的 5.26 官方模板基线
- `minimal-repro`：尚未应用补丁的原始失败版本
- `patched-repro`：包含编译器补丁与修复验证说明的版本

可查看复现代码的完整差异：

```sh
git diff official-template -- src/pages.json src/packages/independent/index.vue
```

`node_modules` 和 `dist` 不纳入 Git，依赖补丁通过 `patches/` 纳入版本管理。
