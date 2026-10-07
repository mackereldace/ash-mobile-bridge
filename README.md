# DSH Mobile · 预览桥（`@dsh-mobile/bridge`）

给 **DeepSeek Harness**（DSH）用的插件：把 DSH **自带的文档预览能力**
（`ctx.sidebarRight.openResource`）暴露给 DSH Mobile 的手机外壳 ——
手机端文件页点开 `.md` / 代码 / 图片 / PDF 时，走的就是它。

本包**不是库**，是一个 **DSH 客户端 bundle 插件**：宿主那半是空实现（`lib/index.js:11-12`），
真正的能力在浏览器半（`lib/client.js`）——它在页面里拿到 DSH 的依赖注入容器后挂出：

```js
globalThis.__DSHM_DSH_PREVIEW__ = { ready, open({ path, line }), state() }
// 定义：lib/client.js:19-22；挂载：lib/client.js:145
```

## 为什么这个包必须自己是一个 bundle

DSH 用官方插件管理从 git 装包时，给 profile 开着 **`blockExoticSubdeps`** ——
插件**不许**在**子依赖**里出现 git 这类外来来源（顶层 `github:` 允许）。原本桥是
`@dsh-mobile/host` 的一个依赖，实测报：

```
[ERR_PNPM_EXOTIC_SUBDEP] Exotic dependency "@dsh-mobile/bridge" (resolved via git-repository)
is not allowed in subdependencies when blockExoticSubdeps is enabled
```

（那段推理与报错原文见本仓 `cordis.patch.yml:1-19`；同一道墙的实测记录见
`10-交接文档.md:2591`，本地文档，未随本仓发布。）

而桥是**客户端插件**、不是库，没法当库内联进去 ⇒ 只能**自己声明成一个 bundle**
（`package.json:17-20` 的 `dsh.bundle`），用户**装两次**（宿主一次、桥一次）。

**代价如实说**：官方那条路是**两条命令**。这不是推测，是实测读数 ——
`10-交接文档.md:2604`：

```
dsh.profile.bundles = [dsh-base, dsh-web-app, @dsh-mobile/host, @dsh-mobile/bridge] ✓
```

## 仓库里有什么

| 文件 | 是什么 |
|---|---|
| `package.json` | 包清单：`name` / `version` / `main` / `exports` / `dsh.bundle` / `dsh.client` / `files` |
| `lib/index.js` | 宿主半：空的 `apply()`（539 B） |
| `lib/client.js` | 浏览器半：唯一的实现（7,258 B） |
| `cordis.patch.yml` | 插件往 profile 注入的那一行（`- insert:` / `id: mobile-preview-bridge`，`cordis.patch.yml:24-26`） |

`files` 白名单**必须**带上 `cordis.patch.yml`（`package.json:29-32`）：git 装包同样按 `files` 筛，
漏了它装完会报 `failed to read overlay …/cordis.patch.yml: ENOENT`，而报错**不会**提示
「是你自己排除的」（`10-交接文档.md:2592` 撞过这道墙）。

## 怎么装

★ **前置**：本包**单独装上没有用** —— 消费方是宿主插件 `@dsh-mobile/host`：它注入页面的
`boot.js` 里就有一行 `var bridge = globalThis.__DSHM_DSH_PREVIEW__`
（`dsh-mobile:packages/client/src/boot.js:20553`）。⇒ **两个包都要装**。

### 路线 A · 官方插件管理（DSH 0.17+）

```bash
# ① 宿主插件
dsh plugin --profile web add https://github.com/mackereldace/dsh-mobile-plugin

# ② 桥（本仓）
dsh plugin --profile web add https://github.com/mackereldace/ash-mobile-bridge
```

出处与状态：

- 命令形式「`dsh plugin --profile web add <本包>`」：`dsh-mobile:packages/host/cordis.patch.yml:8`
  （`dsh-mobile` = `github.com/mackereldace/dsh-mobile`）。
- 等价入口：DSH 的 **Plugins 页面 → Git 栏**填同一个仓库地址（`dsh-mobile-plugin:README.md:8-12`；
  `dsh-mobile-plugin:README.md:16-17` 就是①那条命令）。
- ② 用的是**裸仓库 URL**（不带 `#path:`），依据是 `dsh-mobile-plugin:README.md:16-17`
  的既有写法 —— 那个仓同样是「包在仓库根」的 bundle。★ 但②这条命令**我没有实跑**
  （官方那条路的实测读数是在 `dsh-mobile` 的 `#path:/packages/bridge` 形态上拿到的，
  `10-交接文档.md:2601-2607`）。见文末「已知未验」。

装完**不用手改** `cordis.patch.yml`：本包是 bundle，它自带的那一层会**自动并进** DSH 的补丁栈
（`dsh.bundle.patch` = `./cordis.patch.yml`，`package.json:18-20`）。

之后还要补一次**机器专属配置**（bundle 只给「行」、不给 `config`；官方装完
`listener.enabled=false` ⇒ **手机连不上**，`10-交接文档.md:2611-2614`）：

```bash
node scripts/install-host-plugin.mjs --config-only --dsh-home ~/.dsh --profile web \
  --listener --listener-plain 0.0.0.0:3081 --listener-tls 0.0.0.0:3443 \
  --phone-base-url https://<本机局域网IP>:3443
```

出处：`24-0.17官方插件管理-调研与接入方案.md:332-339`（本地文档，§10.3「三条命令」；
`--config-only` 的用途另见 `dsh-mobile:scripts/install-host-plugin.mjs:104-119`，
用法串见 `:147-151`）。★ 这一步**需要 `dsh-mobile` 的检出** —— 脚本在里面、不在本仓；
这个缺口上游文档自己也写着（同一节末尾）。

### 路线 B · 自研安装器（`10-交接文档.md` §3.1 的口径）

在 `dsh-mobile` 检出目录里：

```bash
cd <dsh-mobile 检出目录>
node scripts/install-host-plugin.mjs --profile web --dsh-home ~/.dsh

# 变体：把手机入口搬进插件进程内监听
node scripts/install-host-plugin.mjs --profile web --dsh-home ~/.dsh \
  --listener --listener-plain 0.0.0.0:3081 --listener-tls 0.0.0.0:3443
```

它把三个包 —— `@dsh-mobile/protocol` / `@dsh-mobile/host` / `@dsh-mobile/bridge` —— **一起**拷进 profile：
`dsh-mobile:scripts/install-host-plugin.mjs:222-226`。

出处：`10-交接文档.md:152-167`（§3.1 安装器口径原文；那里的 `--trusted-host …` 是写作时那台机器
专属的信任项，**换成你自己的**）+ `dsh-mobile:scripts/install-host-plugin.mjs:15-20`（脚本自带的用法头）。
装完 **退出并重开 DeepSeek Harness**（`10-交接文档.md:174`）。

## 依赖什么

- **零 npm 依赖**：本包 `package.json` 里既没有 `dependencies`、也没有 `peerDependencies`
  （整个文件 34 行，可自行核对）。这正是官方那条路能走通的原因：`blockExoticSubdeps` 只管
  **子**依赖，顶层 git 装包没关系（`10-交接文档.md:2591`）。
- **注入 DSH 自带的两个模块**（是 DSH 提供的，不是本包的依赖）：
  `@deepseek-ai/dsh-client-ui-sidebar`、`@deepseek-ai/dsh-api-session-controller`（`package.json:21-27`）。
- **必须同时装宿主插件 `@dsh-mobile/host`**（见上文前置）。桥是被动暴露能力的一方：
  DSH 的模块加载器要求客户端 bundle 的注册**与宿主侧声明的 graph 行匹配**
  （`lib/client.js:12-15`；加载调用本体在 `lib/client.js:29-31`），那一行就在本包的
  `cordis.patch.yml:24-26` 里。
- `license: MIT`（`package.json:33`）；`private: true`（`package.json:4`）**只挡 `npm publish`**，
  走 git 安装（本 README 的两条路线）不受影响。要发 npm 得先改包名并翻成公开
  （`dsh-mobile:scripts/prepare-npm.mjs:69` 就是这么做的）。

## 本仓是怎么来的

`dsh-mobile` 仓提交 `c9f638c` 里 `packages/bridge/` 的**逐字节副本**，四个文件一个字节没改：

| 文件 | sha256（本仓 = 源包） |
|---|---|
| `package.json` | `8947d19a4ae42b8c10140233a502fc6681bf76c0dceeade3b386edc2068c7e9d` |
| `cordis.patch.yml` | `5e706e520659df27767bea84b7cffb86f3f26474a3ec410f8d07ca29032913eb` |
| `lib/client.js` | `4a58e0adad7aafe067eb38975561925adb55c3ecf9421da035f3cbdacab7d1da` |
| `lib/index.js` | `d93f6d305004e73a86d45a0b1163a770a131cac9cac5ffe9546fb907ff535c02` |

版本保持 **`0.0.1`**（`package.json:3`），与源包一致，没有升版。
`files` 白名单（`lib` + `cordis.patch.yml`）已经够用：`npm pack` 打出来的 tarball 就是
`cordis.patch.yml` / `lib/client.js` / `lib/index.js` / `package.json`（外加 npm 自动收进来的本 README）。

## 已知未验（如实）

1. **路线 A 的②未实跑**。已实测的是同一机制下的 `dsh-mobile#path:/packages/bridge` 形态
   （`10-交接文档.md:2601-2607`）；本仓把同一个包放到**仓库根**，命令随之去掉 `#path:`，
   这一步的等价性**没有真机读数**。
2. **桥的 `lib/` 是不是「最新」**：桥**没有 `src/` 目录** —— `lib/*.js` 就是手写的源，
   `dsh-mobile` 仓里 `packages/bridge/` 从建仓至今只出现过这 4 个文件，
   `scripts/build-lib.mjs` 也完全不产出它（全文没有 `bridge`）。所以
   「lib 比 src 旧」这个检查对本包不适用。能给出的最强旁证：本仓 `lib/` 与
   `dsh-mobile:dist/npm/dsh-mobile-preview-bridge/lib/` 逐字节相同（那份是给 npm 发布准备的快照）。
3. 本包**没有 `peerDependencies`**，因此 DSH 的插件版本兼容性检查对它**没有可判的版本范围**
   （宿主管这件事，见 `dsh-mobile:packages/host/package.json:23-27`）。
   会不会被静默置 `disabled` —— **未验**。
4. 文中标注「本地文档」的出处（`10-交接文档.md` / `24-…md`）在**未发布的本地文档仓**里，
   公网上打不开；`dsh-mobile:` 前缀的那些则是 `github.com/mackereldace/dsh-mobile` 里可查的路径。
