
## 一、备份jenkens

**分三个等级：每日备份，需要但不用每日备份，不需要备份**

---

### ✅ 每日备份（小，配置文件）

这些是 Jenkins 的配置文件，体积小可以每日备份。

| 路径              | 大小    | 原因                                      |
| --------------- | ----- | --------------------------------------- |
| `config.xml`    | 40K   | **Jenkins 核心配置**，必须备份                   |
| `*.xml`（根目录下所有） | ~几百 K | 包含所有系统级配置（插件、工具、安全等）                    |
| `jobs/`         | 几M    | **所有 Job 的定义**，备份时会排除 `builds/` 目录，所以很小 |
| `users/`        | 648K  | **所有用户的账号、权限、API Token**                |
| `secrets/`      | 48K   | **加密密钥**，如果没有它，Jenkins 无法解密数据           |
| `sh/`           | 564K  | 自定义脚本，**建议备份**                          |
| `*.sh`          | 十几K   | 自定义脚本                                   |
| `*.py`          |       |                                         |



### ✅ 需要但不用每日备份（大，软件包等）
这些是CI/CD流水线需要的软件包、插件、工具等，重要但是体积较大不需要每日备份

| 路径                                    | 大小   | 原因                            |
| ------------------------------------- | ---- | ----------------------------- |
| `/home/opt/jenkens/plugins/`          | 389M | **所有已安装插件**，确保恢复后版本一致         |
| `/home/opt/jenkens/upgrade_platform/` | 6.1G | 升级工具，不是 Jenkins 配置            |
| `/home/opt/jenkens/repo`              | 6G   | maven本地仓库                     |
| `/home/opt/jenkens/python3/`          | 496M | Python 环境，**建议备份**（如果构建依赖）    |
| `/home/opt/jenkens/.m2`               | 2.6M |                               |
| `/home/opt/jenkens/jdk-21.0.10/`      | 332M | JDK 安装包，可重新下载                 |
| `/home/opt/jenkens/nodes/`            | 354M | **Agent 节点配置**，如果你有分布式构建，需要备份 |
| `/home/opt/jenkens/tools/`            | 1.3G | jenkins工具管理目录                 |
| `/home/opt/jenkens/userContent/`      | 4k   | 用户自定义的图片、CSS、HTML 等           |
| `/home/packing_502/ipaas-packing/`    | 112M | 打包目录                          |
| `/home/packing_502/uosv25_x86`        | 16G  | uos x86的依赖软件包                 |
| `/home/packing_502/uosv25_arm`        | 17G  | uos arm的依赖软件包                 |
| `/home/packing_502/uosv25ti_x86`      | 4.2G | uos x86 本地云查的依赖软件包            |
| `/home/maxs_331_submit/bank_deploy`   | 16G  |                               |
| `/home/maxs_331_submit/deploy`        | 11G  |                               |
| `/home/maxs_331_submit/op_deploy`     | 1.4G |                               |
| `/opt/apache-maven-3.8.6`             | 11M  |                               |

### ❌ 不需要备份的目录和文件（可以排除）

这些是“可重建的”或“非配置类”数据，备份它们只会增加体积。

| 路径               | 大小   | 原因                        |
| ---------------- | ---- | ------------------------- |
| `caches/`        | 57G  | 插件缓存，可重建                  |
| `workspace/`     | 390G | 构建工作空间，可重建                |
| `logs/`          | 31M  | 日志文件，可丢弃                  |
| `war/`           | 106M | Jenkins Web 应用包，镜像自带      |
| `updates/`       | 5.7M | 插件更新缓存，可丢弃                |
| `Temp/`          | 12K  | 临时文件，可丢弃                  |
| `maxs*`          | ~60G | 自定义产物或代码，不是 Jenkins 配置    |
| `image/`         | 155M | 镜像文件，不是 Jenkins 配置        |
| `fingerprints/`  | 42M  | 文件指纹记录，可重建                |
| `workflow-libs/` | 0    | 空目录，忽略                    |
| `jobs_bak/`      | 3.0M | 备份副本，忽略                   |
| `Sonar-scanner/` | 165M | **SonarQube 扫描器工具**，可重新下载 |

### 最终备份脚本
``` shell
## step 1
cd /home/opt

## step 2
tar -czvf jenkins_core_$(date +%Y%m%d).tar.gz \
    --exclude='jenkens/jobs/*/builds' \
    --exclude='jenkens/jobs/**/builds' \
    --exclude='jenkens/caches' \
    --exclude='jenkens/workspace' \
    --exclude='jenkens/tools' \
    --exclude='jenkens/logs' \
    --exclude='jenkens/war' \
    --exclude='jenkens/updates' \
    --exclude='jenkens/Temp' \
    --exclude='jenkens/repo*' \
    --exclude='jenkens/maxs*' \
    --exclude='jenkens/upgrade_platform' \
    --exclude='jenkens/image' \
    --exclude='jenkens/jdk-*' \
    --exclude='jenkens/fingerprints' \
    --exclude='jenkens/workflow-libs' \
    --exclude='jenkens/jobs_bak' \
    --exclude='jenkens/Sonar-scanner' \
    --exclude='jenkens/*.log' \
    --exclude='jenkens/*.tmp' \
    --exclude='jenkens/nodes' \
    --exclude='jenkens/python3' \
    --exclude='jenkens/plugins' \
    jenkens/config.xml \
    jenkens/*.xml \
    jenkens/jobs/ \
    jenkens/users/ \
    jenkens/secrets/ \
    jenkens/sh/ \
    jenkens/*.sh \
    jenkens/.m2/ \
    jenkens/*.py
    
# 若使用该备份文件夹，需要修改用户和用户组，确保目录权限正确
chown -R 1000:1000 /home/opt/jenkens

# 重启docker
docker restart myjenkins
```


## 二、新部署宿主机
> 如果要新部署一台jenkins服务器，需要按照以下步骤：

### 1.环境准备(新部署jenkins实例)
**宿主机需要准备以下目录：**
``` text
直接拷贝：
/home/maxs_331_submit/bank_deploy
/home/maxs_331_submit/deploy
/home/maxs_331_submit/op_deploy
/home/maxs_331_submit/asap/mysql
/home/opt/jenkens
/opt/apache-maven-3.8.6

生成：
/usr/share/zoneinfo/Asia/Shanghai -> timedatectl set-timezone Asia/Shanghai  
/home/5.0.9 -> 空白，直接创建即可
/home/maxs_331_submit/asap/dockertars -> 空白，直接创建即可
/usr/bin/sshpass -> 系统自带，如果没有可以安装
/bin/python2.7  -> uos-x86编译了一个python2.7
/usr/bin/docker               # 这个 Docker 二进制文件要存在
/var/run/docker.sock          # 这个 Docker 服务本身要存在
/usr/local/sbin/mc -> 空白，直接创建即可
```
### 2. docker run命令
```sh
docker run --name=myjenkins \
        --hostname=af4fb70d1fc6 \
        --user=jenkins \
        --mac-address=02:42:ac:11:00:02 \
        --volume /home/maxs_331_submit/bank_deploy:/bank_deploy \
        --volume /usr/local/sbin/mc:/usr/local/sbin/mc \
        --volume /var/run/docker.sock:/var/run/docker.sock \
        --volume /home/5.0.9:/UAP/5.0.9 \
        --volume /home/maxs_331_submit/asap/dockertars:/asap/dockertars \
        --volume /home/maxs_331_submit/deploy:/deploy \
        --volume /home/maxs_331_submit/op_deploy:/op_deploy \
        --volume /usr/bin/sshpass:/usr/bin/sshpass \
        --volume /usr/bin/docker:/usr/bin/docker \
        --volume /home/maxs_331_submit/asap/mysql:/asap/mysql \
        --volume /bin/python2.7:/bin/python \
        --volume /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
        --volume /opt/apache-maven-3.8.6:/usr/local/maven \
        --volume /home/opt/jenkens:/var/jenkins_home \
        --env=JAVA_OPTS=-Duser.timezone=Asia/Shanghai \
        -p 10111:50000 \
        -p 10110:8080 \
        --restart=unless-stopped \
        --runtime=runc \
        --detach=true \
        -t \
        jenkins/jenkins:2.346.3
```
