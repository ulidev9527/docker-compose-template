# zDockerCompose
服务器 docker compose 文件收集仓库，用于快速搭建服务器环境。  
docker compose 配置  

## 前言
工作中巧合下重拾服务器开发，为了分布式而分布式，精力有限抛弃 `k8s` 那套直接用 `Docker-Compose` 方式手搓一套自己的服务器环境  

阅读接下来的内容需要具备: 基础的服务器相关知识(`服务器安装/网络配置/shell等`) / `Docker` 和 `Docker Compose` 基本使用  

注1: 项目 `docker` 文件夹中  
- `sys_**` 表示必要容器配置
- `tp_**` 表示模版容器配置
- `ts_**` 表示模版正在测试中
- 使用的时候注意修改对于容器的服务名称, 这个涉及到各个容器之间的调用, 当然, 你也直接使用 `tp_traefik` 服务发现功能进行更高级的管理。
注2: `docker-compose` 中的 `ports` 都是关闭的, 根据实际情况自己调整


## 安装

### 核心工具：
- 服务器 [Ubuntu 22.0.4](https://ubuntu.com)  
- 容器 [Docker](https://www.docker.com)  
- 容器管理 [Portainer](https://www.portainer.io)  
- DNS服务 [sameersbn/docker-bind](https://github.com/sameersbn/docker-bind), 根据需要安装。  
- 代理管理 [nginx proxy manager](https://nginxproxymanager.com)  

### 服务器安装和配置
1. [Ubuntu 下载](https://ubuntu.com/download/server)  

    下载完成后自行完成安装服务器  

2. `root` 用户添加密码 `$ sudo passwd root`  

3. 切换 `root` 用户 `$ su`  

4. 关闭 `systemd-resolved` 服务
    ```
    systemctl stop systemd-resolved
    systemctl disable systemd-resolved
    ```
5. 修改 `DNS` 解析 **注: 如果不需要自建 `DNS` 服务可以忽略这一步**   

    `Ubuntu 22.04` 停止和关闭 `stop systemd-resolved` 服务
    ```
    systemctl stop systemd-resolved
    systemctl disable systemd-resolved
    ```
    
    修改系统 `DNS` 解析  
    - 内容为添加两个 `DNS` 解析地址和`hostname`补全
    - `nameserver 127.0.0.1` 为后面会在服务器添加 `DNS` 解析服务, 如果服务器不建 `DNS` 解析服务可以不添加
    - `DNS` 解析可以根据需要手动修改
    - **重启服务器后需要再次执行**
    ```
    rm /etc/resolv.conf
    cat > /etc/resolv.conf <<EOF
    nameserver 223.6.6.6 # 阿里 DNS
    nameserver 127.0.0.1 # 自建 DNS
    nameserver 8.8.8.8 # Google DNS
    search .
    EOF
        ```
### Docker 安装

1. 安装  
    [官方文档](https://docs.docker.com/engine/install/ubuntu/)  

    下面是 `Ubuntu` 系统的安装最新稳定版 `Docker` 的命令, 直接复制后粘贴到命令行等待执行完成, 也可以去[这里安官方文档]((https://docs.docker.com/engine/install/ubuntu/))安装  

    ```
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

2. 验证 `$ docker version`  

    正常情况会输出下面这些内容, `...` 是本文档省略的一些内容  
    ```
    Client: Docker Engine - Community
    Version:           25.0.3
    ....

    Server: Docker Engine - Community
    Engine:
    Version:          25.0.3
    ....
    ```

3. 修改 `docker` 配置  
    - 内容主要是添加国内镜像和日志切割
    - **如果文件已存在需要进行手动修改**
    ```
    cat > /etc/docker/daemon.json <<EOF
    {
        "registry-mirrors": [
            "https://dockerproxy.com",
            "https://hub-mirror.c.163.com",
            "https://mirror.baidubce.com",
            "https://ccr.ccs.tencentyun.com"
        ],
        "log-driver": "local",
        "log-opts": {
            "max-size": "10m",
            "max-file": "10"
        }
    }
    EOF
    ```
4. 创建默认网络 `$ docker network create net_def`  

    上面命令创建可能会导致和云服务器之间的 `IP` 段冲突，必要情况下可以[根据文档](https://docs.docker.com/network/)配置 `IP` 段。
    

### 容器创建  

1. 拉取需要的镜像
    ```
    docker pull portainer/portainer-ce:latest
    docker pull jc21/nginx-proxy-manager:latest
    docker pull sameersbn/bind:latest # 如果不需要 DNS 服务可以不用拉取这个
    ```
2. 临时的 [Portainer](https://www.portainer.io) 服务  
    启动一个临时的 `Portainer`  
    ```
    docker run -d \
    -p 8000:8000 -p 9443:9443 -p 9000:9000 \
    --name portainer_start \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v sys_portainer_data:/data \
    portainer/portainer-ce:latest
    ```
    访问: `https:服务器IP:9443` 进行账号密码配置  

    如果配置密码账号超时，重启 `docker restart portainer_start` 后再次访问  

    如果忘记密码  
    先停止 `docker stop portainer_start`  
    如果使用的 [sys_portainer/docker-compose.yml](./docker/sys_portainer/docker-compose.yml), 先找到对应的容器再停止  
    然后执行:`docker run --rm -v /var/lib/docker/volumes/sys_portainer_data/_data:/data portainer/helper-reset-password`, 首次执行会先拉取一个镜像  
    输出  
    ```
            {"level":"info","filename":"portainer.db","time":"2024-02-22T14:29:46Z","message":"loading PortainerDB"}
    2024/02/22 14:29:46 Password successfully updated for user: admin
    2024/02/22 14:29:46 Use the following password to login: 4?o3Q@7b`A0vH)N<s8XjyC$lZ9d6-]U5
    ```
    其中: `... Use the following password to login: ****` 中的 `****` 就是密码  
    最后: `docker start ***` 启动容器完成改名  

3. 如何进行 `docker-compose.yml` 文件创建容器  
    访问: `https:服务器IP:9443` (后面会修改为域名方式访问)  

    点击 `Home` > `local 容器` > `Stacks` > `Add stack` 进行创建  

    根据需要选择创建的方式  

    `Name` 字段填写对于名称  
    `Environment variables` 根据需要添加环境变量  
    `Deploy the stack` 按钮点击后即可创建  

4. 创建必要容器   
    - 容器管理 `portainer`  
        - 访问: `https:服务器IP:9443`
        - [DockerCompose 文件](./docker/sys_portainer/docker-compose.yml)
        - 按照 **3. 如何进行 `docker-compose.yml` 文件创建容器** 教程安装
    - 代理管理 nginx proxy mamager  
        - 访问: `https:服务器IP:9443`
        - [DockerCompose 文件](./docker/sys_nginxproxymanager/docker-compose.yml)
        - 按照 **3. 如何进行 `docker-compose.yml` 文件创建容器** 教程安装
        - 完成后访问: `http:服务器IP:81`
        - 初始邮箱: `admin@example.com`
        - 初始密码: `changeme`
        - 进行账号密码配置
        - 如果密码账号不正确, 参考[官方文档](https://nginxproxymanager.com/guide/#quick-setup)
    - DNS服务 bind  
        - 不使用自建 `DNS` 可以忽略
        - 访问: `https:服务器IP:9443`
        - [DockerCompose 文件](./docker/sys_bind/docker-compose.yml)
        - 环境变量
            - `BIND_HOST` `dns` 解析服务控制台域名
                - 示例: `bind.self.local`
            - `BIND_ROOT_PASSWORD` `dns` 解析服务 `root` 密码
                - 示例: `123!@#abcABC`

### 配置反向代理  
访问: `http:服务器IP:81`(后面会修改为域名方式访问)  
- http 代理
    - 选择 `Hosts` > `Proxy Hosts` > `Add Proxy Host` (右上)
    - `Domain Names` 里面填写自己自定义的一个域名
    - `Forward Hostname / IP`: `docker-compose.yml` 中的服务名称, 文件中 `services:` 的下级就是服务名, 也可以配置其他的域名或者直接配置 `IP`
    - `Forward Port`: 服务的访问端口
    - 其他配置根据需要进行添加
    - 点击 `Save` 按钮存储
    - 退出创建界面后，服务的 `Status` 为 `Online` 表示正常, 如果不正常可以查看容器日志
- Stream 代理
    - 选择 `Hosts` > `Streamd` > `Add Stream` (右上)
    - `Incoming Port` 填写服务器自己的端口，需要 [nginx代理](./docker/sys_nginxproxymanager/docker-compose.yml) `ports` 中开放
    - `Forward Host` 同上面 `Forward Hostname / IP`
    - `Forward Port` 同上面 `Forward Port`
    - 点击 `Save` 按钮存储
    - 退出创建界面后，服务的 `Status` 为 `Online` 表示正常, 如果不正常可以查看容器日志
### `portainer` 域名访问  
- 按照 **配置反向代理** 中进行域名配置  
- 服务器命令行执行 `$ docker stop portainer_start`  
- 找到通过 `portainer` 中使用 `docker-compose.yml` 创建的容器 `$ docker ps -a | grep portainer`  
- 重启容器 `$ docker restart ***`, 本项目中默认容器名为 `sys_portainer-****`  
- 将本机 `host` 文件中添加域名解析
- 访问域名

### `nginx proxy manager` 域名访问
- 按照 **5. 配置反向代理** 中进行域名配置  
- 将本机 `host` 文件中添加域名解析
- 访问域名

### `dns` 服务域名访问和配置 `DNS`  
不使用自建 `DNS` 可以忽略  
1. 域名访问  
    - 按照 **5. 配置反向代理** 中进行域名配置  
    - 将本机 `host` 文件中添加域名解析
    - 访问域名
    - 账号: `root` 密码: 是 `BIND_ROOT_PASSWORD` 环境变量
    - 登陆后, 域名会被添加一个 `:10000` 的端口号, 去掉端口号重新访问域名即可

2. `DNS` 服务配置  
    - 创建 `DNS` 解析
        - 参考 [解析教程](https://blog.csdn.net/qq_36961626/article/details/123314087) 进行 `dns` 服务创建
        - 进入: `Servers (服务)` > `BIND DNS Server (BIND dns 服务)` > `Global Server Options` > `Zone Defaults` > `Default zone settings`
            - `Allow transfers from..`: 选择 ` Listed..`, 内容填: `any`
            - `Allow queries from..`: 选择 ` Listed..`, 内容填: `any`
        - 进入: `Servers (服务)` > `BIND DNS Server (BIND dns 服务)` > `Existing DNS Zones` > `Create master zone`
            - `Domain name / Network`: 填写根域名, 比如: `a.local` / `self.local` / `a.com` / `a.io` ...
            - `Master server`: 填写 `localhost.`
            - `Email address`: 填写一个正常的邮箱地址, 无论是否可用, 比如: `a@b.c`
        - 点击 `Create` 按钮创建
        - 创建完成进入到了: `Edit Master Zone`, 这个也可以从: `Servers (服务)` > `BIND DNS Server (BIND dns 服务)` > `Existing DNS Zones`  > `对应域名(地球图标位置)` 进入
        - 点击 `Address` 按钮
            - `Name`: 填写要解析的域名, 比如: `a.com` / `bind.a.com` / `*.a.com`
            - `Address`: 填写解析的目标IP, 比如: `127.0.0.1`, **是否可以使用 `IP:Port` 待测**
            - 点击 `Create` 按钮创建一个域名解析

    - 添加第三方解析
        - 主要用于可以解析本机配置的 `DNS` 解析之外的域名解析功能
        - 进入: `Servers (服务)` > `BIND DNS Server (BIND dns 服务)` > `Global Server Options` > `Forwarding and Transfers (转发和输出)`
        - 添加 IP `8.8.8.8` `8.8.4.4`, 这个是第三方 `dns` 解析添加，不然无法解析其他网站。
        - `Save` 按钮保存
3. 电脑修改 `DNS`
    - 各系统修改 `DNS` 解析都不一样, 自行搜索解析教程
    - 修改后刷新一下 `DNS`, 自行搜索教程
    - 浏览器输入对应域名测试解析情况, 也可以使用 `dig` / `host` / `ping` / `curl` 等方法测试, 自行搜索教程