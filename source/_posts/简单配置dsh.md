---
title: 简单配置dsh
date: 2026-08-26
tags:
  - dsh
categories:
  - [AI, Agent]
index_img: /img/categories/deepseek.ico
---

[Deepseek Harness](https://github.com/deepseek-ai/deepseek-harness) 测试版今日已经发布，以“一切皆插件”为主要特点，并且更有利于 Deepseek 模型发挥其能力。这里简单介绍如何安装并完成一些基础的配置工作。本文基于 Linux 环境。

<!-- more -->

# 安装 dsh 本体

可以直接使用 `npm` 进行安装。

dsh 对 `node` 的版本是有要求的，请先检查自己的 `node --version` 版本，要求在 `22.19` 及以上，建议直接使用 24 版本。安装命令如下。

```bash
$ npm install -g @deepseek-ai/dsh
```

安装成功之后我们可以运行 `dsh web` 运行后端，默认会将 dsh 挂到 `localhost:3080` ，我 们用浏览器访问即可。第一次打开会有弹窗要求配置 api key，将自己的 api key 粘贴进去 即可。


## npm 允许运行安装脚本

由于 dsh 的部分依赖包需要运行安装脚本，当使用较新的 `npm` 时这可能会被默认阻止，我 们需要在配置允许运行这几个包的安装脚本。

当出现如下输出时说明这几个包的运行脚本被阻止了。

```bash
npm warn deprecated node-domexception@1.0.0: Use your platform's native DOMException instead

added 454 packages in 55s

66 packages are looking for funding
  run `npm fund` for details
npm warn allow-scripts 5 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   @deepseek-ai/dsh-subprocess-local@0.1.1-rc.2 (postinstall: node scripts/ensure-spawn-helper.mjs)
npm warn allow-scripts   koffi@3.1.6 (install: node ./cnoke.cjs -P . -D src/koffi --prebuild --release)
npm warn allow-scripts   node-pty@1.2.0-beta.15 (install: node scripts/prebuild.js || node-gyp rebuild; postinstall: node scripts/post-install.js)
npm warn allow-scripts   @google/genai@1.52.0 (preinstall: echo 'preinstall: no-op')
npm warn allow-scripts   protobufjs@7.6.5 (postinstall: node scripts/postinstall)
npm warn allow-scripts
npm warn allow-scripts Run `npm install -g --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs` to allow these scripts once, or `npm config set allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs --location=user` to allow them for all global installs.
```

我们需要运行后一个，前一个目前在 `npm@11.19.0` 上会出现解析问题。

```bash
npm config set allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs --location=user
```

这将修改 `~/.npmrc` 文件，设置 `allow-scripts` 的值，后续我们再重新运行安装命令就可 以正常安装了。


# 安装 dsh-web

原生的 dsh 还比较简陋，只有最基础的功能，想要获得好一些的体验还需要安装一些插件来 获取额外的功能，比如文件树，git 状态等。这里推荐 [dsh-web](https://github.com/zhu1090093659/dsh-web/) ，这是一个插件组合包， 由多个插件组合实现，能够提供一个相对完整的开发体验。

这里需要先安装 `pnpm` ，直接通过 `npm` 安装即可。

```bash
npm install -g pnpm
```

之后运行下面命令安装插件

```bash
dsh plugin --profile web add @linxin666/dsh-web-ui-all@latest
```

安装成功后，重启 dsh 即可生效。


## pnpm 允许运行安装脚本

这里和使用 `npm` 一样也可能会出现安装脚本被阻止的问题，输出类似下面

```bash
✓ Lockfile passes supply-chain policies (verified 26s ago)
[WARN] Issues with peer dependencies found. Run "pnpm peers check" to list them.
Packages: -12
------------
Progress: resolved 214, reused 4, downloaded 0, added 0, done
[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: cloudflared@0.7.3, cpu-features@0.0.10, node-pty@1.1.0, ssh2@1.17.0

Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.
dsh: pnpm failed in profile directory /home/van/.dsh/profiles/web
```

我们直接运行 `pnpm approve-builds` 是没用的，只会提示当前没有包待批准。我们直接到 `~/.dsh/profiles/web` 这个文件夹下，这里存放着 `web` 这套配置的配置文件和 node 包。 我们修改这个文件夹下 `pnpm-workspace.yaml` 这个文件，该文件即 `pnpm` 的配置文件。

这个文件大致如下所示，我们在 `allowBuilds` 项中添加下面这四个包，之后再次运行安装 命令，即可成功安装。

```yaml
packages:
  - .

nodeLinker: hoisted
autoInstallPeers: false
allowBuilds:
  cloudflared: true
  cpu-features: true
  node-pty: true
  ssh2: true
```


# 安装插件商店

为了方便安装更多插件并管理，推荐安装 [dsh-market](https://github.com/dsh-market/dsh-market) 这个插件，这个插件可以方便的浏览 和安装大量插件，并且对插件进行分类，可以按照 GitHub stars 和 npm 下载量来排序。

安装命令如下

```bash
dsh plugin --profile web add dshmarket
```
