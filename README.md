# gitea-ci

个人或小团队搭建私有 git 加 ci 服务的参考部署示例，重点关注私有部署的安全配置。借助 [gitea](https://gitea.io) 和 [droneci](https://drone.io) 及可选使用 [caddy](https://caddyserver.com/)，满足部署在云服务器或纯内网的不同要求。

## 服务与配置说明
使用 caddy 作为访问入口，gitea 作为代码仓库（数据库用 postgresql），droneci 进行自动化构建。

关键的配置特点是 droneci 使用 oauth 连接 gitea。即使内网部署，gitea/dronci server/droneci runner 之间也通过 https 连接，证书为统一的内部 CA 颁发，防止 token 被窃听轻易导致源代码泄露。


### 公共配置
.env.example 示例配置说明：
* DATA_PATH，所有服务的数据存储会分别挂载到该目录的特定子目录下
* MAIN_DOMAIN 指定主域名，示例中为 `example.com`，那么 git 服务和 ci 服务的访问域名就是 `https://git.example.com` 和 `https://ci.example.com`
* TZ 指定所有服务容器系统的默认时区
* CACERT_PATH 指定 HOST 系统中根证书文件位置，该文件会挂载到容器系统中。
    - 为了让 droneci 及 runner 能通过 https 访问 git，容器系统需要正确的根证书。
    - 内网部署时需要特别注意，caddy 生成的自签名根证书需要先安装到 HOST 系统中才能在其它服务中生效。
* INTERNAL_DNS 和 EXTERNAL_DNS 可以指定两个 DNS 服务器。
    - 主要目的是内网部署时可以把第一个指定为内网 DNS。例如本人测试时在路由器上用 hosts 解析 exmaple.com 域名，让它可以在内网使用。不用买域名了🤣
    - 公网部署请随意指定公共 DNS 服务器

### caddy
caddy 是可选的。加入 caddy 作为反向代理的有两个原因：
* 一是能自动申请配置 https 证书；
* 二是内网部署时可以充当 CA 服务器，且能为其它服务提供证书服务（支持 acme 协议）。

示例中的默认配置说明：
* 默认使用的是 Caddy 本地证书，可以内网部署，不需要开放公网访问。如果允许公网访问，可以删除配置文件 caddy/Caddyfile 中的 `tls internal` ， caddy 会使用 Let's Encrypt ACME 申请免费 HTTPS 证书。
* Caddyfile 中以下配置提供本地证书服务器的 acme 协议支持。如果不需要可以删除：
  ```
  ca.{$MAIN_DOMAIN} {
    tls internal
    acme_server
  }
  ```
* Caddy 本地证书的根证书路径及在各系统的安装方法见后面的 [FAQ](#faq)


### gitea

首次启动服务之后，需要访问 https://git.example.com/install 进行初始安装配置。注意事项或安全配置建议：
* docker-compose.yml 中的环境变量配置**只影响首次启动**安装配置时显示的**默认值**。安装配置完成之后再修改，必须修改配置文件 `./data/gitea/config/app.ini`。
* 注意数据库相关配置
* 注意 url 和 ssh 端口配置
* 私有部署建议配置：
  * 启用本地模式
  * 禁止用户自助注册
  * 取消“启用 OpenID 登录”和“启用 OpenID 自助注册”
  * 启用页面访问限制（游客只能看到登录和注册页面）
  * （禁止注册时）提供初始管理员账户信息
  * 如有需要，勾选“默认情况下隐藏电子邮件地址”
* 禁止自助注册时，必须在安全配置期间创建管理员帐户
* 允许注册时，建议提供电子邮件相关配置，并勾选需要发电子邮件确认注册。
* 管理后台可以添加 Web 钩子，支持钉钉、飞书、Matrix、Telegram 等。


### droneci
虽然 droneci 的开源版本（oss）功能受限，一般使用场景还是能对付的。不过因为版本 2 开始会提示注册，所以示例中默认使用 v1.x 。可根据需要选择使用官方的 2.x 版本，或者切换到更纯粹的社区分支版本 [woodpecker](https://github.com/woodpecker-ci)。

默认配置：
* droneci 使用 gitea 的 oauth 认证登录。
  - 需要先配置好 gitea 并生成 oauth token 写到 .env_drone 里，droneci 才能正常运行。
  - 配置还需要指定用来作为 droneci 管理员的 gitea 中的某个用户名（该用户需要具备所有使用 ci 服务的仓库的权限）
* droneci 社区版只能使用默认的 sqlite 数据库
* runner 在参考配置里是部署在同一 docker 环境中，这样并不安全，建议单独主机运行。
* 注意：使用自签名证书时，runner 的 docker 运行环境也需要安装根证书。
  - docker-compose.yml 中的 `DRONE_RUNNER_VOLUMES=${CACERT_PATH:?err}:/etc/ssl/certs/ca-certificates.crt` 配置会将 docker host 的证书挂载到每个步骤运行的插件环境中。如果插件运行环境不是 alpine/debian/ubuntu 等，需要修改目标路径。
  - 这可以避免在插件的运行环境中手动安装自签名根证书。

## 部署实践

在任意支持 docker 的主机环境都可以部署。推荐 NAS 方案，多台 NAS 还能配置异地容灾备份哦。

### Linux 或 macOS 主机
要求先安装好 docker engine 和 docker-compose。Windows 也可以使用支持 WSL2 的 Docker Desktop。

1. **基础配置**
  - 复制 .env.example 为 .env，修改其中的
    - 自定义 DNS 服务器（方便解析内网域名或测试域名）
    - 主域名（服务的统一主域名）
    - 根证书路径（这个根据 docker 运行环境的 host 主机来选择，共享 host 的根证书到容器中）
  - 修改 .POSTGRES_PASSWORD 文件，指定 postgresql 的密码
2. 启动 caddy
  - `docker-compose up -d caddy`，先启动 caddy
  - 如果是内网部署并启用了本地证书服务，参考[如何安装和信任自签名证书](https://kompost.cn/posts/%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%85%E5%92%8C%E4%BF%A1%E4%BB%BB%E8%87%AA%E7%AD%BE%E5%90%8D%E8%AF%81%E4%B9%A6/)把 Caddy 生成的根证书安装到 docker host 系统中
  - 完全关闭浏览器再重新打开，访问 https://ca.example.com 不会弹出证书警告，则证明根证书已安装好。
3. 启动并配置 gitea
  - `docker-compose up -d gitea`，启动 gitea
  - 如果服务启动失败并提示 `mkdir: can't create directory '/var/lib/gitea/git': Permission denied`，需要修改 ./data/gitea/ 下两个目录的所有者权限：`sudo chown 1000:1000 ./data/gitea/config ./data/gitea/data`
  - 访问 https://git.example.com/install，参考前面的说明完成 gitea 的配置。
4. 准备 droneci 配置
  - 在 gitea 中，从右上角用户菜单进入**设置->应用**，应用名称：drone，重定向 URI：https://ci.example.com/login，**创建应用**。
  - 把显示的客户端 ID 和客户端密钥复制到 .env_drone 中。
  - 使用`openssl rand -hex 16`生成一段随机密码，作为 .env_drone 中的 DRONE_RPC_SECRET 值
5. 启动 drone 服务
  - 现在可以启动 droneci 了，用 `docker-compose up -d drone` 启动服务。
  - 访问 https://ci.example.com 并按提示授权登录


### NAS 环境

NAS 上运行 git 服务对个人或小团队来说是非常有性价比的选择。配合不间断电源（UPS）和双 NAS 异地同步备份，可以提供较高级别的数据安全性。

本示例中适合将主要服务部署在 NAS 上，cirunner 另找台性能更高的主机运行。

以群晖的系统为例，应用套件中可以下载 docker 套件。然后可以用两种方式运行：

1. SSH 到 NAS，以本示例中的配置为基础直接用 docker-compose 方式运行。注意端口映射有没有与群晖中的其它配置产生冲突。

2. 在 Web 管理界面的 Docker 套件中一个一个配置。需要手动配置相关的启动参数、环境变量、卷映射绑定等。

另外，如果不需要自签名证书服务，还可以直接利用群晖 NAS 上的反代服务（也需要 Web 界面手动配置），不需要 Caddy。


* 如何在群晖 Synology 上使用自定义 docker 镜像？
  - 方法一，本地 build 之后，上传 drone-oss.tar.gz 到 NAS，然后导入（Docker 套件 -> 映像 -> 新增 -> 从文件添加）
    ```bash
    docker build -t drone:oss -f ./drone/server.Dockerfile
    docker save drone:oss | gzip > drone-oss.tar.gz
    ```
  - 方法二，使用 SSH 连接到群晖系统，用 docker 命令本地 build

## 升级与安全
* Gitea 使用 Dockerfile 集成证书，需要用 `docker-compose build gitea --no-cache` 重新 build 镜像，再重启服务
  - 注意关注 Gitea 的版本更新日志，出现安全性改进时及时升级
* Caddy 只用 `docker-compose pull caddy` 再重启就可以了
* Droneci 1.x 版本不会有升级了，换用 2.x 或 [woodpecker](https://github.com/woodpecker-ci)


## FAQ
* 使用 Caddy 本地自签名证书，如何下载根证书？
  * Caddy 本地证书的根证书首次运行自动生成，路径为 data/caddy/caddy/pki/authorities/local/root.crt

* 如何在浏览时信任自签名证书？
  * Windows 下双击 root.crt，选择安装到“受信任的根证书颁发机构”中，支持 Edge 或 Chrome 等。
  * Firefox 浏览器需要在“选项”里搜索“证书”，查看证书并选择“导入”到证书颁发机构。
  * Linux 下某些命令行工具可以通过参数指定忽略自签名证书警告，或者指定根证书文件。
  * Linux 下要安装到系统证书存储，不同发行版有所区别。
    以 alpine 为例：`apk add ca-certificates && cp root.crt /usr/local/share/ca-certificates/ && update-ca-certificates`
 