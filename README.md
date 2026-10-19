[![GitHub](https://img.shields.io/github/license/wukongdaily/DockerTarBuilder.svg?label=LICENSE&logo=github&logoColor=%20)](https://github.com/wukongdaily/DockerTarBuilder/blob/master/LICENSE)
![GitHub Stars](https://img.shields.io/github/stars/wukongdaily/DockerTarBuilder.svg?style=flat&logo=appveyor&label=Stars&logo=github)
![GitHub Forks](https://img.shields.io/github/forks/wukongdaily/DockerTarBuilder.svg?style=flat&logo=appveyor&label=Forks&logo=github) [![Github](https://img.shields.io/badge/RELEASE:DockerTarBuilder-123456?logo=github&logoColor=fff&labelColor=green&style=flat)](https://github.com/wukongdaily/DockerTarBuilder/releases) [![Bilibili](https://img.shields.io/badge/Bilibili-123456?logo=bilibili&logoColor=fff&labelColor=fb7299)](https://www.bilibili.com/video/BV1yyq6YREdF) 

## 🤔 这是什么？
[操作步骤](README_CN.md) | 简介 <br>
它是一个工作流。可快速构建指定架构/平台的单个或多个docker镜像 并存储在Artifact文件或Release文件

## 使用说明
图文步骤见 [README_CN.md](README_CN.md)。
> 如果你的docker镜像小于2GB 你可以使用红色框框这三种Release工作流<br> 
> 如果你的docker镜像大于2GB且小于5GB 你可以使用蓝色框框的这三种Artifact工作流<br>
> 如果你的docker镜像大于5GB 比如 `vllm/vllm-openai:v0.6.4.post1` 抱歉了 只能自己解决网络问题直接pull了 本项目爱莫能助啦！

### ⚠️ 使用前必须完成的两项仓库设置
> 1. `Settings → Actions → General → Actions permissions` 选择 **Allow all actions**（fork 仓库默认可能关闭 Actions）<br>
> 2. 同一页面的 `Workflow permissions` 选择 **Read and write permissions**（否则 Release 工作流会报 `403 Resource not accessible by integration`）

<img src="https://github.com/user-attachments/assets/e537c746-ec4f-4588-9315-ecf73a05b8d0" width="40%" />

## 2024-12-09 新增功能：
> 增加了新的工作流<br>
> 可以将docker镜像转存到自己项目的Github Release文件里<br>
## 2025-5月 新增下载提醒
对于Release文件，可以尝试直接使用“迅雷下载”，有一定概率加速。比如我这个项目里的Release 由于很多人下载过了，所以用迅雷下载很快。
如果迅雷速度是0 就把地址套用一下 下面的这些国内加速站的前缀 然后在复制到迅雷吧 这样下载大文件很快。

> [![Github](https://img.shields.io/badge/国内加速站下载-FC7C0D?logo=github&logoColor=fff&labelColor=000&style=for-the-badge)](https://wkdaily.cpolar.top/archives/1)

> [![Github](https://img.shields.io/badge/RELEASE:DockerTarBuilder-123456?logo=github&logoColor=fff&labelColor=green&style=for-the-badge)](https://github.com/wukongdaily/DockerTarBuilder/releases)<br>

> 如果Github Release文件下载缓慢 你可以在<br>         

> [![Github](https://img.shields.io/badge/国内加速站前缀套用点此处查看网址列表-FC7C0D?logo=github&logoColor=fff&labelColor=000&style=for-the-badge)](https://wkdaily.cpolar.top/archives/1)
 
## 教学视频
https://www.bilibili.com/video/BV1EZ421M7mL<br>
https://www.bilibili.com/video/BV1yyq6YREdF
## 新手常见问题必看
https://github.com/wukongdaily/DockerTarBuilder/wiki
## 解压工具
> Windows 上推荐使用7zip<br>
> macOS 推荐使用MacZip<br>
> Linux上推荐直接用tar 命令

## ❤️其它GitHub Action项目推荐🌟 （建议收藏）⬇️
- ### [一键生成run插件] 🆕
- https://github.com/wukongdaily/RunFilesBuilder<br>
- ### [一键构建自定义软件包大小的Immortalwrt] 🆕
- https://github.com/wukongdaily/AutoBuildImmortalWrt
- ### [OpenWrt/Armbian 安装器ISO] 🆕
- https://github.com/wukongdaily/armbian-installer
## 在哪里可以搜索或查询docker镜像的详细信息
### [查询镜像的详细信息 点击这里直达](https://docker.fxxk.dedyn.io/)

## ❤️赞助作者 ⬇️⬇️

[![点击这里赞助我](https://img.shields.io/badge/点击这里赞助我-支持作者的项目-orange?logo=github)](https://wkdaily.cpolar.top/01)

---

## 维护说明 / Maintenance

本仓库已做可用性修复，当前依赖（均为 Node 24 运行时）：

| 依赖 | 版本 | 说明 |
| --- | --- | --- |
| 运行器 | `ubuntu-24.04` | 可用仓库变量切到自建/云主机（见下节） |
| `actions/checkout` | `v7` | `v4` 及更早为 Node 20，运行时已被 GitHub 移除 |
| `actions/upload-artifact` | `v7` | `v4` / `v5` 默认仍为 Node 20 |
| `softprops/action-gh-release` | `v3.0.3` | `v2` 为 Node 20 |

**此前全部工作流无法运行的原因**：GitHub 于 2026-09-23 从所有托管运行器中删除了 Node.js 20 运行时。任何在 `action.yml` 中声明 `runs.using: node20` 的 action 都会在启动阶段直接失败，因此上述三个依赖的旧版本会让 6 个工作流全部失效。

仓库已加入 `.github/dependabot.yml`，每周自动检查 action 版本更新，避免同类问题再次发生。

### 切换到自建运行器（GitHub 托管机不可用时）

6 个工作流的 `runs-on` 都写成：

```yaml
runs-on: ${{ fromJSON(vars.RUNNER_LABELS || '["ubuntu-24.04"]') }}
```

因此**不用改代码**，只要在 `Settings → Secrets and variables → Actions → Variables`
新增变量 `RUNNER_LABELS`，值填 JSON 数组即可：

```json
["self-hosted","linux","x64"]
```

删掉该变量就自动回到 GitHub 托管运行器。自建中转机的完整采购、部署、安全与运维方案见
[docs/alibaba-cloud-swas-fallback.md](docs/alibaba-cloud-swas-fallback.md)。

> 自建机需要预先装好 Docker、git、curl，磁盘可用空间 ≥ 镜像体积的 2.5 倍，
> 并保证能访问目标 registry。
