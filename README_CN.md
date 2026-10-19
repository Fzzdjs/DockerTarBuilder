# 使用本项目的方法步骤

## 0、仓库设置要求（重要）

工作流依赖以下两项仓库设置，缺一项就会失败：

1. **开启 Actions**：`Settings → Actions → General → Actions permissions` 选择 `Allow all actions and reusable workflows`。
   > fork 过来的仓库默认可能关闭 Actions，此时在工作流页面看不到任何可运行的工作流。

2. **给予写入权限**：同一页面下方的 `Workflow permissions` 选择 **`Read and write permissions`**，并保存。
   > Release 工作流需要用它创建 / 更新 Release。若保持 `Read only`，会报
   > `403 Resource not accessible by integration`。
   > 工作流文件里已经声明了 `permissions: contents: write`，但该声明无法突破仓库级的只读限制。

另外，`workflow_dispatch` 触发的工作流必须存在于**默认分支（master）**上，才能在页面上看到 "Run workflow" 按钮。

## 1、fork 本项目到自己的空间
![1](https://github.com/user-attachments/assets/8bfc9d45-76fb-4cf4-be80-19fb202e8a95)
![2](https://github.com/user-attachments/assets/cf990b0c-a149-4f5b-9be8-d0f909f8f402)
![3](https://github.com/user-attachments/assets/d9560061-4092-4ea2-9b8c-a580c9fdcf2d)

## 2、输入要构建的docker名称和标签

> 如果不输入标签 则默认拉取latest标签，但请确保该镜像有latest <br>
> 如图左侧选项代表要构建的平台。支持 amd64、arm64、arm32 三种。<br>
> 多个镜像用**英文逗号**或**换行**分隔均可，前后多余空格会自动忽略。<br>
> 若构建的docker镜像小于2GB 推荐发布到release，选择红色框的部分。绝大多数镜像都是小于2GB的 <br>
> 若构建的docker镜像大于2GB 小于5GB 推荐使用另外三个工作流 <br>
> 若要构建大于5GB的镜像。本项目不适合。 <br>
> Release 工作流会在打包结束后检查文件是否超过 2 GiB，超过时直接给出明确报错，避免上传阶段才失败。

![4](https://github.com/user-attachments/assets/1a28afc2-2af2-47ff-91fa-0d42282a260a)
![5](https://github.com/user-attachments/assets/5fc5e388-ffb7-4242-aa0e-076136e1e974)
![6](https://github.com/user-attachments/assets/06bf9e37-cfbe-48cc-9012-1c7c92783d6c)
![7](https://github.com/user-attachments/assets/2fb1babf-1179-45fe-a686-2946939e1ac9)

## 3、加载离线docker镜像

```bash
docker load -i xxx.tar.gz
```

或

```bash
docker load < xxx.tar.gz
```

> 如果是 Artifacts 压缩文件 则需要先解压。解压后还是 `tar.gz` 无需再解压 直接 load 即可。
> 下载下来的 `xxx.tar.gz` 本身就是 gzip 压缩的镜像包，`docker load -i` 会自动识别，不需要再解压。

## 4、维护说明

仓库已经过可用性修复，当前依赖如下（均为 Node 24 运行时）：

| 依赖 | 版本 | 说明 |
| --- | --- | --- |
| 运行器 | `ubuntu-24.04` | 可用仓库变量切到自建/云主机（见下） |
| `actions/checkout` | `v7` | `v4` 及更早为 Node 20，已被移除 |
| `actions/upload-artifact` | `v7` | `v4`/`v5` 为 Node 20，已被移除 |
| `softprops/action-gh-release` | `v3.0.3` | `v2` 为 Node 20，已被移除 |

> **为什么之前完全跑不起来**：GitHub 于 2026-09-23 从所有托管运行器中删除了 Node.js 20 运行时。
> 任何 `action.yml` 中声明 `runs.using: node20` 的 action 都会在启动阶段直接失败，
> 因此上面三个依赖的旧版本会让全部 6 个工作流都无法执行。
>
> 仓库已配置 `.github/dependabot.yml`，每周自动检查 action 版本更新并提交 PR，
> 以便在 GitHub 再次淘汰旧运行时时及时跟进。

### 切到自建中转机（GitHub 托管机不可用时）

6 个工作流的运行器标签都取自仓库变量，默认值是 `ubuntu-24.04`：

```yaml
runs-on: ${{ fromJSON(vars.RUNNER_LABELS || '["ubuntu-24.04"]') }}
```

在 `Settings → Secrets and variables → Actions → Variables` 新增
`RUNNER_LABELS`，值填 `["self-hosted","linux","x64"]` 即可切到自建机；
删除该变量就回到 GitHub 托管运行器。

自建机（如阿里云国际轻量应用服务器）的选型、部署、安全与运维方案见
[docs/alibaba-cloud-swas-fallback.md](docs/alibaba-cloud-swas-fallback.md)。
