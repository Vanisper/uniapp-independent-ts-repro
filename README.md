# UniApp 微信独立分包 TypeScript 最小复现

在官方 Vue 3 + TypeScript CLI 模板上，新增一个独立分包和一个 7 行的 `<script setup lang="ts">` 页面，即可触发微信小程序构建错误。当前 `main` 分支保留失败场景。

## 复现步骤

在项目目录执行：

```sh
npm ci
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

## 最小改动

相对 `official-template` 标签，复现代码只有两处改动。

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

`vite.config.ts`、`App.vue`、`main.ts`、manifest 和主包页面保留官方模板内容。项目使用独立安装的 npm 依赖，无分享插件、uni-helper、monorepo、依赖补丁或自定义编译配置。

## 对照验证

以下结果于 2026-10-03 在本项目实测：

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

这些版本由官方模板和 UVM 安装得到，未复制业务项目的依赖配置。

## 已定位的编译链路

5.26 [发布源码](https://github.com/dcloudio/uni-app/commit/724a07c41c10ac214e258bf444d6b855ed491c5b)中，独立分包会在 Vue 脚本模块 ID 末尾[追加 `uni_mp_independent_root`](https://github.com/dcloudio/uni-app/blob/724a07c41c10ac214e258bf444d6b855ed491c5b/packages/uni-cli-shared/src/json/mp/subpackage.ts#L219-L224)，而 esbuild 的[语言筛选规则](https://github.com/dcloudio/uni-app/blob/724a07c41c10ac214e258bf444d6b855ed491c5b/packages/vite-plugin-uni/src/config/index.ts#L47-L54)为 `/\.(tsx?|jsx|uts)$/`。

追加参数后，脚本 ID 不再以 `.ts` 结尾；Vite 去掉查询参数后又只剩 `.vue`，两种匹配都无法选中 TS 脚本。编译器生成的类型标注未被擦除，后续 JavaScript 解析阶段报错。交付版本保留官方配置，便于直接验证这一问题。

## 版本管理

Git 历史分为三步：官方模板与锁文件、最小复现代码、复现说明。

- `official-template`：已验证能够构建微信小程序的 5.26 官方模板基线
- `minimal-repro`：包含最小复现和验证说明的交付版本

可查看复现代码的完整差异：

```sh
git diff official-template -- src/pages.json src/packages/independent/index.vue
```

仓库未配置远程地址；`node_modules` 和 `dist` 不纳入 Git。
