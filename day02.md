# day02

[toc]

## gitlab

- 它是一个开源的git仓库服务器。用于实现代码集中托管。
- 分为企业版和CE社区版。
- 部署方式：软件包部署、容器部署。

### 通过容器部署gitlab服务器

- 将虚拟机192.168.4.20作为gitlab服务器。它需要4GB以上内存。
- 将/linux-soft/2/gitlab_zh.tar拷贝到192.168.4.20

```shell
[root@zzgrhel8 ~]# scp /linux-soft/2/gitlab_zh.tar 192.168.4.20:/root
```

- 部署gitlab容器

```shell
# 安装容器管理软件podman
[root@git ~]# yum install -y podman

# 修改192.168.4.20的ssh端口号。因为gitlab容器也要用到22端口，有冲突。
# +17是打开文件时，光标直接定位到第17行。
[root@git ~]# vim +17 /etc/ssh/sshd_config 
 17  Port 2022
[root@git ~]# systemctl restart sshd
# ssh连接退出，再登陆时，需要指定端口号
[root@zzgrhel8 ~]# ssh -p2022 192.168.4.20

# 导入镜像。一个镜像可以创建很多容器。镜像是只读的，容器是可以改变的。
# 容器相当于是精简的虚拟机，可以像虚拟机一样，对外提供服务。
[root@git ~]# podman load < gitlab_zh.tar

# 查看导入的镜像
[root@git ~]# podman images
REPOSITORY            TAG      IMAGE ID       CREATED       SIZE
localhost/gitlab_zh   latest   1f71f185271a   3 years ago   1.73 GB

# 容器如果出现故障，首先的排错方法是重启它；如果无效，删掉重建。
# 为了删容器，不丢失数据，需要把容器需要的数据保存在宿主机上。在哪台主机上启动容器，哪台主机就是宿主机。
# 在192.168.4.20上创建用于保存容器数据的目录
[root@git ~]# mkdir -p /srv/gitlab/{config,logs,data}
[root@git ~]# ls /srv/gitlab/
config  data  logs

# gitlab容器需要/etc/resolv.conf文件。不存在则创建它
[root@git ~]# touch /etc/resolv.conf

# 创建容器
# -d后台运行。-h gitlab设置容器的主机名。--name gitlab是podman ps查看到的容器名；-p指定发布的端口号，当访问宿主机443/80/22端口时，这样的请求就发给容器的相关端口；--restart always是开机自启；-v是映射路径，将容器中指定的路径，映射到宿主机，以便保存容器产生的数据；最后的gitlab_zh是镜像名。
[root@git ~]# podman run -d -h gitlab --name gitlab -p 443:443 -p 80:80 -p 22:22 --restart always -v /srv/gitlab/config:/etc/gitlab -v /srv/gitlab/logs:/var/log/gitlab -v /srv/gitlab/data:/var/opt/gitlab gitlab_zh
# 查看容器
[root@git ~]# podman ps
# 如果一切正常，几分钟后，可以访问http://192.168.4.20/
```

> 附：如果容启动失败，再次创建有以下错误：
>
> ```shell
> Error: error creating container storage: the container name "gitlab" is already in use by "ca425e33d7ff2d282cbec1033023851cff285fe9b819ed50d47a08a875372fde". You have to remove that container to be able to reuse that name.: that name is already in use
> ```
>
> 则：
>
> ```shell
> [root@git ~]# podman rm gitlab
> [root@git ~]# podman run -d -h gitlab --name gitlab -p 443:443 -p 80:80 -p 22:22 --restart always -v /srv/gitlab/config:/etc/gitlab -v /srv/gitlab/logs:/var/log/gitlab -v /srv/gitlab/data:/var/opt/gitlab gitlab_zh
> ```

### 配置gitlab

- 第一次登陆时，要求改密码。密码需要是复杂密码，如`1234.com`。修改之后，登陆的用户名是root。

- 改变显示配置。

![image-20211025112251219](../imgs/image-20211025112251219.png)

![image-20211025112429859](../imgs/image-20211025112429859.png)

![image-20211025112525416](../imgs/image-20211025112525416.png)

![image-20211123115027081](../imgs/image-20211123115027081.png)

点击下面的保存后，LOGO图标将会改变。退出后，登陆界面也会有变化。

![image-20211123115101885](../imgs/image-20211123115101885.png)

#### gitlab中主要的概念

- 用户：为使用gitlab的用户创建的账号。
- 组：用户的集合。一般可以为部门创建组。将来可以在项目上为组授权，组 中所有的用户都会得到相应的权限。
- 项目：用于保存代码文件的空间。

- 创建用户

![image-20211025114636003](../imgs/image-20211025114636003.png)

![image-20211025114735216](../imgs/image-20211025114735216.png)

填写截图上的几项后，其他使用默认配置，点保存。

创建好用户后，点击编辑，可以为他/她设置密码：

![image-20211025115000568](../imgs/image-20211025115000568.png)

![image-20211025115043634](../imgs/image-20211025115043634.png)

保存修改后，退出当前账号，使用新账号登陆测试。第一次登陆时，也是要求修改密码，新密码可以设置与旧密码一样。新建的jerry用户因为权限较小，所以看到的界面，没有root的功能多。

> 容器的删除
>
> ```shell
> # 查看容器的名字和ID号，删除时，使用名字或ID号均可
> [root@git ~]# podman ps
> # 强制删除容器
> [root@git ~]# podman rm -f gitlab
> # 新建容器
> [root@git ~]# podman run -d -h gitlab --name gitlab -p 443:443 -p 80:80 -p 22:22 --restart always -v /srv/gitlab/config:/etc/gitlab -v /srv/gitlab/logs:/var/log/gitlab -v /srv/gitlab/data:/var/opt/gitlab gitlab_zh
> ```

- 创建组。注意，需要使用root账号

![image-20211025140457129](../imgs/image-20211025140457129.png)

![image-20211025140547961](../imgs/image-20211025140547961.png)

点击“创建群组”

- 将jerry加到devops组中，角色是“主程序员”

![image-20211025140816833](../imgs/image-20211025140816833.png)

- 创建项目

![image-20211025141232784](../imgs/image-20211025141232784.png)

![image-20211025141434398](../imgs/image-20211025141434398.png)

## 客户端上传代码到gitlab服务器

### 查看项目路径，采用http方式上传

- 查看项目说明

![image-20211025142444187](../imgs/image-20211025142444187.png)

![image-20211025142530464](../imgs/image-20211025142530464.png)

- 切换jerry用户，设置jerry的密码。

![image-20211123150849727](../imgs/image-20211123150849727.png)

![image-20211123150919805](../imgs/image-20211123150919805.png)

![image-20211123150953791](../imgs/image-20211123150953791.png)

![image-20211123151222139](../imgs/image-20211123151222139.png)

![image-20211123151032761](../imgs/image-20211123151032761.png)

- 在客户端192.168.4.10上下载项目，编写代码并上传

```shell
[root@develop ~]# git clone http://192.168.4.20/devops/myproject.git
正克隆到 'myproject'...
warning: 您似乎克隆了一个空仓库。
[root@develop ~]# ls    # 本地出现一个myproject目录
anaconda-ks.cfg  myproject

# 创建说明文件并上传。一般来说，git服务器在首页默认可以显示readme文件的内容
[root@develop ~]# cd myproject/
[root@develop myproject]# vim README.md
- 这是我的第1个测试项目
​```
echo 'Hello World!'
​```
[root@develop myproject]# git add .    # 保存到暂存区
[root@develop myproject]# git commit -m "init data"  # 确认到版本库
# 将master分支推送到origin仓库。origin是默认仓库名。
[root@develop myproject]# git push -u origin master
Username for 'http://192.168.4.20': jerry   # 用户名
Password for 'http://jerry@192.168.4.20': 1234.com   # 密码
# 在服务器上刷新web页面


# 将来就可以重得操作：写代码、确认到版本库、上传到服务器
[root@develop myproject]# cp /etc/hosts .
[root@develop myproject]# git add .
[root@develop myproject]# git commit -m "add hosts"
[root@develop myproject]# git push   # 不必再使用-u选项
Username for 'http://192.168.4.20': jerry
Password for 'http://jerry@192.168.4.20': 1234.com


# 模拟另一个客户端同步数据
[root@zzgrhel8 ~]# ssh 192.168.4.10
[root@develop ~]# cd /var/tmp/
[root@develop tmp]# git clone http://192.168.4.20/devops/myproject.git
[root@develop tmp]# ls
myproject
[root@develop tmp]# cd myproject/
[root@develop myproject]# ls
hosts  readme.md

# 在家目录的myproject中上传新文件
[root@develop myproject]# cp /etc/issue .
[root@develop myproject]# ls
hosts  issue  readme.md
[root@develop myproject]# git add .
[root@develop myproject]# git commit -m "add issue"
[root@develop myproject]# git push
Username for 'http://192.168.4.20': jerry
Password for 'http://jerry@192.168.4.20': 1234.com

# 在/tmp/myproject中同步数据
[root@develop myproject]# git pull
[root@develop myproject]# ls
hosts  issue  readme.md
```

### 使用ssh免密推送代码

- 本质上与ssh免密登陆服务器一样。

1. 在客户端192.168.4.10上生成密钥对

```shell
[root@develop myproject]# ssh-keygen   # 三个问题，都直接回车
```

2. 将公钥保存到gitlab服务器

```shell
# 查看并复制公钥内容
[root@develop myproject]# cat ~/.ssh/id_rsa.pub 
```

在gitlab上切换Jerry用户登陆

![image-20211025154017719](../imgs/image-20211025154017719.png)

![image-20211025154051000](../imgs/image-20211025154051000.png)

把jerry的公钥粘贴到密钥框中

![image-20211025154146637](../imgs/image-20211025154146637.png)

3. 将推送代码的方式改为ssh

查看ssh路径

![image-20211025154726675](../imgs/image-20211025154726675.png)

![image-20211025154752012](../imgs/image-20211025154752012.png)

在192.168.4.10上将推送代码的路径改为ssh的方式

```shell
# 查看仓库信息，当前是http方式
[root@develop myproject]# git remote -v
origin	http://192.168.4.20/devops/myproject.git (fetch)
origin	http://192.168.4.20/devops/myproject.git (push)

# 删除http的路径
[root@develop myproject]# git remote remove origin

# 添加ssh路径
[root@develop myproject]# git remote add origin git@192.168.4.20:devops/myproject.git

# 查看修改后的路径
[root@develop myproject]# git remote -v
origin	git@192.168.4.20:devops/myproject.git (fetch)
origin	git@192.168.4.20:devops/myproject.git (push)

# 推送代码测试
[root@develop myproject]# cp /etc/passwd .
[root@develop myproject]# git add .
[root@develop myproject]# git commit -m "add passwd"
[root@develop myproject]# git push -u origin master  # 不再需要密码
[root@develop myproject]# git push 
```

### 巩固练习

1. 在gitlab上新建名为myweb的项目，为devops组创建，可见等级为公开。
2. 将192.168.4.10上的myweb项目关联到gitlab的myweb，以ssh方式关联。
3. 在192.168.4.10上，把myweb目录中文件上传。

```shell
[root@develop ~]# cd myweb/
[root@develop myweb]# git remote add origin git@192.168.4.20:devops/myweb.git
[root@develop myweb]# git push -u origin master
# 在gitlab上查看myweb项目
```



## CI（持续集成）/CD（持续交付）

![image-20211025171518916](../imgs/image-20211025171518916.png)

### 软件程序上线流程

1. 程序员将代码上传到gitlab服务器
2. 云计算工程师，通过jenkins服务器自动下载gitlab上的代码
3. 云计算工程师编写自动部署到服务器上的脚本

### 安装Jenkins服务器

- jenkins的IP地址是：192.168.4.30。它必须能与其他主机通信
- 关闭selinux/防火墙
- 安装jenkins

```shell
# 安装依赖包
# jenkins需要通过git下载代码，所以装git。
# jenkins是java程序，所以装java
# postfix和mailx是邮件程序，jenkins可以通过它们给管理员发邮件
[root@jenkins ~]# yum install -y git postfix mailx java-11-openjdk

# 把jenkins软件包拷贝到192.168.4.30
[root@zzgrhel8 ~]# ls /linux-soft/2/jenkins*
/linux-soft/2/jenkins-2.263.1-1.1.noarch.rpm
/linux-soft/2/jenkins_plugins.tar.gz
[root@zzgrhel8 ~]# scp /linux-soft/2/jenkins* 192.168.4.30:/root/

# 在192.168.4.30上安装jenkins
[root@jenkins ~]# yum install -y jenkins-2.263.1-1.1.noarch.rpm

# 启动服务，并设置为开机自启
[root@jenkins ~]# systemctl enable jenkins
jenkins.service is not a native service, redirecting to systemd-sysv-install.   # 注意：这里不是错误，忽略即可
Executing: /usr/lib/systemd/systemd-sysv-install enable jenkins
[root@jenkins ~]# systemctl start jenkins
```

- 访问http://192.168.4.30:8080，进行初始化

```shell
# 查看初始化密码
[root@jenkins ~]# cat /var/lib/jenkins/secrets/initialAdminPassword
2c58512973be4a44aec3ef5c1463d00a
```

把查看到的密码粘贴到文本框中，如下：

![image-20211025174540336](../imgs/image-20211025174540336.png)

![image-20211025174612766](../imgs/image-20211025174612766.png)

不用创建管理员，使用自带的admin

![image-20211025174714079](../imgs/image-20211025174714079.png)

![image-20211025174822335](../imgs/image-20211025174822335.png)

![image-20211025174846396](../imgs/image-20211025174846396.png)

- 修改admin密码

![image-20211025175013126](../imgs/image-20211025175013126.png)

![image-20211025175112535](../imgs/image-20211025175112535.png)

![image-20211025175213100](../imgs/image-20211025175213100.png)

![image-20211025175303837](../imgs/image-20211025175303837.png)

