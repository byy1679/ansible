# day03

[toc]

## 永久关闭防火墙和selinux

```shell
[root@git ~]# systemctl stop firewalld
[root@git ~]# systemctl disable firewalld
[root@git ~]# setenforce 0
[root@git ~]# vim +7 /etc/selinux/config
  7 SELINUX=disabled
```



## CI/CD

![image-20211025171518916](../imgs/image-20211025171518916.png)

- 启动之前准备好的机器：192.168.4.10、192.168.4.20、192.168.4.30

- 设置gitlab容器开机自启：`/etc/rc.local`是开机后会自动运行的脚本，写到这个文件中的命令，开机后都会自动运行

```shell
[root@git ~]# vim /etc/rc.d/rc.local   # 在文件尾部追加一行内容如下：
... ...
podman start gitlab
[root@git ~]# chmod +x /etc/rc.d/rc.local
```

### 配置jenkins

- 访问http://192.168.4.30:8080，用户名是admin

- 安装插件：jenkins的很多功能都是能过插件实现的，比如发邮件、比如中文支持。

```shell
[root@jenkins ~]# yum install -y tar
[root@jenkins ~]# tar xf jenkins_plugins.tar.gz 
# 拷贝文件的时候，注意选项，-r可以拷贝目录，-p保留权限
[root@jenkins ~]# cp -rp jenkins_plugins/* /var/lib/jenkins/plugins/
[root@jenkins ~]# systemctl restart jenkins
# 刷新web页面，如果出现中文，则插件安装成功
```

### 软件版本管理

- 可以在git中使用tag标记将某一次commit提交标识为某一版本

```shell
[root@develop ~]# cd myproject/    # 进入项目目录
[root@develop myproject]# git tag  # 查看标记，默认没有标记
[root@develop myproject]# git tag 1.0  # 将当前提交，标识为1.0
[root@develop myproject]# git tag
1.0
[root@develop myproject]# echo 'hello world' > index.html
[root@develop myproject]# git add .
[root@develop myproject]# git commit -m "add index.html"
[root@develop myproject]# git tag 1.1

# 将本地文件和tag推送到gitlab服务器
[root@develop myproject]# git push    # 只推送文件，不推送标记
[root@develop myproject]# git push --tags
```

在gitlab上查看标记：

![image-20211026103014032](../imgs/image-20211026103014032.png)

![image-20211026103037105](../imgs/image-20211026103037105.png)

### 配置jenkins访问gitlab代码仓库

![image-20211026104151358](../imgs/image-20211026104151358.png)

![image-20211026104236038](../imgs/image-20211026104236038.png)参数化构建过程中，“名称”是自己定义的变量名，用于标识tag或分支

![image-20211026104432176](../imgs/image-20211026104432176.png)

git仓库地址，在gitlab上找到myproject仓库的http地址，注意将gitlab名称改为IP地址

![image-20211026104830294](../imgs/image-20211026104830294.png)

指定分支构建的时候，使用上面步骤创建的变量`$web`

![image-20211026105126841](../imgs/image-20211026105126841.png)

点击保存。

在项目页面， 可以进行构建测试。

![image-20211026105352177](../imgs/image-20211026105352177.png)

![image-20211026105417219](../imgs/image-20211026105417219.png)

### 测试下载

![image-20211026111903786](../imgs/image-20211026111903786.png)

![image-20211026111940079](../imgs/image-20211026111940079.png)

![image-20211026112029641](../imgs/image-20211026112029641.png)

构建过程中，边栏左下角会有一个闪烁的灰球，构建成功是蓝球，失败是红球。点击它，可以看详情。

![image-20211026112125421](../imgs/image-20211026112125421.png)

![image-20211026112303969](../imgs/image-20211026112303969.png)

![image-20211026112330420](../imgs/image-20211026112330420.png)

在jenkins上查看下载的内容：

```shell
[root@jenkins ~]# ls /var/lib/jenkins/workspace/myproject/
README.md  hosts  passwd
```

### 下载到子目录

- jenkins下载不同的版本到自己的子目录，不共享相同目录

![image-20211026113614654](../imgs/image-20211026113614654.png)

![image-20211026113638980](../imgs/image-20211026113638980.png)

新增时，如果没有中文，英文是“checkout to a sub directory”

![image-20211026113820175](../imgs/image-20211026113820175.png)

![image-20211026113857142](../imgs/image-20211026113857142.png)

点击保存。

测试：

```shell
# 删除之前下载的内容
[root@jenkins ~]# rm -rf /var/lib/jenkins/workspace/myproject/
```

执行多次构建，构建不同版本：

![image-20211026114526435](../imgs/image-20211026114526435.png)

查看下载目录：

```shell
[root@jenkins ~]# ls /var/lib/jenkins/workspace/myproject/
myproject-1.0  myproject-1.1
```

## 准备两台web服务器

- web1：192.168.4.100，配置YUM，关闭SELINUX/防火墙
- web2：192.168.4.200，配置YUM，关闭SELINUX/防火墙

## 部署代码到web服务器

### 自动化部署流程

1. 程序员编写代码，推送到gitlab服务器
2. Jenkins服务器从gitlab上下载代码
3. Jenkins处理下载的代码
   - 删除下载目录的版本库
   - 将下载的代码打包
   - 计算程序压缩包的md5值
   - 在Jenkins上安装ftp服务，共享程序压缩包

4. web服务器下载软件包，并应用（通过脚本实现）
5. 访问测试

#### 在Jenkins上配置FTP服务器

```shell
# 安装vsftpd
[root@jenkins ~]# yum install -y vsftpd

# 启用ftp的匿名访问
[root@jenkins ~]# vim +12 /etc/vsftpd/vsftpd.conf 
anonymous_enable=YES

# 起服务
[root@jenkins ~]# systemctl enable vsftpd --now

# ftp的数据目录默认是/var/ftp。
# 在ftp上创建保存压缩包的路径
[root@jenkins ~]# mkdir -p /var/ftp/deploy/packages
# 因为jenkins服务需要向该目录保存文件，所以设置jenkins对它有权限
[root@jenkins ~]# chown -R :jenkins /var/ftp/deploy
[root@jenkins ~]# chmod -R 775 /var/ftp/deploy/
```

#### 配置jenkins把gitlab下载的代码打包

在jenkins上修改myproject项目

![image-20211026143752366](../imgs/image-20211026143752366.png)

![image-20211026144533155](../imgs/image-20211026144533155.png)

```shell
# 定义存储软件包路径的变量
pkg_dir=/var/ftp/deploy/packages
# 将下载的代码目录拷贝到下载目录
cp -r myproject-$web $pkg_dir
# 删除下载目录的版本库，不是必须的，只是为了严谨
rm -rf $pkg_dir/myproject-$web/.git
cd $pkg_dir   # 切换到下载目录
# 将下载的目录打包
tar czf myproject-$web.tar.gz myproject-$web
# 下载目录已打包，目录就不需要了，删除它
rm -rf myproject-$web
# 计算压缩包的md5值，保存到文件
md5sum myproject-$web.tar.gz | awk '{print $1}' > myproject-$web.tar.gz.md5
cd ..
echo -n $web > ver.txt   # 将版本号写入文件
```

以上步骤改好后，保存。

测试修改的任务。

![image-20211026151614872](../imgs/image-20211026151614872.png)

![image-20211026151719555](../imgs/image-20211026151719555.png)

## web服务自动部署

### 安装httpd服务

```shell
[root@web1 ~]# yum install -y httpd tar wget
[root@web1 ~]# systemctl enable httpd --now
[root@web1 ~]# ss -tlnp | grep :80
LISTEN    0         128                      *:80                     *:*        users:(("httpd",pid=9721,fd=4),("httpd",pid=9720,fd=4),("httpd",pid=9719,fd=4),("httpd",pid=9717,fd=4))
```

### 编写自动上线脚本

- 下载软件包
- 检查软件包是否损坏
- 解压、部署到web服务器

```shell
[root@web1 ~]# vim /usr/local/bin/web.sh
#!/bin/bash

# 定义软件包服务器和本地路径
ftp_url=ftp://192.168.4.30/deploy
deploy_dir=/var/www/deploy
dest=/var/www/html/tedu-cloud

# 创建用于部署的函数
down_file(){
        # 获取要下载的软件版本
        version=$(curl -s $ftp_url/ver.txt)
        # 下载版本文件到本地
        wget -q $ftp_url/ver.txt -O $deploy_dir/ver.txt
        # 下载软件压缩包
        wget -q $ftp_url/packages/myproject-$version.tar.gz -O $deploy_dir/myproject-$version.tar.gz
        # 计算本地压缩包Md5值
        hash=$(md5sum $deploy_dir/myproject-$version.tar.gz | awk '{print $1}')
        # 获取网上md5文件中的md5值
        ftp_hash=$(curl -s $ftp_url/packages/myproject-$version.tar.gz.md5)
        # 如果文件未损坏则解压
        if [ "$hash" == "$ftp_hash" ]; then
            tar xf $deploy_dir/myproject-$version.tar.gz -C $deploy_dir
        else
            rm -f $deploy_dir/myproject-$version.tar.gz
        fi
        # 如果存在目标软链接，先删除它
        if [ -e "$dest" ]; then
        	rm -f $dest
        fi
        # 创建软链接
        ln -s $deploy_dir/myproject-$version $dest
}

# 如果$deploy_dir不存在，先创建它
if [ ! -e "$deploy_dir" ]; then
	mkdir $deploy_dir
fi

# 如果本地不存在版本文件，则意味着是新服务器，要部署软件
if [ ! -f $deploy_dir/ver.txt ]; then
    down_file
fi

# 如果本地存在版本文件，但是和服务器上的版本文件不一样，则要部署新版本
if [ -f $deploy_dir/ver.txt ]; then
        ftp_ver=$(curl -s $ftp_url/ver.txt)
        local_ver=$(cat $deploy_dir/ver.txt)
        if [ "$ftp_ver" != "$local_ver" ]; then
            down_file
        fi
fi
[root@web1 ~]# chmod +x /usr/local/bin/web.sh
[root@web1 ~]# yum install -y wget
[root@web1 ~]# web.sh 
# 访问http://192.168.4.100/tedu-cloud/可以看到部署的文件
[root@web1 html]# ls /var/www/html/
tedu-cloud
```

- 完整测试流程：
  - 程序员编写新版本并推送到服务器
  - Jenkins上构建新版本
  - web服务器上执行`web.sh`部署新版本

```shell
# 程序员编写新版本
[root@develop myproject]# vim index.html 
<marquee>Welcome to tedu</marquee>
[root@develop myproject]# git add .
[root@develop myproject]# git commit -m "modify index.html"
[root@develop myproject]# git tag 2.0
# 程序员推送到服务器
[root@develop myproject]# git push
[root@develop myproject]# git push --tags
```

![image-20211026172611104](../imgs/image-20211026172611104.png)

```shell
# web服务器上执行`web.sh`部署新版本
[root@web1 html]# web.sh 
[root@web1 html]# ls /var/www/deploy/
myproject-1.1         myproject-2.0         ver.txt
myproject-1.1.tar.gz  myproject-2.0.tar.gz
# 访问http://192.168.4.100/tedu-cloud
```

