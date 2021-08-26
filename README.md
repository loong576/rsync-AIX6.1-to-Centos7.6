**背景**：NBU的catalog日志在/drfile路径下，该日志很重要，记录了每天备份的详细行为，极端情况下，可以通过该日志将NBU恢复到对应某天状态，然后在进行数据恢复（前提是虚拟带库或者物理带库没有被复写）。现将该日志备份至某台Centos7.6。

**环境说明：**

| 主机名     | ip           | 操作系统版本 | 同步目录    | 备注   |
| ---------- | ------------ | ------------ | ----------- | ------ |
| nbu-master | 172.28.4.xx  | AIX 6.1      | /drfile     | 源端   |
| ansible    | 172.28.6.xxx | CentOS 7.6   | /drfile_bak | 目标端 |

## 一、源端rsync安装

### 1.Openssh安装

```bash
nbu-master:/ #ssh -V
OpenSSH_6.0p1, OpenSSL 0.9.8y 5 Feb 2013
```

查看是否安装了Openssh，本文已安装，这里忽略安装步骤。若未安装请参考[AIX环境下文件远程传输复制工具--rsync安装测试](https://blog.51cto.com/loong576/2445554)安装，安装包请在文末连接下载。

### 2.rsync安装

```bash
nbu-master:/tmp #rpm -ivh popt-1.7-2.aix5.1.ppc.rpm
popt                        ##################################################
nbu-master:/tmp #rpm -ivh rsync-2.6.2-1.aix5.1.ppc.rpm
rsync                       ##################################################
```

![image-20210826155651297](https://i.loli.net/2021/08/26/RUiFvexkIsQJnq8.png)

安装方式可以是rpm命令，也可以用smitty。

## 二、源端rsync配置

服务器端为源端（172.28.4.xx），源端配置文件主要为rsyncd.conf(主配置文件)、rsyncd.pwd(密码文件)、rsyncd.motd(rsync服务器信息)，在/etc下新建rsync目录，进入/etc/rsync新建配置文件**rsyncd.conf、rsyncd.pwd、rsyncd.motd**

### 1.rsyncd.conf

```bash
uid=root
gid=system
use chroot=true
log file=/var/log/rsyncd.log
pid file=/var/run/rsyncd.pid
motd file = /etc/rsync/rsyncd.motd
secrets file=/etc/rsync/rsyncd.pwd
transfer logging = true
hosts allow=172.28.6.xxx
[rsync]
path=/drfile
comment = home rsync 
read only = yes
list = yes 
auth users = root 
secrets file=/etc/rsync/rsyncd.pwd
```

rsyncd.conf 是rsync服务器主要配置文件，该文件默认不存在需手动创建。

### 2.rsyncd.pwd

```bash
nbu-master:/etc/rsync #cat rsyncd.pwd
root:A123abc!
nbu-master:/etc/rsync #ls -l|grep rsyncd.pwd
-rw-------    1 root     system           14 Aug 26 11:05 rsyncd.pwd
```

rsyncd.pwd为密码文件，格式为：user:password；此用户必须系统中存在，密码为rsync同步密码，可以与系统密码不同，服务器端与客户端保持一致即可；为保证密码的安全性，**密码文件权限应设置为600，属主为root**。

### 3.rsyncd.motd

```bash
nbu-master:/etc/rsync #cat rsyncd.motd
++++++++++++++++++++++++++++++++++++++++++++++
Welcome to use the Aix 6.1 To Centos7.6 rsync services!
backup files: /drfile
++++++++++++++++++++++++++++++++++++++++++++++
```

rsyncd.motd是定义rysnc 服务器信息的，也就是用户登录信息。比如让用户知道这个服务器是谁提供的等；类似ftp服务器登录时，我们所看到的提示信息。

## 三、目标端rsync安装

```bash
[root@ansible ~]# yum -y install rsync
```

Centos安装rsync可以直接使用yum安装

## 四、目标端rsync配置

### 1.rsync.passwd

```bash
[root@ansible etc]# more rsync.passwd 
A123abc!
[root@ansible etc]# chmod 600 /etc/rsync.passwd
```

在/etc目录下新建rsync.passwd并修改属性；客户端密码文件格式与服务器端不同，密码文件权限属性为属主可读。

### 2.新建同步目录

```bash
[root@ansible etc]# mkdir /drfile_bak
```

新建同步目录drfile_bak，用户同步源端的/drfile下的文件。

## 五、同步测试

### 1.源端启动rsync

```bash
nbu-master:/etc/rsync #ps -ef|grep rsync
    root 15335498  8847504   0 14:32:12  pts/1  0:00 grep rsync
nbu-master:/etc/rsync #/usr/bin/rsync --daemon --config=/etc/rsync/rsyncd.conf
nbu-master:/etc/rsync #ps -ef|grep rsync
    root  9633794        1   0 14:32:36      -  0:00 /usr/bin/rsync --daemon --config=/etc/rsync/rsyncd.conf
    root 17170500  8847504   0 14:32:41  pts/1  0:00 grep rsync
nbu-master:/etc/rsync #netstat -ano|grep 873
tcp4       0      0  *.873                  *.*                    LISTEN
```

![image-20210826161220534](https://i.loli.net/2021/08/26/cIYWHB6x2XGkSjR.png)

此服务项不会开机启动，服务端机器重启后需启动该服务;检查端口（rsync默认端口为873，端口监听证明服务拉起）

### 2.客户端同步测试

```bash
[root@ansible drfile_bak]# rsync -atvz --password-file=/etc/rsync.passwd root@172.28.3.xx::rsync /drfile_bak
++++++++++++++++++++++++++++++++++++++++++++++
Welcome to use the Aix 6.1 To Centos7.6 rsync services!
backup files: /drfile
++++++++++++++++++++++++++++++++++++++++++++++


receiving file list ... done
./
.lck
Catalog_1374690299_FULL
Catalog_1374775253_FULL
Catalog_1374861660_FULL
Catalog_1374948045_FULL
Catalog_1375034450_FULL
Catalog_1375120847_FULL
Catalog_1375156862_UBAK
......
[root@ansible drfile_bak]# ll|wc -l
5846
```

在客户端执行同步同步命令，同步了5846个文件

![image-20210826162225593](https://i.loli.net/2021/08/26/ZXCDPqN8pkL7QKu.png)

### 3.配置定时任务

```bash
[root@ansible drfile_bak]# crontab -l
0 0 * * * rsync -atvz --password-file=/etc/rsync.passwd root@172.28.3.xx::rsync /drfile_bak >>/drfile_bak/rsync.log
```

在客户端部署定时任务，每天晚上0点同步。
