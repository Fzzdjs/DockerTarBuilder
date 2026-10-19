# 阿里云国际轻量应用服务器 中转下载方案（GitHub Actions 不可用时的替代链路）

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v1.0 |
| 编制日期 | 2026-10-19 |
| 适用对象 | 国内网络环境下需要稳定获取 docker.io / ghcr.io / registry.k8s.io / quay.io 镜像的团队 |
| 方案性质 | 配置方案（本文只做设计与步骤说明，不含实际开通与变更操作） |
| 关联项目 | `Fzzdjs/DockerTarBuilder`、技能 `docker-image-fetch` |

---

## 1. 背景与目标

本章说明现有链路的依赖点，以及为什么需要一条不依赖 GitHub 的备用通道。


### 1.1 现有链路

当前镜像获取依靠 GitHub Actions 中转，链路为：

```
本地脚本 → api.github.com 触发 workflow_dispatch
        → GitHub 海外托管运行器 docker pull + docker save | gzip
        → 产物上传 Artifact / Release
        → 本地多线程分片下载（Azure Blob / Fastly）
        → docker load
```

这条链路成本为零、维护量小，但存在四个不可控依赖：

1. **账号与政策**：GitHub 账号被限制、组织策略收紧、Actions 被禁用，链路立即中断。
2. **运行器可用性**：托管运行器镜像（如 `ubuntu-22.04`）会按 GitHub 的时间表弃用、停用；`ubuntu-24.04` 同样有生命周期。
3. **网络可达性**：`api.github.com` 与产物域名在国内的可达性会波动，单连接速度长期只有 30–80 KB/s。
4. **配额**：私有仓库 Actions 分钟数、Artifact 存储与保留期均受套餐限制。

### 1.2 本方案目标

在阿里云国际站轻量应用服务器（Simple Application Server，下称 SWAS）上自建一台**镜像中转机**，实现：

- 不依赖 GitHub 即可拉取公网镜像；
- 保留 `docker pull` 的原生使用习惯（Docker Hub 走透明缓存）；
- 仍能导出 `tar.gz` 供离线/内网机器 `docker load`；
- 月成本控制在人民币 40 元量级；
- 单台机器故障时可在 30 分钟内重建（快照 + 初始化脚本）。

### 1.3 非目标

- 不做高可用集群、不做多地域容灾；
- 不承载业务流量，不对外提供公共服务；
- 不解决私有 registry 的凭据托管（需另行配置）。

---

## 2. 方案选型

本章比较四条可选路线，并给出单机双通道的整体架构。


### 2.1 四条路线对比

| 方案 | 月成本（参考） | 国内下载速度 | 维护量 | 适用判断 |
| --- | --- | --- | --- | --- |
| GitHub Actions（现状） | 0 | 0.03–0.08 MB/s（单连接） | 低 | 继续作为首选，作为常态链路 |
| **阿里云 SWAS 自建中转（本方案）** | **¥34 起** | **1–3 MB/s** | 中 | **推荐作为兜底与提速链路** |
| 阿里云容器镜像服务 ACR 企业版 | ¥数百起 | 快 | 低 | 多团队、需要托管与审计时选 |
| 公网免费镜像加速器 | 0 | 不稳定、随时失效 | 无 | 只做临时应急，不可依赖 |

### 2.2 为什么不直接用 ECS

SWAS 已包含公网带宽、流量包、系统盘与快照，按套餐计费，不需要单独购买 EIP 与带宽；对"一台中转机"这种单机场景，总价比同规格 ECS + 按流量计费更低，控制台也更简单。当日志量、并发量增长到需要弹性伸缩或多盘时，再迁移到 ECS。

### 2.3 单机双通道架构

一台 SWAS 上同时提供两种能力，互为补充：

```
                        ┌──────────────── 阿里云 SWAS（新加坡） ────────────────┐
                        │                                                      │
  docker pull ──443──▶  │  nginx (TLS 终止 + Basic Auth + 限流)                 │
  (Docker Hub 透明缓存) │    ├─ /v2/*  → registry-hub:5000   (docker.io 代理)  │
                        │    ├─ /v2/*  → registry-ghcr:5001  (ghcr.io 代理)    │
                        │    └─ /v2/*  → registry-k8s:5002   (k8s.io 代理)     │
                        │                                                      │
  脚本/离线导出 ──443─▶  │  /api/*  → pull-api:8080  (拉取-打包-下载)            │
                        │  /files/* → 静态 tar.gz（nginx 原生支持 Range 分片）  │
                        │                                                      │
                        │  /data/registry  缓存目录（可整目录清空重建）          │
                        │  /data/images    导出目录（按保留期清理）              │
                        └──────────────────────────────────────────────────────┘
```

- **通道 A：透明缓存（Pull-through Cache）**。适合 `docker pull`、`docker compose`、CI 构建等日常场景，客户端无需改镜像名，只改 `daemon.json`。
- **通道 B：拉取-导出服务**。适合离线交付、内网机器、需要 `tar.gz` 的场景，接口与现有技能的输出格式一致。

> `registry:2` 的一个实例只能代理**一个**上游 registry，因此按上游拆成多个容器，用 nginx 的 `server_name` 或路径前缀区分。

---

## 3. 服务器选型

本章给出地域、规格、流量与磁盘的量化依据，作为采购与升配的判断标准。


### 3.1 地域选择

| 地域 | 国内平均延迟 | 超额流量单价 | 结论 |
| --- | --- | --- | --- |
| **新加坡** | 60–90 ms | $0.081/GB | **首选**，价格与线路均衡 |
| 马来西亚（吉隆坡） | 70–100 ms | $0.077/GB | 备选，价格略低 |
| 日本（东京） | 50–80 ms | $0.087/GB | 延迟更低，单价略高 |
| 中国香港 | 30–50 ms | $0.153/GB | 延迟最低，流量最贵，适合小镜像高频 |
| 美国（硅谷/弗吉尼亚） | 150–200 ms | $0.076/GB | 不推荐，跨境链路长 |

> 境外地域**不需要 ICP 备案**。若企业有数据出境合规要求，本方案只传输公开镜像，不涉及业务数据。

### 3.2 套餐规格

参考在售套餐（中国站境外地域活动价，单位为人民币；国际站约为 $8/月起，具体以购买页为准）：

| 套餐 | 系统盘 | 月流量 | 峰值带宽 | 参考月价 | 适用 |
| --- | --- | --- | --- | --- | --- |
| 2 核 1 GB | 40 GB ESSD | 1 TB | 30 Mbps | ¥24 | 仅做 Docker Hub 缓存 |
| **2 核 2 GB** | **60 GB ESSD** | **2 TB** | **30 Mbps** | **¥34** | **推荐起步**，双通道 + 20–40 个镜像 |
| 2 核 4 GB | 80 GB ESSD | 3 TB | 30 Mbps | ¥67 | 并发客户端多、缓存量大 |
| 2 核 8 GB | 100 GB ESSD | 4 TB | 30 Mbps | ¥133 | 团队级、需同时跑 Harbor |

> 2026-04-20 起部分旧规格已停售，上表为当前可购规格的典型档位。轻量套餐支持后续升配，建议**先买 2C2G，观察两周再决定是否升配**。

### 3.3 关键容量测算

**带宽**：30 Mbps 峰值的理论下载速度为 3.75 MB/s。跨境实际受线路与国际出口影响，通常 **1–3 MB/s**；仍比 GitHub 产物单连接快 20–50 倍。

**流量**：套餐内为**出网流量**（服务器 → 客户端），入网不计费。

| 使用强度 | 月下载量 | 是否超出 2 TB |
| --- | --- | --- |
| 30 个 1 GB 镜像 | 30 GB | 否，余量充足 |
| 200 个 1 GB 镜像 | 200 GB | 否 |
| 1000 个 2 GB 镜像 | 2000 GB | 临界 |
| 持续给多团队分发 | > 2 TB | 是，超额 $0.081/GB |

结论：**流量不是瓶颈，带宽与磁盘才是**。峰值 30 Mbps 下，1 GB 镜像的理想传输时间为 4.5 分钟，实际 6–15 分钟。

**磁盘**：系统盘 60 GB，扣除系统与 Docker 自身约 12–15 GB，可用于镜像缓存约 45 GB。

| 单镜像压缩后大小 | 可缓存数量（45 GB 可用） |
| --- | --- |
| 200 MB | ~200 个 |
| 1 GB | ~40 个 |
| 4 GB | ~10 个 |

必须在缓存目录设置水位清理，否则磁盘写满会导致 `registry` 直接 500。

### 3.4 购买时的选项清单

| 选项 | 建议值 | 说明 |
| --- | --- | --- |
| 地域 | 新加坡 | 见 3.1 |
| 镜像 | Ubuntu 24.04 LTS | 生态最全，避免 Alibaba Cloud Linux 的 docker 源差异 |
| 套餐 | 2 核 2 GB / 60 GB / 2 TB | 起步档 |
| 登录方式 | **SSH 密钥** | 不设置密码登录 |
| 数据盘 | 暂不购买 | 先用系统盘，缓存可随时清空重建 |
| 自动快照 | 每周 1 次，保留 2 份 | 重建成本最低的保险 |
| 安全组 | 仅放行 22、80、443 | 5000/5001/5002/8080 **只监听 127.0.0.1** |
| 到期策略 | 自动续费 | 中转机停机会影响构建 |

---

## 4. 实施步骤

本章按系统初始化到客户端接入的顺序给出可执行步骤，并估算实施工时。


> 以下命令均为**方案说明**，用于评估工作量与操作顺序，不在本文档中执行。命令以 Ubuntu 24.04 为准。

### 4.1 系统初始化

```bash
# 1) 更新系统、设置时区
sudo apt update && sudo apt -y upgrade
sudo timedatectl set-timezone Asia/Shanghai

# 2) 创建运维账号（禁用 root 密码登录）
sudo adduser --disabled-password --gecos "" ops
sudo usermod -aG sudo ops
sudo mkdir -p /home/ops/.ssh && sudo chmod 700 /home/ops/.ssh
# 将本地公钥写入 /home/ops/.ssh/authorized_keys 后：
sudo chown -R ops:ops /home/ops/.ssh && sudo chmod 600 /home/ops/.ssh/authorized_keys

# 3) 加固 SSH：禁用密码与 root 直登
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh

# 4) 防火墙（云侧安全组 + 主机侧 ufw 双层）
sudo apt -y install ufw fail2ban
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable

# 5) 自动安全更新
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 4.2 安装 Docker

```bash
sudo apt -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### 4.3 目录与权限

```bash
sudo mkdir -p /data/registry/{hub,ghcr,k8s,quay} /data/images /data/auth /data/nginx
sudo chown -R 1000:1000 /data/registry        # registry 容器内以 UID 1000 运行
sudo chmod 755 /data/images
```

### 4.4 部署四个 Pull-through Cache

以 Docker Hub 为例，`/data/registry/hub/config.yml`：

```yaml
version: 0.1
log:
  level: info
  fields:
    service: registry-hub
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: 0.0.0.0:5000
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://registry-1.docker.io
  username: ""          # 公共镜像留空；有 Docker Hub 账号可填以提升限流额度
  password: ""
```

其余三个只需替换 `remoteurl` 与端口：

| 容器 | 端口 | remoteurl |
| --- | --- | --- |
| registry-hub | 5000 | `https://registry-1.docker.io` |
| registry-ghcr | 5001 | `https://ghcr.io` |
| registry-k8s | 5002 | `https://registry.k8s.io` |
| registry-quay | 5003 | `https://quay.io` |

`/opt/mirror/docker-compose.yml`：

```yaml
services:
  registry-hub:
    image: registry:2
    restart: always
    volumes:
      - /data/registry/hub:/var/lib/registry
      - ./hub.yml:/etc/docker/registry/config.yml:ro
    ports: ["127.0.0.1:5000:5000"]

  registry-ghcr:
    image: registry:2
    restart: always
    volumes:
      - /data/registry/ghcr:/var/lib/registry
      - ./ghcr.yml:/etc/docker/registry/config.yml:ro
    ports: ["127.0.0.1:5001:5000"]

  registry-k8s:
    image: registry:2
    restart: always
    volumes:
      - /data/registry/k8s:/var/lib/registry
      - ./k8s.yml:/etc/docker/registry/config.yml:ro
    ports: ["127.0.0.1:5002:5000"]

  registry-quay:
    image: registry:2
    restart: always
    volumes:
      - /data/registry/quay:/var/lib/registry
      - ./quay.yml:/etc/docker/registry/config.yml:ro
    ports: ["127.0.0.1:5003:5000"]
```

> 端口只绑定 `127.0.0.1`，公网访问一律经 nginx，避免 5000 端口被扫描后匿名使用。

### 4.5 认证

```bash
sudo apt -y install apache2-utils
# 用户名 mirror，按提示输入强口令
sudo htpasswd -B -c /data/auth/htpasswd mirror
sudo chown root:root /data/auth/htpasswd && sudo chmod 640 /data/auth/htpasswd
```

### 4.6 nginx 反向代理 + TLS

`/etc/nginx/conf.d/mirror.conf`（`registry.example.com` 替换为自有域名）：

```nginx
# Docker Hub 透明缓存：客户端 daemon.json 的 registry-mirrors 指向此域名
server {
    listen 443 ssl http2;
    server_name hub.example.com;

    ssl_certificate     /etc/letsencrypt/live/hub.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hub.example.com/privkey.pem;

    client_max_body_size 0;
    chunked_transfer_encoding on;

    auth_basic           "mirror";
    auth_basic_user_file /data/auth/htpasswd;

    location /v2/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 900;
        proxy_request_buffering off;
    }
}

# ghcr.io 缓存：客户端显式 pull ghcr.example.com/<owner>/<image>:<tag>
server {
    listen 443 ssl http2;
    server_name ghcr.example.com;
    ssl_certificate     /etc/letsencrypt/live/ghcr.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ghcr.example.com/privkey.pem;
    client_max_body_size 0;
    auth_basic           "mirror";
    auth_basic_user_file /data/auth/htpasswd;
    location /v2/ {
        proxy_pass http://127.0.0.1:5001;
        proxy_set_header Host $host;
        proxy_read_timeout 900;
        proxy_request_buffering off;
    }
}
# k8s / quay 同理，分别指向 5002、5003
```

证书申请（域名需先解析到服务器公网 IP）：

```bash
sudo apt -y install certbot python3-certbot-nginx
sudo certbot --nginx -d hub.example.com -d ghcr.example.com -d k8s.example.com -d quay.example.com
sudo systemctl enable --now certbot.timer
```

> 没有域名时：可用自签证书 + 客户端 `insecure-registries`，但**不建议**放在公网，容易遭遇中间人替换镜像。

### 4.7 客户端接入（通道 A）

`/etc/docker/daemon.json`（Windows 为 Docker Desktop → Settings → Docker Engine）：

```json
{
  "registry-mirrors": ["https://hub.example.com"],
  "insecure-registries": []
}
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
docker pull nginx:alpine          # 自动走 hub.example.com，镜像名不变
docker pull ghcr.example.com/tencentcloud/octop:latest   # 非 Docker Hub 显式改前缀
```

> `registry-mirrors` **只对 Docker Hub 生效**，这是 Docker 客户端的行为，不是本方案的缺陷。ghcr.io / k8s.io / quay.io 必须显式改镜像前缀，拉下来后可 `docker tag` 回原名。

### 4.8 通道 B：拉取-导出服务

用途：给离线机器、内网机器、或需要 `tar.gz` 的场景。接口约定：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/jobs` | body：`{"image":"ghcr.io/tencentcloud/octop:latest","platform":"linux/amd64"}`，返回 `job_id` |
| GET | `/api/jobs/{id}` | 返回 `status`（queued/running/success/failed）、`progress`、`file`、`size` |
| GET | `/files/{name}` | 下载 tar.gz，nginx 原生支持 Range，可 16 路分片 |
| DELETE | `/api/jobs/{id}` | 删除任务与文件 |

参考实现要点（约 100 行，FastAPI + 后台线程）：

```python
# /opt/mirror/api.py  —— 仅示意，部署前需补齐鉴权、并发上限与磁盘水位判断
import os, re, subprocess, threading, uuid
from fastapi import FastAPI, HTTPException, Header

ROOT = "/data/images"
TOKEN = os.environ["MIRROR_TOKEN"]          # 由 systemd EnvironmentFile 注入
jobs = {}
lock = threading.Lock()
app = FastAPI()

def safe(image: str, arch: str) -> str:
    return re.sub(r"[/:@]", "_", image) + "-" + arch + ".tar.gz"

def run(job_id: str, image: str, platform: str):
    arch = platform.split("/")[-1]
    name = safe(image, arch)
    dest = os.path.join(ROOT, name)
    try:
        jobs[job_id].update(status="running", progress=0.1)
        subprocess.run(["docker", "pull", "--platform", platform, image],
                       check=True, capture_output=True, timeout=3600)
        jobs[job_id].update(progress=0.6)
        with open(dest + ".tmp", "wb") as fh:
            p1 = subprocess.Popen(["docker", "save", image], stdout=subprocess.PIPE)
            p2 = subprocess.Popen(["gzip", "-c"], stdin=p1.stdout, stdout=fh)
            p1.stdout.close(); p2.communicate(timeout=1800)
        os.replace(dest + ".tmp", dest)
        jobs[job_id].update(status="success", progress=1.0,
                            file="/files/" + name, size=os.path.getsize(dest))
    except Exception as e:
        jobs[job_id].update(status="failed", error=str(e)[:300])

@app.post("/api/jobs")
def create(body: dict, authorization: str = Header("")):
    if authorization != "Bearer " + TOKEN:
        raise HTTPException(401)
    job_id = uuid.uuid4().hex[:12]
    jobs[job_id] = {"id": job_id, "status": "queued", "image": body["image"]}
    threading.Thread(target=run, args=(job_id, body["image"],
                                       body.get("platform", "linux/amd64")), daemon=True).start()
    return {"job_id": job_id}
```

配套 systemd 单元（`/etc/systemd/system/mirror-api.service`）：

```ini
[Unit]
Description=Mirror Pull API
After=docker.service

[Service]
EnvironmentFile=/etc/mirror/env
ExecStart=/opt/mirror/venv/bin/uvicorn api:app --host 127.0.0.1 --port 8080
Restart=always

[Install]
WantedBy=multi-user.target
```

`/files/` 由 nginx 直接托管（带 Basic Auth、禁止目录列表、限制并发）：

```nginx
location /files/ {
    alias /data/images/;
    autoindex off;
    auth_basic "mirror";
    auth_basic_user_file /data/auth/htpasswd;
    add_header Accept-Ranges bytes;
}
```

客户端下载（与现有技能的 16 路分片思路一致）：

```bash
aria2c -x16 -s16 -k8M --http-user=mirror --http-passwd='***' \
  https://hub.example.com/files/ghcr.io_tencentcloud_octop_latest-amd64.tar.gz
```

### 4.9 实施顺序与工时

| 阶段 | 内容 | 预计耗时 |
| --- | --- | --- |
| 1 | 购买实例、配置安全组与快照策略 | 20 分钟 |
| 2 | 系统初始化、安装 Docker | 30 分钟 |
| 3 | 部署 4 个 registry 与 nginx/TLS | 60 分钟 |
| 4 | 客户端接入与三类镜像验证 | 40 分钟 |
| 5 | 部署通道 B（可选） | 60 分钟 |
| 6 | 清理策略、监控、文档归档 | 30 分钟 |
| 合计 | 单台从零到可用 | **约 4 小时**（含通道 B） |

---

## 5. 安全与合规

本章列出自建中转机必须落实的安全措施，以及被滥用后的实际后果。


### 5.1 必须做的六件事

1. **禁止匿名访问**：nginx `auth_basic` 全站开启；registry 端口只监听 `127.0.0.1`。
2. **强制 TLS**：Docker 对 `registry-mirrors` 要求 HTTPS；明文 registry 只在临时排障时开启。
3. **最小端口**：安全组只开 22/80/443；80 仅用于 ACME 校验与 301 跳转。
4. **SSH 加固**：仅密钥登录、禁 root、fail2ban 默认策略。
5. **磁盘水位**：`/data` 使用率 > 80% 触发清理，> 90% 告警，避免 registry 写入失败。
6. **镜像来源白名单**：通道 B 只允许 `docker.io`、`ghcr.io`、`registry.k8s.io`、`quay.io`、`mcr.microsoft.com`，防止被当作任意网络的开放代理。

### 5.2 被滥用的典型后果

公网开放的 registry 通常在数小时内被扫描到，攻击者会把中转机当作免费代理拉取大量镜像，直接产生：

- 超额流量费用（新加坡 $0.081/GB，跑满 30 Mbps 一天约 324 GB，约 **$26/天**）；
- 磁盘被撑满，registry 服务不可用；
- 出口 IP 被目标 registry 限流或封禁。

因此**认证与限流不是可选项**。建议再加 nginx `limit_req`（如 `10r/s`，burst 20）与单 IP 并发连接上限。

### 5.3 合规说明

- 境外地域无需 ICP 备案；
- 仅中转公开镜像，不落业务数据；
- 若企业有数据出境评估要求，应把本机定位为"公开软件制品缓存"，并在资产管理台账中登记。

---

## 6. 运维

本章覆盖缓存清理、日志监控、备份重建与验收标准。


### 6.1 缓存清理

`registry:2` 的 proxy 缓存可以整目录清空后重建，比 `garbage-collect` 更简单可靠：

```bash
# /usr/local/bin/mirror-gc.sh
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD=80
USE=$(df --output=pcent /data | tail -1 | tr -dc '0-9')
if [ "$USE" -ge "$THRESHOLD" ]; then
  systemctl stop docker
  rm -rf /data/registry/*/docker/registry/v2/repositories/*
  find /data/images -name '*.tar.gz' -mtime +3 -delete
  systemctl start docker
  logger -t mirror-gc "cleaned, was ${USE}%"
fi
```

```bash
sudo chmod +x /usr/local/bin/mirror-gc.sh
# 每天 03:30 检查
echo '30 3 * * * root /usr/local/bin/mirror-gc.sh' | sudo tee /etc/cron.d/mirror-gc
```

### 6.2 日志与监控

| 项目 | 做法 |
| --- | --- |
| 容器日志 | compose 里限制 `logging.options.max-size: 20m`、`max-file: 3` |
| nginx 日志 | logrotate 按天切割，保留 14 天 |
| 健康检查 | `curl -fsS -u mirror:*** https://hub.example.com/v2/` 返回 200 |
| 关键告警 | 磁盘 > 85%、5xx 比例 > 1%、出口流量 > 套餐 80%、容器重启 |
| 云监控 | 开启 SWAS 自带 CPU/流量/磁盘监控与告警通知 |

### 6.3 备份与重建

- **快照**：每周自动快照，保留 2 份；变更配置前手动打一次快照。
- **配置入库**：`/opt/mirror` 全部文件（compose、registry 配置、nginx、清理脚本、systemd 单元）纳入 Git 仓库，重建时 `git clone` 即可。
- **重建演练**：每季度用快照恢复一台临时实例，验证 30 分钟内可用。

### 6.4 验收标准

| 场景 | 目标 |
| --- | --- |
| 首次拉取 `nginx:alpine` | < 60 秒 |
| 二次拉取同一镜像（缓存命中） | < 10 秒 |
| 1 GB 镜像经通道 B 下载 | < 20 分钟（≥ 0.9 MB/s） |
| 3 个客户端并发拉取 | 无 5xx，带宽接近 30 Mbps 上限 |
| 磁盘水位触发清理 | 自动完成，服务不中断 |

---

## 7. 切换与回退预案

本章定义启动切换的触发条件、时间表与回退动作。


### 7.1 触发条件

出现以下任一情况，启动切换：

- GitHub Actions 连续 24 小时无法触发或全部失败；
- 托管运行器镜像被停用且无可用替代；
- 账号被限制、组织策略禁用 Actions；
- 产物下载速度持续低于 100 KB/s，导致交付 SLA 无法满足。

### 7.2 切换步骤

| 时间点 | 动作 | 验证 |
| --- | --- | --- |
| T-7 天 | 采购并部署 SWAS，通道 A/B 就绪 | 验收标准全通过 |
| T-3 天 | 在测试机修改 `daemon.json`，跑通构建 | `docker pull` 与 CI 构建成功 |
| T-1 天 | 导入 3 个高频镜像预热缓存 | 缓存命中 < 10 秒 |
| T-0 | 生产客户端切换；GitHub 链路保留 | 业务无感，失败率 0 |
| T+30 天 | 评估是否下线 GitHub 链路 | 成本与稳定性复盘 |

### 7.3 回退

| 场景 | 回退动作 | 耗时 |
| --- | --- | --- |
| 中转机故障 | 删除客户端 `registry-mirrors`，恢复 GitHub 链路 | 5 分钟 |
| 带宽不足 | 升级套餐（2C2G → 2C4G） | 10 分钟（需重启） |
| 缓存污染 | 清空 `/data/registry/*/repositories` 并重启 | 3 分钟 |
| 实例损坏 | 从快照恢复或按 Git 仓库重建 | 30 分钟 |

> 关键原则：**客户端切换只改一个 `daemon.json` 文件**，因此回退成本极低。不要在业务镜像里硬编码中转域名。

---

## 8. 成本估算

以新加坡 2 核 2 GB / 60 GB / 2 TB 套餐、月下载 200 GB 为例：

| 项目 | 金额（人民币/月） | 说明 |
| --- | --- | --- |
| 套餐费 | 34 | 活动价；国际站约 $8 |
| 超额流量 | 0 | 200 GB 远低于 2 TB |
| 快照 | 0 | 套餐内免费额度 |
| 域名（可选） | ~5 | 摊薄到月 |
| **合计** | **约 39** | 不含人力 |

对比 GitHub Actions：直接费用为 0，但下载 1 GB 镜像耗时 5–9 小时；自建链路按 1.5 MB/s 计算约 11 分钟。**按人力成本折算，自建链路在每月超过 5 个镜像时即已回本**。

---

## 9. 风险与限制

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| 跨境链路抖动 | 下载速度波动 | 多线程分片；必要时换日本/香港地域 |
| 单机故障 | 中转中断 | 快照 + Git 化配置，30 分钟重建；GitHub 链路保留 |
| 磁盘写满 | registry 5xx | 水位清理 + 告警 |
| 被扫描滥用 | 流量费用激增 | Basic Auth + 限流 + 非默认端口 + IP 白名单 |
| 上游限流（Docker Hub 匿名 100 次/6h） | 拉取失败 | 在 registry 配置中填 Docker Hub 账号 |
| 大镜像（> 20 GB） | 磁盘不足 | 改用 ECS + 数据盘，或仍走 Artifact 通道 |

---

## 10. 与现有工具链的衔接

本章说明现有脚本、工作流与离线交付方式如何平滑迁移。


| 现有能力 | 切换后的用法 |
| --- | --- |
| `docker pull` | 改 `daemon.json` 的 `registry-mirrors`，镜像名不变 |
| `docker-image-fetch` 技能 | 通道 B 的 `/api/jobs` + `/files/` 输出同样为 `tar.gz`，可复用其 16 路分片下载器 |
| `DockerTarBuilder` 工作流 | 保留；将仓库变量 `RUNNER_LABELS` 指向挂在本机的 self-hosted runner，即可用同一套工作流跑在阿里云上 |
| 离线交付 | 通道 B 导出 `tar.gz`，目标机 `docker load -i` |

### 10.1 接入 self-hosted runner（可选）

若希望继续用现有工作流，而不是自己写打包脚本：

```bash
# 在 SWAS 上以 ops 用户执行
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -fsSL -o runner.tar.gz https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf runner.tar.gz
./config.sh --url https://github.com/Fzzdjs/DockerTarBuilder --token <一次性注册令牌> \
            --labels self-hosted,linux,x64 --unattended
sudo ./svc.sh install ops && sudo ./svc.sh start
```

然后在仓库 `Settings → Secrets and variables → Actions → Variables` 增加：

```
RUNNER_LABELS = ["self-hosted","linux","x64"]
```

工作流会自动改用本机运行器；删除该变量即回到 GitHub 托管运行器。

> 注意：self-hosted runner 仍需要访问 `api.github.com` 领取任务。若 GitHub 整体不可达，请走通道 A/B，不要再依赖 Actions。

---

## 附录 A：故障排查

| 症状 | 可能原因 | 处理 |
| --- | --- | --- |
| `docker pull` 报 `x509: certificate signed by unknown authority` | 自签证书未加入信任 | 换 Let's Encrypt 证书，或把 CA 放入 `/usr/local/share/ca-certificates` 后 `update-ca-certificates` |
| 报 `401 Unauthorized` | Basic Auth 未带或口令错 | `docker login hub.example.com` 后重试 |
| 报 `502 Bad Gateway` | registry 容器未启动 | `docker compose ps` / `docker logs registry-hub` |
| 拉取卡在 `Waiting` | 上游不可达或限流 | 服务器上 `curl -I https://registry-1.docker.io/v2/` 验证 |
| 磁盘 100% | 缓存未清理 | 执行 `mirror-gc.sh`，检查 cron 是否生效 |
| 速度只有几十 KB/s | 走了 IPv6 或链路绕行 | 服务器与客户端分别 `mtr` 定位；必要时换地域 |
| `registry` 启动即退出 | 配置文件 YAML 缩进错误 | `docker logs` 查看具体行号 |

## 附录 B：配置项速查

| 配置 | 位置 | 建议值 |
| --- | --- | --- |
| 缓存根目录 | `/data/registry/<upstream>` | 独立子目录，便于单独清理 |
| 导出目录 | `/data/images` | 保留 3 天，定时删除 |
| registry 监听 | `127.0.0.1:5000-5003` | 不暴露公网 |
| API 监听 | `127.0.0.1:8080` | 经 nginx `/api/` 反代 |
| TLS | nginx + certbot | 自动续期 |
| 认证 | htpasswd bcrypt | 每季度更换口令 |
| 磁盘水位 | cron 每天 03:30 | 80% 清理、90% 告警 |
| 快照 | SWAS 控制台 | 每周 1 次，保留 2 份 |
| 日志 | docker json-file | 20 MB × 3 |

---

**结论**：在 GitHub Actions 可用时继续用它（零成本、零维护）；同时把本方案作为**已验收的备用链路**常备一台 2C2G 轻量服务器（月成本约 ¥34），即可在托管运行器停用、账号受限或下载速度不可接受时，30 分钟内完成切换。
