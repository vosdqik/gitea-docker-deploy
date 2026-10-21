# gitea安装部署：Docker Compose 搭私有 Git，数据库选型、HTTPS、SSH 克隆与备份一次做对

Gitea 安装部署本身不复杂：一台 2GB 内存的云服务器、一份 compose 文件，就能得到一个带 PR、Issue、Webhook 和包仓库的私有 Git 服务。容易出问题的地方在安装之外——数据库选型、SSH 克隆端口、备份范围，这三件事任何一个考虑错了，后面都得补课。

Gitea 是 Go 写的单二进制服务，MIT 许可，资源占用很小，空闲时只有几百 MB 内存。真正吃配置的是仓库体量和 CI 任务。下面的流程按“先定方案，再动手装，最后补运维”的顺序走。

> **赶时间的话**：个人用或两三个人的小组，一台 1–2GB 内存、40GB 磁盘的云服务器，Docker Compose 配 SQLite 或 PostgreSQL 就能跑起来；确定要跑 CI 再单独加一台 runner 机器。
> 👉 [在腾讯云开一台部署用的云服务器](https://bit.ly/yunhuodong)

## 先定数据库和机器规格，再动手

数据库要在第一次启动前定下来。Gitea 的安装向导会把它写进 app.ini，中途换库虽然能做，但要停服务、导数据、改配置，比一开始选对麻烦得多。机器规格按仓库体量和是否跑 CI 来算，跟 Gitea 进程本身关系不大。

**数据库怎么选**

| 选项 | 适合谁 | 要接受的代价 |
| --- | --- | --- |
| SQLite | 个人、两三个人、几十个仓库 | 数据库就是 `/data/gitea/gitea.db` 一个文件，备份极简单；写操作串行，多人同时推代码加 CI 会排队 |
| PostgreSQL | 小团队，开了代码搜索，并发推送 | 多一个 db 服务和一块卷，备份要连库一起做 |
| MariaDB / MySQL | 团队本来就有 MySQL 运维经验 | Gitea 配置里 `DB_TYPE` 填 `mysql`，驱动和 MySQL 相同 |

一个人用，或者只是给自己搭个私有仓库，SQLite 完全够。团队到了三五个人、经常同时 push 和跑流水线，再换 PostgreSQL。

**机器规格怎么选**

| 配置 | 能撑住什么 | 说明 | 入口 |
| --- | --- | --- | --- |
| 1 vCPU / 1GB | 个人 forge、SQLite、几十个仓库 | 够用，但别在上面跑 CI | [选 1GB 入门配置](https://bit.ly/yunhuodong) |
| 2 vCPU / 2GB | 小团队 + PostgreSQL + 代码搜索 + 并发推送 | 比较稳妥的默认档 | [选 2GB 默认配置](https://bit.ly/yunhuodong) |
| 4 vCPU / 4GB 以上 | 同一台机器上还要跑 Actions runner | 加的是构建预算，Gitea 本身用不了这么多 | [选 4GB CI 配置](https://bit.ly/yunhuodong) |

磁盘是最容易低估的一项。仓库、LFS 对象、包仓库和 CI 产物全落在同一个卷上，而且没有一个会自己变小。起步 40GB，选一台之后能在线扩容的机型，比一开始买超大磁盘划算。

👉 [挑一台磁盘支持在线扩容的云服务器](https://bit.ly/yunhuodong)

## 用 Docker Compose 部署

官方镜像已经把运行时依赖打进去了，一个 compose 文件加一个数据库服务就能起来。下面这份配置用 PostgreSQL，版本固定到 1.27.3（2026 年 9 月的稳定版）。把域名、密码和 SSH 端口换成你自己的，就可以直接 `docker compose up -d`。

yaml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: 换成长随机密码
      POSTGRES_DB: gitea
    volumes:
      - ./postgres:/var/lib/postgresql/data

  gitea:
    image: docker.gitea.com/gitea:1.27.3
    container_name: gitea
    restart: unless-stopped
    depends_on:
      - db
    environment:
      USER_UID: "1000"
      USER_GID: "1000"
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: 换成长随机密码
      GITEA__server__DOMAIN: git.example.com
      GITEA__server__ROOT_URL: https://git.example.com/
      GITEA__server__SSH_DOMAIN: git.example.com
      GITEA__server__SSH_PORT: "222"
      GITEA__server__SSH_LISTEN_PORT: "22"
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "127.0.0.1:3000:3000"
      - "222:22"


需要改的地方只有三处：三行里的域名、两处数据库密码、以及 222 这个对外 SSH 端口（被占用就换一个，`SSH_PORT` 和 ports 映射要同步改）。镜像写死具体版本号，不要图省事用 latest，升级应该是你主动做的动作。

bash
docker compose up -d
docker compose logs -f gitea


等日志跑完，浏览器打开 `http://服务器IP:3000` 进安装向导。数据库那一栏是关键：主机填 `db`，不是 localhost，也不是 127.0.0.1——`db` 是 compose 里的服务名，容器之间靠它解析。类型选 PostgreSQL，库名、用户名、密码和 compose 里保持一致。填错的话向导只会一直报连接失败，不会告诉你具体错在哪。

服务器还没买的话，compose 写得再对也没地方跑。👉 [先在腾讯云开一台 2GB 内存的机器，再回来复制上面的配置](https://bit.ly/yunhuodong)

升级的流程就是改 compose 里的版本号，然后 `docker compose pull && docker compose up -d`。跨大版本之前先做一次完整备份。

## HTTPS 和 ROOT_URL

HTTPS 交给反向代理，Caddy 是最省事的选择——一个域名块加一行反代，证书自动签发、自动续期。Gitea 容器只监听 `127.0.0.1:3000`，公网流量全部走代理，3000 端口不会直接暴露出去。


git.example.com {
    reverse_proxy 127.0.0.1:3000
}


云服务器控制台里的安全组（轻量应用服务器叫防火墙）放行 80、443 和 222。80 要留着，证书签发和续期都走它。3000 和 5432 不要对公网开放。

ROOT_URL 决定 Gitea 生成的所有外部链接：克隆地址、邮件里的跳转、OAuth 回调。它填错的时候，典型表现是页面能打开但克隆命令里的域名不对，或者登录完被重定向到内网地址。碰到这类问题先改 ROOT_URL，别急着在反向代理里加 header 去绕——源头就一个配置项，绕过去只会在下一个地方再冒出来。

## SSH 克隆连不上，先核对两个端口

SSH 克隆失败，九成是端口映射没对上，或者两个 SSH 端口配得不一致。Gitea 有两个容易搞混的配置：`SSH_PORT` 是显示在克隆 URL 里的对外端口，`SSH_LISTEN_PORT` 是容器内 sshd 实际监听的端口。上面那份配置对外 222、对内 22，克隆地址就是：

bash
git clone ssh://git@git.example.com:222/用户名/仓库.git


宿主机 22 端口留给系统自己的 sshd，Gitea 用 222，两者不冲突。测试认证是否通了：

bash
ssh -T -p 222 git@git.example.com


返回带用户名的欢迎信息就说明认证正常。公钥要加在 Gitea 的“用户设置 → SSH / GPG 密钥”里，加到服务器的 authorized_keys 里没用，Gitea 自己管理这套公钥。

## 装完先关掉三样东西

新装的 Gitea 默认开放注册、默认用 Gravatar 头像、新建仓库默认公开。放在公网上的实例，这三项都该改。

在 `/data/gitea/conf/app.ini` 里：

ini
[service]
DISABLE_REGISTRATION = true

[repository]
DEFAULT_PRIVATE = true


Gravatar 那项关掉的理由跟功能无关，是隐私：开启后访客加载头像时，邮箱的哈希会被发到第三方 CDN。私有实例没必要为了一个头像把这个信息送出去。

改完 `docker compose restart gitea` 生效。有个坑要注意：如果某个配置项已经在 compose 的环境变量里设过，改 app.ini 不会生效，环境变量优先级更高，得去改 compose 或者删掉那行。

## 备份要连数据库一起

只打包 `/data` 目录不算完整备份。Gitea 的数据分两半：仓库、附件、配置、密钥在 `/data` 卷，用户、issue、PR 这些关系数据在数据库卷里。恢复的时候两个都要，缺一个就是半套系统。

用 SQLite 的人省事一些，数据库文件就在 `/data/gitea/gitea.db`，拷一个目录就全了，这是个人实例选 SQLite 的一个实际好处。用 PostgreSQL 或 MariaDB 的话，备份脚本要包含 `pg_dump` 或 `mysqldump`，并且和 `/data` 的备份放在同一批次执行，保证两边时间点接近。

备份做完要试恢复。找一台干净机器，还原两个卷、起 compose、登录看仓库和 issue 在不在。没试过的备份不算备份。

## 要不要装 Actions runner

Gitea 自己不会执行 workflow，任务是由 runner 拉过去跑的。workflow 文件放在仓库的 `.gitea/workflows/` 目录，语法和 GitHub Actions 一样，现成的 YAML 大多能直接搬过来。Actions 从 1.21 起默认启用。

runner 的官方镜像 2026 年是 `docker.io/gitea/runner:3`。旧名字 act_runner 在网上还能搜到大量教程，别搞混。注册 token 在“站点管理 → Actions → Runners”里生成。

runner 消耗的是 CPU 和内存，跟 Gitea 进程没关系。仓库代码在 runner 里执行，把它和 Gitea 放在同一台机器上，等于让构建任务跟 Git 服务抢资源。能分开就分开，这也是前面那张表里“要跑 CI”直接跳到 4GB 以上的原因。另外 runner 执行的是仓库里的代码，不要把它和 Gitea 塞进同一个容器。

## 不用容器：二进制加 systemd

不想在服务器上跑 Docker 的话，二进制安装也能用，代价是 systemd、目录权限和升级都要自己管。

bash
sudo wget -O /usr/local/bin/gitea https://dl.gitea.com/gitea/1.27.3/gitea-1.27.3-linux-amd64
sudo chmod +x /usr/local/bin/gitea
sudo adduser --system --group --home /var/lib/gitea git
sudo mkdir -p /var/lib/gitea /etc/gitea


权限要分开：`/var/lib/gitea` 归 git 用户，`/etc/gitea` 归 `root:git`、权限 770。配置文件让 git 用户可写，等于让服务自己改自己的配置，没必要。systemd 单元里设 `GITEA_WORK_DIR=/var/lib/gitea`，`ExecStart` 指向 `gitea web --config /etc/gitea/app.ini`。

1.27 起官方发布的二进制带 Sigstore 签名，下载后可以在官方下载站对照校验值再装。

## 按你的情况选

一个人用、仓库不多：1GB 内存 + SQLite + compose，备份就是打包 `/data`。三五个人的团队：2GB 内存 + PostgreSQL，关掉注册，备份脚本覆盖两个卷。要跑 CI：Gitea 一台、runner 另一台，workflow 放 `.gitea/workflows/`。

买机器之前确认两件事：磁盘能不能在线扩容，安全组能不能只放行 80、443、222。这两项装完再改，成本比一开始选对高得多。

服务器还没着落：👉 [去腾讯云挑一台按上面配置能跑 Gitea 的机器](https://bit.ly/yunhuodong)
