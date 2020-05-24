# React+Tornado+MySQL+Nginx+Supervisor 部署方案

## 为什么要记录分享？

- 我做了一个个人的小网站，第一次接触 Centos 这方面的部署，也是独立部署，遇到了非常多的问题，网上的资料五花八门，但是能用的没多少。
- 通过摸索思考，渐渐的理清了很多部署的思路，重装了 N 次服务器，最后终于成功部署。
- 希望能借文档记录一下这次经历。

## 主要环境

- 阿里云-Centos8
- Python3.8
- MySQL8-数据库

## 项目构造

- React-前端
- Tornado-Python 开发后端接口

## 部署工具

- Nginx-部署 React 及反向代理 Tornado
- Supervisor-部署 Tornado

## 远程工具（Mac）

- FileZilla-类似 FTP 工具，方便查看服务器目录结构及操作文件
- zoc7-SSH 连接服务器（阿里云有提供终端，也可在阿里云上操作）

**_PS：工具这是个人日常用的，如果有更好的欢迎使用哈～_**

---

## No.1

### **_升级 Python_**

- 默认版本

```
[root@Server ~]# python3 -V
Python 3.6.8
```

- 这里建议新建一个文件夹专门放置下载的文件

```
[root@Server ~]# mkdir /Downloads
[root@Server /]# ls
bin  boot  dev  Downloads  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

可见已新建一个“Downloads”文件夹

- 下载 Python3.8.1（切换到 Downloads 文件夹下）

```
[root@Server /]# cd /Downloads/
[root@Server Downloads]# wget https://www.python.org/ftp/python/3.8.1/Python-3.8.1.tgz
```

下载的速度稍微会有点慢，耐心等待一下就好～

- 安装编译 Python 所需要的包

```
[root@Server Downloads]# yum -y install gcc zlib* libffi-devel openssl-devel
```

- 解压包

```
[root@Server Downloads]# tar -zxvf Python-3.8.1.tgz
```

- 进入解压后的文件夹，设置 python 安装位置

```
[root@Server Downloads]# cd Python-3.8.1/
[root@Server Python-3.8.1]# ./configure --prefix=/usr/local/bin/python3
```

- 编译&安装

```
[root@Server Python-3.8.1]# make
[root@Server Python-3.8.1]# make install
```

- 将系统 python 版本改为 3.8.1，创建一个软连接

```
[root@Server ~]# rm -rf /usr/bin/python3
[root@Server ~]# ln -s /usr/local/bin/python3/bin/python3 /usr/bin/python3
[root@Server ~]# python3 -V
Python 3.8.1
```

- 当然，pip 也需要改，否则 pip 下载的东西还是会默认存到原来 python 的文件夹下

```
[root@Server ~]# rm -rf /usr/bin/pip3
[root@Server ~]# ln -s /usr/local/bin/python3/bin/pip3 /usr/bin/pip3
[root@Server ~]# pip3 -V
pip 19.2.3 from /usr/local/bin/python3/lib/python3.8/site-packages/pip (python 3.8)
```

这里显示 pip 是 python3.8 下面的 pip 就代表设置成功了

- 到这里，python 安装就完成了

## No.2

### **_安装&配置 Nginx_**

- 安装

```
yum install nginx
```

- 将 React 生成的文件夹放入服务器中 home 文件夹，可以用上面提到的 FileZilla 软件上传一下（当然可以选择其他文件夹，个人喜好）

- 配置首页

  - 切换到 nginx 目录下

  ```
  [root@Server ~]# cd /etc/nginx/
  ```

  - 编辑 nginx.conf 配置文件

  ```
   ...
   server {
        ...
        # 根目录为 React 的 build 文件夹
	    root /home/build/;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            # 首页为 build 内的 index.html
	        index  index.html; 
        }
        ...
   }
   ...
  ```

  - 运行测试

  ```
  [root@Server ~]# nginx
  ```

  访问你的域名，这时就能看到 React 的内容了。（如果不行，检查一下服务器 80 端口开了没。）

## No.3

### **_安装&配置 Supervisor_**

- 安装

```
[root@Server ~]# yum install -y supervisor
```

- 开机自启

```
[root@Server ~]# systemctl enable supervisord
Created symlink /etc/systemd/system/multi-user.target.wants/supervisord.service → /usr/lib/systemd/system/supervisord.service.
```

- 启动服务

```
[root@Server ~]# systemctl start supervisord
```

- 配置 Tornado 启动

  - 切换到 supervisord.d 文件夹（存放 supervisor 额外配置文件）

  ```
  [root@Server ~]# cd /etc/supervisord.d/
  ```

  - 新建 tornado.ini

  ```
  [group:tornadoes]
    programs=nginx-0,tornado-0

    # 这里我把nginx放到supervisor里自启的时候一并启动
    [program:nginx-0]
    command=nginx
    startsecs=1
    autostart=true
    autorestart=true
    redirect_stderr=true
    stdout_logfile=/home/nginx.log
    loglevel=info
    stdout_logfile_maxbytes = 50MB
    stdout_logfile_backups  = 10

    # tornado启动，我使用了一个sh文件
    [program:tornado-0]
    command=sh -x /home/tornado_start.sh
    directory=tornado项目文件夹路径
    startsecs=1
    autostart=true
    autorestart=true
    redirect_stderr=true
    stdout_logfile=tornado项目文件夹路径/tornado.log
    loglevel=info
    stdout_logfile_maxbytes = 50MB
    stdout_logfile_backups  = 10
  ```

  - 将 Tornado 项目文件放到服务器中，注意要与配置文件中的路径一致

  - 配置 Nginx 反向道理（因为 tornado 项目启动是http://127.0.0.1:8888这样的地址，那么我们需要把它映射到我们别的域名上供React调用）

  - 新建 tornado.conf 放入/etc/nginx/conf.d 中（conf.d 文件夹与 supervisord.d 文件夹功能相似）

  ```
    server {
        listen 80;
        server_name 域名;

        location / {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_pass http://127.0.0.1:8888;
        }
    }
  ```

   接下来就是要保证项目运行不报错，安装项目所需的包了，pip3 xxx（根据具体项目需求）。

## No.4

### **_安装 MySQL_**

- 下载 rpm 包

```
[root@Server Downloads]# wget https://repo.mysql.com//mysql80-community-release-el8-1.noarch.rpm
```

- 安装 rpm 包

```
[root@Server Downloads]# rpm -ivh mysql80-community-release-el8-1.noarch.rpm
```

- 安装 myql 服务

```
[root@Server ~]# yum install mysql-server
```

- 开机自启

```
[root@Server ~]# systemctl enable mysqld.service
Created symlink /etc/systemd/system/multi-user.target.wants/mysqld.service → /usr/lib/systemd/system/mysqld.service.
```

- 启动服务

```
[root@Server ~]# systemctl start mysqld.service
```

- 设置 mysql 密码

```
[root@Server ~]# mysql
...
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
Query OK, 0 rows affected (0.01 sec)
```

- 接下来就可以创建数据库、表和导入数据库文件啦。

最后重启一下服务器。

## 完工！

---

## 其他可能遇到的问题

### **_Python 提示 No module named '\_ssl'_**

- 修改 Setup 文件

```
[root@Server Python-3.8.1]# vi Modules/Setup

# Socket module helper for socket(2)
_socket socketmodule.c（去除注释）

# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
#SSL=/usr/local/ssl
_ssl _ssl.c \
       -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
       -L$(SSL)/lib -lssl -lcrypto
```

- 再次 make 和 make install 一下

- 导入包验证，没有报错就是没问题了。

```
>>> import ssl
>>> import _ssl
```

### **_怎么导入数据库文件_**

- 先进入 mysql

```
[root@Server ~]# mysql -u root -p
Enter password:
```

- 如果没有数据库，先新建一下

```
create database 数据库名;
use 数据库名;
set names utf8;
```

- 导入

```
source 数据库文件.sql;
```

注意数据库语句要加分号;

### **_指定配置文件启动 supervisor 时报错_**

- 指定配置文件启动 supervisor 命令

```
supervisord -c /etc/supervisord.conf
```

- 报错处理

```
find / -name supervisor.sock
# unlink 地址根据find返回的地址
unlink /run/supervisor/supervisor.sock
```

---

## 题外话

### **_为什么没用虚拟环境？_**

- 尝试过 anaconda，但是没有成功，有经验的小伙伴可以分享一下。
- 另外考虑站点不会有多版本的 python，也可以不用虚拟环境。

### **_为什么某些命令在远程中能使用，而服务器找不到该命令？_**

- 服务器不同的用户有不同的环境变量配置，这就是为什么会有系统环境变量这个概念。具体可以了解一下/etc/profile 的配置。
- 服务器会默认进入自己的 env，当然，阿里云也不例外。

### **_为什么下载的 pip 包显示已安装而项目却一直提示找不到？_**

- 关于这个就是 python 和 pip 地址是否一致的问题了。
- 很多文章只提到了改默认 python，但是要知道 pip 依然依赖原本默认的 python，并没有随着更改默认 python 而一并改过去。
- 所以 pip 安装的包，都到了之前的 python 文件夹下，自然读取不到。
- 在上面安装 python 的过程中，我提到了如果修改 pip，可以回头看看哦。

---

## 一些常用命令（个人用的比较多）

### **_Supervisor_**

- supervisorctl reload-重新加载
- supervisorctl status-查看状态
- supervisorctl restart xxx-重启进程
- supervisorctl start xxx-启动进程

### **_查看系统进程_**

- ps -ef | grep nginx-查看 nginx 进程
- kill -QUIT 进程号-关闭进程

### **_快速删除大量文件_**

```
# 新增一个空文件夹
mkdir /tmp/empty
rsync --delete-before -d /tmp/empty/ 目标文件夹地址
```

原理是将一个空文件夹替换掉目标文件夹的内容

## OK，BYE ～
