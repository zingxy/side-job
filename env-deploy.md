# 环境变量与部署说明

## 云服务器目录结构

CI/CD 部署后，每台目标服务器上的文件位于 `/opt/myapp/`：

```
/opt/myapp/
├── .env                 # 环境变量配置（首次部署自动生成，后续 CI/CD 不覆盖）
├── nginx.conf           # Nginx 反向代理配置（首次部署自动生成，后续 CI/CD 不覆盖）
├── docker-compose.yml   # 容器编排文件（每次部署由 CI/CD 覆盖）
└── certs/               # SSL 证书目录（需手动创建）
    ├── server.crt       # HTTPS 证书文件
    └── server.key       # HTTPS 私钥文件
```

- **`.env`**：包含数据库、OSS、微信支付、支付模式等运行时环境变量。首次部署时由仓库上传，之后可在服务器上手动修改。
- **`nginx.conf`**：Nginx 主配置，`server_name` 在部署时通过 `envsubst` 注入。首次部署后不覆盖。
- **`docker-compose.yml`**：每次部署时由 CI/CD 将 `.env` 中注入的 `APP_IMAGE` / `PAY_IMAGE` 变量替换后生成并上传覆盖。
- **`certs/`**：SSL 证书目录，通过 volume 只读挂载到 Nginx 容器的 `/etc/nginx/certs/`。需手动创建并放入证书文件，启用 HTTPS 时必填（见下方 HTTPS 配置一节）。

## 环境变量详解（`.env`）

### Git 仓库配置

| 变量 | 示例 | 说明 |
|---|---|---|
| `GIT_REPO_URL` | `git@github.com:apEeach/DS_Course.git` | C++ 后端代码的 Git 仓库地址（SSH 格式） |
| `PAY_GIT_REPO_URL` | `git@github.com:apEeach/DS_Course_Pay.git` | Go 支付服务代码的 Git 仓库地址（SSH 格式） |
| `GIT_SSH_KEY_PATH` | `/home/user/.ssh/id_rsa` | SSH 私钥在**宿主机**上的绝对路径。本地构建时，Docker 会通过 BuildKit SSH mount 将密钥传入容器，用于 clone 私有仓库 |

### 数据库配置

| 变量 | 示例 | 说明 |
|---|---|---|
| `ENABLE_MYSQL` | `true` / `false` | 是否启动内置 MySQL 容器。云服务器部署时设为 `false`（使用外部云数据库），本地开发时可设为 `true` |
| `ENABLE_INIT_DB` | `true` / `false` | 是否执行容器内 `init_db.sh` 初始化脚本。使用 Flyway 管理迁移时建议设为 `false` |
| `DB_HOST` | `db` | 数据库地址。使用内置 MySQL 时填 `db`（Docker 网络内的容器名）；使用外部数据库时填实际 IP 或域名 |
| `DB_USER` | `root` | 数据库用户名 |
| `DB_PASS` | `your_password` | 数据库密码 |
| `DB_NAME` | `mydb` | 数据库名称 |

### 阿里云 OSS 配置

| 变量 | 说明 |
|---|---|
| `OSS_ENDPOINT` | 阿里云 OSS Endpoint（如 `https://oss-cn-hangzhou.aliyuncs.com`） |
| `OSS_ACCESS_KEY_ID` | 阿里云 OSS AccessKey ID |
| `OSS_ACCESS_KEY_SECRET` | 阿里云 OSS AccessKey Secret |
| `OSS_BUCKET_NAME` | 阿里云 OSS Bucket 名称 |

### Nginx 域名配置

| 变量 | 示例 | 说明 |
|---|---|---|
| `SERVER_NAME` | `192.168.1.11` 或 `api.example.com` | Nginx `server_name` 指令值，部署时通过 `envsubst` 注入到 `nginx.conf` 中 |

### 支付服务配置

| 变量 | 示例 | 说明 |
|---|---|---|
| `PAY_MODE` | `mock` / `real` | 支付模式：`mock` 为模拟支付（本地开发），`real` 为真实微信支付（生产环境） |
| `MOCK_ORDER_STATE` | `NOTPAY` | Mock 查单返回状态，可选值：`NOTPAY` / `SUCCESS` / `CLOSED` / `REVOKED` |
| `WECHAT_APPID` | `wx1234567890` | 微信 AppID，`PAY_MODE=real` 时必填 |
| `WECHAT_MCH_ID` | `1234567890` | 微信商户号，`PAY_MODE=real` 时必填 |
| `WECHAT_SERIAL_NO` | `商户证书序列号` | 商户 API 证书序列号，`PAY_MODE=real` 时必填 |
| `WECHAT_API_V3_KEY` | `APIv3密钥` | 微信支付 APIv3 密钥，`PAY_MODE=real` 时必填 |
| `WECHAT_CERT_PATH` | `/app/certs` | 商户证书在容器内的路径，默认 `/app/certs` |
| `WECHAT_NOTIFY_URL` | `https://domain/pay/notify` | 微信支付回调通知地址，需公网可达 |

### 定时查单兜底配置

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ENABLE_ORDER_CHECK` | `true` | 是否开启定时查单兜底机制 |
| `ORDER_PAYMENT_TIMEOUT_MINUTES` | `30` | 订单支付超时时间（分钟），超过此时间后不再查询 |
| `ORDER_CHECK_INTERVAL_SECONDS` | `300` | 定时查单间隔（秒），默认 5 分钟 |

## 容器网络

### 网络架构

所有容器通过 Docker Compose 定义的自定义 Bridge 网络 `my_net` 互联：

```
                         ┌─────────────────────────── my_net (bridge) ───────────────────────────┐
                         │                                                                       │
┌──────────┐    :80      │   :8080                                                                │
│ 客户端   │ ◄─────────► │  ┌─────────┐                                                          │
│ (浏览器)  │            │  │ nginx   │                                                          │
└──────────┘             │  │ :80     │                                                          │
                         │  └────┬────┘                                                          │
                         │       │  反向代理 (所有请求统一转发)                                    │
                         │       │                                                                │
                         │  ┌────┴─────┐                 ┌──────────┐                            │
                         │  │   app    │  ◄── C++ 服务   │   pay    │  ◄── Go 支付服务 (9090)     │
                         │  │ (8080)   │  ◄──────────── │ (9090)   │                            │
                         │  └────┬─────┘   HTTP 调用     └──────────┘                            │
                         │       │               PAY_HOST=http://my_go_pay:9090                   │
                         │       │                                                                │
                         │       └──► 外部 RDS MySQL (ENABLE_MYSQL=false 时)                      │
                         │              或 db:3306 (ENABLE_MYSQL=true 时，profiles: mysql)         │
                         │                                                                       │
                         └───────────────────────────────────────────────────────────────────────┘
```

### 流量说明

```
客户端 ──:80──► Nginx ──反向代理──► app(C++ 8080)
                                        │
                                    PAY_HOST
                                        ▼
                                  pay(Go 9090)
```

- **外部请求全部走 Nginx :80 入口**，反向代理到 app（C++ 服务）
- **pay（Go 支付服务）不对外暴露**，仅由 app 通过内网 `http://my_go_pay:9090` 调用
- 使用云 RDS 时（`ENABLE_MYSQL=false`），`my_net` 中**不存在 db 容器**，app 直接通过外网连接 RDS

### 网络说明

- **网络名称**：`my_net`（在 `docker-compose.yml` 的 `networks` 块中定义，driver 为 `bridge`）
- **网络类型**：Docker 自定义 Bridge 网络
- **DNS 解析**：Docker 为自定义网络内置 DNS，容器可通过 `container_name` 互相访问：
  - Nginx 通过 `app:8080` 访问 C++ 服务
  - app 通过 `http://my_go_pay:9090` 调用 Go 支付服务（`PAY_HOST` 环境变量）
  - 使用内置 MySQL 时，app 通过 `db:3306` 连接；使用云 RDS 时，app 直接连接外部地址（`DB_HOST` 填实际 IP/域名）

### 端口映射（宿主机可见端口）

| 容器 | 宿主机端口 | 容器内端口 | 说明 |
|---|---|---|---|
| nginx | `80` | `80` | 唯一对外入口，HTTP 反向代理 |
| app | `80` | `8080` | 可直接访问（通常通过 Nginx 间接访问） |
| pay | `9090` | `9090` | 本地调试用，生产环境应通过安全组限制 |
| db | `3306` | `3306` | **仅在使用内置 MySQL 时存在**（`ENABLE_MYSQL=true`），云 RDS 部署时无此容器 |

### 如何自定义网络

如果需要将容器加入已有的外部网络（例如与其他服务共享网络），可以在 `docker-compose.yml` 中引用外部网络：

```yaml
# docker-compose.yml
services:
  nginx:
    networks:
      - my_net
      - external_net    # 新增：加入已有外部网络
  app:
    networks:
      - my_net
      - external_net    # 新增

networks:
  my_net:
    driver: bridge
  external_net:
    external: true      # 引用已存在的 Docker 网络（如 docker network create shared-net 创建的）
```

如果需要指定具体的外部网络名称：

```yaml
  external_net:
    external:
      name: your-existing-network-name
```

### Nginx 路由规则

```
客户端请求                          Nginx 转发目标
───────────────────────────────────────────────────────
GET /                      →       静态文件 /usr/share/nginx/web/html/index.html
GET /api/*                 →       upstream crow_backend → app:8080
POST /pay/notify           →       upstream go_pay → my_go_pay:9090（微信回调）
POST /pay/mockOrderQuery/  →       upstream go_pay → my_go_pay:9090（Mock 查单）
```

## 部署流程概览

```
推送 main 分支
    │
    ▼
GitHub Actions (deploy.yml)
    │
    ├── build-and-push
    │   ├── clone C++ 代码 → Ubuntu 22.04 编译 → 推送 ACR
    │   └── clone Go 代码  → golang:1.23 编译   → 推送 ACR
    │
    ├── deploy-to-server-1
    │   ├── envsubst 生成 docker-compose.yml 和 nginx.conf
    │   ├── SCP 上传到服务器 /opt/myapp/
    │   └── SSH 执行: down → pull → up
    │
    └── deploy-to-server-2 (可选，需配置第二套 ACR + SSH)
        └── 同上
```

### `DEPLOY_TARGET` 控制

在 `deploy.yml` 顶部设置：

```yaml
env:
  DEPLOY_TARGET: 1  # 1=只部署服务器1, 2=只部署服务器2, both=都部署
```

### 服务器端手动操作

```bash
cd /opt/myapp

# 修改 .env 或 nginx.conf 后
docker compose restart nginx   # 改 nginx.conf 后重启

# 这两步可以不需要，重新nginx就可以了
docker compose pull            # 拉新镜像
docker compose up -d           # 启动容器

# 完整清理（含数据卷，慎用）
docker compose --profile mysql down -v && rm -rf mysql_data/
```

## HTTPS 配置

### 1. 准备 SSL 证书

在云服务器 `/opt/myapp/certs/` 目录下放置证书文件：

```
/opt/myapp/certs/
├── server.crt    # 证书文件
└── server.key    # 私钥文件
```

证书来源：
- **Let's Encrypt**：使用 certbot 免费申请
- **云厂商**：从阿里云/腾讯云等申请后下载

### 2. 启用 HTTPS server 块

编辑 `/opt/myapp/nginx.conf`，找到注释掉的 HTTPS server 块，取消注释：

```nginx
# HTTPS 服务（取消注释后启用）
server {
    listen 443 ssl;
    server_name ${SERVER_NAME};

    ssl_certificate         /etc/nginx/certs/server.crt;
    ssl_certificate_key     /etc/nginx/certs/server.key;
    ssl_protocols           TLSv1.2 TLSv1.3;
    ssl_ciphers             HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # === 需要 HTTPS 的接口写在这里 ===
}
```

### 3. 重启 Nginx

```bash
cd /opt/myapp
docker compose restart nginx
```

### 注意事项

- 证书未到位时 HTTPS server 块保持注释，否则 Nginx 容器启动失败
- `certs/` 目录通过 `docker-compose.yml` 中 `./certs:/etc/nginx/certs:ro` 只读挂载到容器内
