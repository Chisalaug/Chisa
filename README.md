# DSH Codex 启动动画移植指南（纯技术文档）

> ⚠️ **免责声明**
> 本项目**不包含**任何动画、图片、视频或第三方插件源码，仅提供纯文本技术原理与操作指引。
> 请自行从原作者仓库获取相关素材，并严格遵守原作者的版权与许可协议。
> 本项目为非官方第三方教程，与 DSH 官方及原素材作者无任何关联。

## 0. 原理简介
DSH 的插件机制要求每个 profile（`~/.dsh/profiles/<名字>/`）是一个独立的 npm 工程。
本教程介绍如何通过编写服务端注入脚本和浏览器端播放器，将原版的启动动画变成 DSH 的网页启动画面。
另外附带 Web UI 生态与 GPT 娘挂件的安装思路。

## 1. 前置准备与素材获取
由于版权原因，你需要自行获取以下素材：
1. **启动动画视频**：请前往 [yushua0808-cpu/dragon-codex-boot](https://github.com/yushua0808-cpu/dragon-codex-boot) 仓库，从其 `media/` 目录或 Releases 中获取 `startup.mp4`。
2. **DSH Web UI 生态**：文档基于 `@linxin666/dsh-web-all@0.4.8`。请使用 npm 安装，**不要**用 dev 分支 link。
3. **GPT 娘挂件**：参考 [xixingyingmie/dsh-gpt-niang-widget](https://github.com/xixingyingmie/dsh-gpt-niang-widget) 仓库。

## 2. 安装 Web UI 生态
请在你的 profile 目录（如 `~/.dsh/profiles/web`）下操作：

1. 配置镜像加速（防超时）：
   在 profile 目录创建 `.npmrc`，写入：
   ```text
   registry=https://registry.npmmirror.com
   fetch-retries=5
   fetch-timeout=120000
   network-concurrency=8
