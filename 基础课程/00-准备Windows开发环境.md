# 准备 Windows 开发环境

整理日期：2026-10-24。先检查已有工具，缺什么再补什么。此页是操作说明，没有替你安装或更改系统。

## 第一步：看你的电脑是什么类型

Windows“设置 → 系统 → 系统信息”中查看“系统类型”。普通 Intel/AMD 64 位电脑选择 x64/amd64 安装包；ARM 电脑选择对应 ARM64，不靠猜。

本项目本机验证环境是 Windows amd64、Go 1.26.9。Node 24 是当前项目 CI 使用的版本线；Python 至少 3.12，当前 CI 使用 3.14。新版本不应为了“越新越好”而未经验证一次性全部升级。

## 第二步：检查，不要重复安装

打开 PowerShell，分别输入 `go version`、`node --version`、`npm --version`、`python --version`、`git --version`。能显示符合要求的版本，就先保留。窗口需要在安装后重新打开，才能读到更新的 PATH。

只运行已经构建好的 MCP、timer、settings-center 三个 exe 时，虚拟机不需要这整套环境。环境主要在编译、测试和运行原生前端/embedding 时需要。

## 第三步：通过官方图形安装器补齐

| 工具 | 在官方页面选择什么 | 安装后怎么确认 |
| --- | --- | --- |
| Go | Windows 对应架构的 MSI，选符合 go.work 的版本；已有私有 Go 也可直接用完整路径 | 重新开终端，`go version` |
| Node.js | Windows、x64 或实际架构、24 LTS 版本线的安装器；npm 随常规 Node 安装提供 | `node --version` 与 `npm --version` |
| Python | 符合项目要求的稳定版 Windows 安装器，例如 3.14；不要选 embeddable 精简包当通用开发环境 | `python --version`；找不到时确认启动器与 PATH |
| Git | Git for Windows 官方下载入口，按安装向导安装；现有可用 Git 可保留 | `git --version` |

Go 官方 Windows 流程是运行 MSI，按向导安装，再重新开命令窗口验证。[Go 安装说明](https://go.dev/doc/install)

Node 页面可选择平台、架构和版本；此处跟随项目的 24 LTS 版本线，不把网页默认的 Current 当作本项目已验证选择。[Node 官方下载](https://nodejs.org/en/download)

Python 官方 Windows 页同时有安装器、安装管理器和嵌入式包。先选择普通开发用途的稳定版安装方式；不同安装器界面可能不同，按页面说明启用命令入口。[Python Windows 下载](https://www.python.org/downloads/windows/)

Git 的 Windows 下载入口由官方站点提供。安装配置不清楚时保留默认值，不需要为了本项目改全局证书校验或代理。[Git Windows 下载](https://git-scm.com/downloads/win)

## 第四步：先装项目依赖，再谈业务

Go 运行构建或测试时按 go.mod 下载库。前端在 frontend 文件夹运行 `npm ci`。Python 的 embedding 使用项目自己的虚拟环境；原启动脚本会准备它。需要网络下载依赖，下载失败不能被当成测试成功或源码逻辑失败。

不要把 API 密钥当作安装环境的一部分。模型密钥由你自己的供应商提供，填入项目已有秘密设置项；不要发送到聊天、提交到 Git 或写入公开示例。

## 第五步：回到一次具体复现

环境确认后读 [第一次复现](02-第一次复现.md)。先做一条可核对的链，不必一次启动所有入口、连 QQ、放真实模型、搭公网。

安装器确实会在你主动运行时安装软件并配置命令入口。本知识库只提供步骤，不会自动执行它们，也不要求永久放宽 PowerShell 执行策略。
