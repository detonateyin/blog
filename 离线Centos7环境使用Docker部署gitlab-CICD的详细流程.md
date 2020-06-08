
- [需求](#需求)
- [环境](#环境)
- [实施方案](#实施方案)
  - [方案一](#方案一)
    - [结果](#结果)
  - [方案二](#方案二)
    - [结果](#结果-1)
- [一、安装docker](#一安装docker)
    - [准备环境](#准备环境)
    - [安装](#安装)
    - [启动docker](#启动docker)
- [二、安装gitlab-ce](#二安装gitlab-ce)
    - [准备环境](#准备环境-1)
    - [安装](#安装-1)
    - [环境配置](#环境配置)
    - [gitlab配置](#gitlab配置)
      - [1. 修改/mnt/gitlab/etc/gitlab.rb](#1-修改mntgitlabetcgitlabrb)
      - [2. 修改/mnt/gitlab/data/gitlab-rails/etc/gitlab.yml](#2-修改mntgitlabdatagitlab-railsetcgitlabyml)
    - [启动gitlab](#启动gitlab)
- [三、安装gitlab-runner](#三安装gitlab-runner)
    - [准备环境](#准备环境-2)
    - [配置](#配置)
    - [启动gitlab-runner](#启动gitlab-runner)
    - [注册](#注册)
    - [测试](#测试)
    - [坑点](#坑点)
- [备份](#备份)
    - [手动备份](#手动备份)
    - [自动备份](#自动备份)
    - [坑点](#坑点-1)
    - [双机备份](#双机备份)
- [同步](#同步)
- [结语](#结语)
- [参考链接](#参考链接)



## 需求
公司在内网环境下，需要自建一套gitlab的CI/CD环境

## 环境
windowsServer2008R
Centos 7.5.1804

## 实施方案
### 方案一

在windowServer2008R上安装centos的虚拟机，并用桥接的方式使得虚拟机能够接入网内
- 使用Hyper-V或vmware安装centos7.5
- centos7.5中安装docker
- docker中运行gitlab、gitlab-runner

#### 结果
gitlab非常占用资源，官网推荐至少有4G的内存用来运行，winServer服务器本身要跑很多JAVA服务，所以我们抠抠搜搜的分配了4G内存给虚拟机，结果是只是跑了docker-gitlab，虚拟机非常容易出现卡死状态，所以我放弃了此方案，虚拟机上只保留gitlab-runner，以及用以为gitlab储存数据备份

此方案除了虚拟机安装、网络处理、共享挂载等方面需要单独处理外，与方案二无异

### 方案二
- 直接使用linux服务器centos系统中安装docker
- docker运行gitlab、gitlabrunner

#### 结果
此方案相关资料较多，上手比较简单，不涉及虚拟机的性能、网络、挂载等等问题，有条件上服务器就上服务器，简单可靠
使用docker的考虑是基于内网环境情况下，虽然内网docker环境的安装有些许繁琐，但能根本上解决环境问题，通过制作镜像能更方便移植和维护
目前运行良好


## 一、安装docker

#### 准备环境

由于是内网环境，离线安装docker相对来说较为麻烦，需要下载多个rpm包进行安装和更新，这里把需要使用的包列出来。
在http://mirrors.163.com/centos/7/os/x86_64/Packages/中下载如下rpm安装包：

    audit-libs-python-2.8.5-4.el7.x86_64.rpm
    checkpolicy-2.5-8.el7.x86_64.rpm
    libcgroup-0.41-21.el7.x86_64.rpm
    libsemanage-python-2.5-14.el7.x86_64.rpm
    libtool-ltdl-2.4.2-22.el7_3.x86_64.rpm
    policycoreutils-python-2.5-33.el7.x86_64.rpm
    python-IPy-0.75-6.el7.noarch.rpm
    setools-libs-3.3.8-4.el7.x86_64.rpm

    audit-2.8.5-4.el7.x86_64.rpm
    audit-libs-2.8.5-4.el7.x86_64.rpm
    libselinux-2.5-14.1.el7.x86_64.rpm
    libselinux-python-2.5-14.1.el7.x86_64.rpm
    libselinux-utils-2.5-14.1.el7.x86_64.rpm
    libsemanage-2.5-14.el7.x86_64.rpm
    libsepol-2.5-10.el7.x86_64.rpm
    policycoreutils-2.5-33.el7.x86_64.rpm
    
在https://download.docker.com/linux/centos/7/x86_64/stable/Packages/中下载docker的安装包：

    docker-ce-17.12.0.ce-1.el7.centos.x86_64.rpm
    
在http://rpm.pbone.net/index.php3/stat/4/idpl/36266349/dir/scientific_linux_7/com/container-selinux-2.9-4.el7.noarch.rpm.html中下载container-selinux安装包：

    container-selinux-2.9-4.el7.noarch.rpm

这里没有按照帖子中的方法将其分为需要update的包、需要install的包分别更新和安装，因为在实际调试中发现，包的安装和更新，会有相互依赖前置，导致更新或者安装失败，我这里没有过多尝试和研究，简单直接使用无差别强制安装来达到目的，还请这方面有经验的同学帮忙解答。

#### 安装
先将下载好的依赖打成一个包

    tar cf docker-ce.offline.tar *.rpm
    
然后将压缩包拷贝到内网环境下解压安装

    tar xf docker-ce.offline.tar // 解压
    
    sudo rpm -ivh --force  *.rpm 安装 
    sudo rpm -Uvh *.rpm --force  *.rpm 更新

这里使用强制安装--force来覆盖安装，如果提示缺依赖，则传上去安装即可

#### 启动docker

```
systemctl start docker 
systemctl enable docker // 设置开机启动
docker version // 验证docker安装是否成功
```
出现以下信息，表示docker安装成功
```
Client:
 Version:	17.12.0-ce
 API version:	1.35
 Go version:	go1.9.2
 Git commit:	c97c6d6
 Built:	Wed Dec 27 20:10:14 2017
 OS/Arch:	linux/amd64
```


## 二、安装gitlab-ce

#### 准备环境
我们先再互联网机拉取镜像，这里一并拉取了gitlab-runner的镜像

```
docker pull gitlab/gitlab-ce
docker pull gitlab/gitlab-runner
docker images
```
这里没有限制版本，我这里的gitlab的是12.10.3的版本

```
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-runner   latest              f726de7cf9ee        11 days ago         443MB
gitlab/gitlab-ce       latest              f2e48729e35c        3 weeks ago         2GB
```
#### 安装
然后我们需要将镜像保存下来，上传到内网机中

```
// 互联网机
docker save gitlab的IMAGE ID > gitlab.tar
docker save gitlab-runner的IMAGE ID > gitlab-runner.tar

// 内网机
docker load < gitlab.tar
docker load < gitlab-runner.tar
docker images
```
显示出镜像列表，则导入成功
```
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-runner   latest              f726de7cf9ee        11 days ago         443MB
gitlab/gitlab-ce       latest              f2e48729e35c        3 weeks ago         2GB
```
>PS:如果出现镜像名称为none的情况，可以用docker tag imageid name:tag命令给镜像重命名


#### 环境配置
为了便于升级和备份，在gitlab的配置放到容器外，在/mnt/gitlab目录下准备三个文件夹
```
mkdir -p /mnt/gitlab/etc
mkdir -p /mnt/gitlab/log
mkdir -p /mnt/gitlab/data
```
然后在/gitlab文件夹下编写gitlab的运行脚本start.sh
```
#!/bin/sh
GITLAB_HOME=/mnt/gitlab     # 建立gitlab本地目录
docker stop gitlab           # 停止之前gitlab容器
docker rm gitlab             # 删除之前gitlab容器
docker run \
--detach \
--hostname XXX.XXX.XXX.XXX `# 内网没有配置gitlab域名，所以直接使用服务器IP访问`\
-p 8443:443 `# 容器443端口映射到主机8443端口(https)`\
-p 8090:80 `# 容器80端口映射到主机8090端口(http)`\
-p 2222:22 `# 容器22端口映射到主机2222端口(ssh)`\
--name gitlab `# 容器名称`\
--restart unless-stopped `# 容器自动重启`\
-v $GITLAB_HOME/etc:/etc/gitlab `# 挂载本地目录到容器配置目录`\
-v $GITLAB_HOME/log:/var/log/gitlab `# 挂载本地目录到容器日志目录`\
-v $GITLAB_HOME/data:/var/opt/gitlab `# 挂载本地目录到容器数据目录`\
gitlab/gitlab-ce                       # 使用的镜像:版本
```


#### gitlab配置
##### 1. 修改/mnt/gitlab/etc/gitlab.rb

```
vim /mnt/gitlab/etc/gitlab.rb

external_url 'http://XXX.XXX.XXX.XXX:8090'  # 把external_url改成部署机器的域名或者IP地址
gitlab_rails['gitlab_shell_ssh_port'] = 2222 # 由于设置了端口映射把ssh的端口改为2222，与容器配置一致，使得clone时获得正常可用的ssh地址

```
##### 2. 修改/mnt/gitlab/data/gitlab-rails/etc/gitlab.yml
```
vim /mnt/gitlab/data/gitlab-rails/etc/gitlab.yml
```

找到关键字 ` ## Web server settings `
将host的值改成映射的外部主机ip地址和端口，这里会显示在gitlab克隆地址
```
## Web server settings
host XXX.XXX.XXX.XXX
port 8090
```

#### 启动gitlab
运行脚本
```
sh start.sh
docker ps
```
出现gitlab的运行信息则表明运行成功
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                 PORTS                                                 NAMES
993236646a3e        gitlab/gitlab-ce    "/assets/wrapper"   11 days ago         Up 2 hours (healthy)   22/tcp, 0.0.0.0:8090->80/tcp, 0.0.0.0:8443->443/tcp   gitlab

```

浏览器进入`http://gitlab的IP地址:8090`，使用root账户登录,会提示需要设置root密码，之后用root及密码登录即可进入管理员界面


## 三、安装gitlab-runner
有看到帖子说，最好不要把runner和gitlab装在同一台机器上，具体的原因没有细说，不过我们之后会根据服务器资源配置多台runner，所以这里就不考虑，先在docker完成CI/CD设施建设，其他环境的具体安装和注册方式大同小异

#### 准备环境
上一步已经把`gitlab-runner`的镜像加载了，我们这一步就省去说明步骤。

#### 配置

还是在`/mnt/gitlab-runner`下准备一个配置文件夹
```
mkdir -p /mnt/gitlab-runner/etc
```
在`/gitlab-runner`下准备gitlab-runner的脚本start.sh

```
$ GITLAB_HOME = /mnt/gitlab-runner     # 建立gitlab-runner本地目录
$ docker stop gitlab-runner           # 停止之前gitlab容器
$ docker rm gitlab-runner             # 删除之前gitlab容器
$ docker run -d \
--restart unless-stopped  \             # 容器自动重启
-v $GITLAB_HOME/etc:/etc/gitlab-runner \        # 挂载本地目录到容器配置目录
gitlab-runner                      # 使用的镜像:版本
```

#### 启动gitlab-runner

执行`start.sh`将gitlab-runner容器启动
```
sh start.sh
docker ps
```
这时，我们可以看到gitlab和gitlab-runner的容器均跑起来了


#### 注册
然后我们需要注册runner，我们先注册一个shared Runner测试，流程都是一样的
首先打开gitlab控制台，然后在`Admin Area > Overview > Runners` 中找到`Set up a shared Runner manually`，复制url地址和token，接下来runner注册要用，然后在控制台输入命令开始注册
```
docker exec -it gitlab-runner bash #进入容器bash
gitlab-runner register #注册runner

# 输入 GitLab 地址
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://XXX.XXX.XXX.XXX:8090/

# 输入 GitLab Token
Please enter the gitlab-ci token for this runner:
1Lxq_f1NRfCfeNbE5WRh

# 输入 Runner 的说明
Please enter the gitlab-ci description for this runner:
可以为空

# 设置 Tag，可以用于指定在构建规定的 tag 时触发 ci
Please enter the gitlab-ci tags for this runner (comma separated):
shared-runner

# 这里选择 true ，可以用于代码上传后直接执行
Whether to run untagged builds [true/false]:
true

# 这里选择 false，可以直接回车，默认为 false
Whether to lock Runner to current project [true/false]:
false

# 选择 runner 执行器，这里我们选择的是 shell
Please enter the executor: virtualbox, docker+machine, parallels, shell, ssh, docker-ssh+machine, kubernetes, docker, docker-ssh:
shell

```

#### 测试
此时，我们在项目中增加`.gitlab-ci.yml`的脚本，用刚才注册的`shared-runner`这个`tag`即可测试CI/CD
```
stages:
  - install_deps
  - test
  - build
  - deploy_test
  - deploy_production

cache:
  key: ${CI_BUILD_REF_NAME}
  paths:
    - node_modules/
    - dist/

# 安装依赖
install_deps:
  stage: install_deps
  only:
    # - develop
    - master
  script:
    # - npm install
    - echo '模拟安装依赖阶段'
  tags:
    - shared-runner

# 运行测试用例
test:
  stage: test
  only:
    # - develop
    - master
  script:
    # - npm run test
    - echo '模拟运行测试用例阶段'
  tags:
    - shared-runner
# 编译
build:
  stage: build
  only:
    # - develop
    - master
  script:
    # - npm run clean
    # - npm run build:client
    # - npm run build:server
    - echo '模拟编译阶段'
  tags:
    - shared-runner
    
# 部署测试服务器
deploy_test:
  stage: deploy_test
  only:
    # - develop
    - master
  script:
    # - pm2 delete app || true
    # - pm2 start app.js --name app
    - echo '模拟部署测试服务器阶段'
  tags:
    - shared-runner

# 部署生产服务器
deploy_production:
  stage: deploy_production
  only:
    # - develop
    - master
  script:
    # - bash scripts/deploy/deploy.sh
    - echo '模拟部署生产服务器阶段'
  tags:
    - shared-runner
    
```

然后提交到git库中，我们就可以在项目代码库中的CI/CD的位置查看是否成功，`status`为`passed`代表成功，点击`status`即可进入`pipeline`查看每个`stage`的详情和日志，具体截图我这里就不粘了

#### 坑点
> `fatal: unable to access 'http://……`

在`pipeline`的日志中，如果出现runner无法拉取git库，很有可能是runner的clone地址不正确，我们在配置文件中将地址和端口号写上即可
我们需要修改`/mnt/gitlab-runner/etc/config.toml`
找到`[[runners]]`，添加`clone_url`以解决runner无法拉取git库代码的问题

```
clone_url = "http://XXX.XXX.XXX.XXX:8090/"
```


## 备份
GitLab默认备份目录为`/var/opt/gitlab/backups`，可以修改`/etc/gitlab/gitlab.rb`里面的默认存放备份文件目录，这里使用默认备份目录
```
gitlab_rails['backup_path'] = '/var/opt/gitlab/backups'
```

#### 手动备份
```
//在容器里面
gitlab-rake gitlab:backup:create

//在宿主机里面
docker exec -it gitlab gitlab-rake gitlab:backup:create
```

可以看到我们映射目录`/mnt/gitlab/data/backups/`中新增了一个备份包

#### 自动备份

我们先在`/mnt/gitlab/data/backups/`中添加一个自动备份的执行脚本`auto_backup.sh`

```
docker exec gitlab gitlab-rake gitlab:backup:create
```
然后我们手动执行一下测试脚本是否编写正确
```
sh auto_backup.sh
```

然后通过crontab服务来添加定时任务，实现自动备份，我们先配置crontab，增加备份定时任务

```
crontab -e
```

```
# 每天23点执行一次自动备份脚本
0  23   * * *   /mnt/gitlab/data/backups/auto_backup.sh
```

然后，需要重新启动cron服务
```
# 重新加载cron配置文件
systemctl reload crontab 
# 重启cron服务
systemctl restart crontab
```
定时任务执行之后之后可以在备份目录下，看到自动备份包

可以在`/var/log/cron`查看定时任务执行日志
```
tail -300 /var/log/cron
```

#### 坑点

> 注意自动备份脚本文件中备份命令不能有`-it`命令，原因是`exec `后面加了 `-it` 参数就开启了一个终端，计划任务是无法进入任何终端的，会导致命令无效

> 注意权限的问题，如果发现定时任务执行了无效果，可以去看看`/var/spool/mail/root`的日志，我操作的时候发现很诡异的情况就是`root`居然没有`auto_backup.sh`执行权限，改成755就好了

> 如果发现文件权限没有问题，那么很有可能是`selinux`的问题，去到`/ect/selinux/config`中看`SELINUX`的配置，把它设置为`disabled`即可。这个比较坑，会导致`root`没有权限


#### 双机备份

定时通过`scp`向linux服务器拷贝我们的备份文件，以防止我们的服务器出现问题，能够通过找回代码资产

首先需要向远程服务器传递我们的ssh公钥，如果之前没有生成秘钥，则执行

```
ssh-keygen -t rsa
```
一路
生成过秘钥，命令行会提示是否覆盖，一路回车就行

然后在`~/.ssh`目录下找到公钥文件`id_ras.pub`文件，上传到远程服务器
```
scp id_rsa.pub 服务器登录用户名(root)@XXX.XXX.XXX.XXX(服务器地址):/tmp/id_rsa.pub.gitlab
```

然后使用同一用户(这里用root)，登录远程服务器，上找到`~/.ssh`目录(为方便使用root用户目录，其他用户则需要到`/home/XXX用户名/.ssh`)，如果没有就创建一个，或者用`ssh-keygen -t rsa`生成秘钥文件，建议使用`ssh-keygen`命令

找到`~/.ssh`目录下的`authorized_keys`文件，如果没有就创建一个

```
# 创建authorized_keys文件
touch authorized_keys
```

然后将`/tmp/id_rsa.pub.gitlab`追加到`authorized_keys`中

```
cat id_rsa.pub.gitlab >> ~/.ssh/authorized_keys
```

然后回到gitlab服务器，用ssh登录远程备份服务器，测试公钥是否配对成功

```
ssh 用户名(这里用root)@远程服务器地址
```
没有提示输入密码，则成功

如果失败，则检查拷贝过程是否成功，查看`authorized_keys`中追加的公钥是否与gitlab服务器的公钥一致，然后通过调试信息查看问题
```
ssh -vvv 用户名(这里用root)@远程服务器地址
```

>ssh对目录的权限有要求，需注意`~/.ssh`的是700， `~/.ssh/* `的是600


先在`/mnt/gitlab/data/backups/`中添加一个自动备份的执行脚本`auto_scp.sh`

```
#!/bin/bash

# gitlab备份文件存放路径 
BACKUPDIR=/mnt/gitlab/data/backups

# 远程备份服务器 登录账户（这里为了方便就直接用root）
RemoteUser=root

# 远程备份服务器 IP地址 
RemoteIP=XXX.XXX.XXX.XXX

#当前系统日期 
DATE=`date "+%Y-%m-%d-%H:%M:%S"`

#Log存放路径
LogFile=$BACKUPDIR/log/$DATE.log

#查找本地备份目录下时间为1天之内并且后缀为.tar的gitlab备份文件
BACKUPFILE_SEND_TO_REMOTE=$(find $BACKUPDIR -type f -mmin -1440 -name '*.tar')

#新建日志文件
touch $LogFile

echo "[$DATE] 远程备份gitlab" >> $LogFile

#输出日志，打印出远程服务器地址
echo "[$DATE] 远程服务器地址：$RemoteUser@$RemoteIP:$BACKUPDIR" >> $LogFile

#输出日志，打印出每次scp的文件名
echo "[$DATE] 本次备份文件: $BACKUPFILE_SEND_TO_REMOTE" >> $LogFile

#备份到远程服务器
scp $BACKUPFILE_SEND_TO_REMOTE $RemoteUser@$RemoteIP:$BACKUPDIR

```

手动执行一下`auto_scp.sh`，能看到Log文件夹下有备份日志，去到远程服务器备份路径查看备份文件

然后，在crontab中增加定时任务

```
# 每天23点30执行一次自动备份脚本
30  23   * * *   /mnt/gitlab/data/backups/auto_scp.sh
```

之后还可以做一些`备份文件自动删除`、`gitlab自动恢复`等定时操作，具体的可以查看参考文献中的步骤，这里就不赘述了

## 同步

我们的需求是，本地有一个库，远端也有一个库，我们分开单独管理，但本地库代码更新需要推送给远端

项目库页面，在左侧菜单栏选择`Settings->Repository`
选择`Push to a remote repository`,勾选`Remote mirror repository`,填写`Git repository URL`

这里的url需要携带账号密码信息
```
http://[userName]:[password]@XXXX.git
```

然后提交一次代码测试一下，如果有错误信息，则会在statu的tip弹窗中显示


## 结语
感谢各位参考文章中的作者，为搭建提供了很多参考和帮助，这里把本次搭建的整体流程整理出来，有不对的地方，还请各位大神指正

## 参考链接
- [CentOS7离线部署docker](https://juejin.im/post/5d47d0a76fb9a06afa326d62)
- [docker安装配置gitlab详细过程](https://www.cnblogs.com/zuxing/articles/9329152.html)
- [gitlab,gitlab runner自动化部署docker容器](https://blog.51cto.com/11750513/2422946)
- [gitlab自动备份和定时删除](https://www.cnblogs.com/linyouyi/p/10753569.html)
- [GitLab从安装到全自动化备份一条龙](https://juejin.im/post/5ce6519fe51d454fbf54095b)