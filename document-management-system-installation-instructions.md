# 单证查询系统搭建说明

## 准备工作

全部网络分为三层：

1. `swift` 存储仓库，包括5个存储节点和1个存储代理（proxy）服务器
2. 包括查询服务器、和存储代理服务器，存储代理服务器跨第一层和第二层
3. 查询服务器，对外提供 `web` 服务。

总共需要安装 *7* 个服务器

### PXE 安装服务器

首先准备安装 `PXE` 服务器 , 后面的服务器都将由这台服务器自动安装完成 .

安装PC系统环境使用 *Ubuntu12.04 Server LTS + dnsmasq + tftp + nginx + apt-cacher*。


	apt-get install openssh-server dnsmasq tftpserver apache2

修改配置文件：`/etc/dnsmasq.conf`

	bogus-priv
	filterwin2k
	interface=eth0
	dhcp-range=192.168.33.50,192.168.33.150,12h
	dhcp-boot=pxelinux.0
	enable-tftp
	tftp-root=/var/lib/tftpboot
	dhcp-authoritative

**配置 `NetBoot`	**


	cd /var/lib/tftpboot/
	#10.04
	wget http://cn.archive.ubuntu.com/ubuntu/dists/lucid/main/installer-amd64/current/images/netboot/netboot.tar.gz
	#11.10
	wget http://cn.archive.ubuntu.com/ubuntu/dists/oneiric/main/installer-amd64/current/images/netboot/netboot.tar.gz
	#12.04
	wget http://cn.archive.ubuntu.com/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/netboot.tar.gz

解压缩文件：
	
	tar -xzvf netboot.tar.gz

复制安装的系统文件到 `/var/www/ubuntu` 目录下


**配置 `pxe` 的 `MAC地址` 检测文件**

假设安装目标的服务器MAC地址为`d4-ae-52-64-b3-1d`，建立文件`/var/lib/tftpboot/pxelinux.cfg/01-d4-ae-52-64-b3-1d`，内容：
	
	default install
	label install
        menu label ^Install
        menu default
        kernel ubuntu-installer/amd64/linux
        append ks=http://192.168.33.1/ks.cfg preseed/url=http://192.168.33.1/ubuntu/preseed/	ubuntu-server.seed vga=normal initrd=ubuntu-installer/amd64/initrd.gz – quiet

建立文件 `/var/www/ks.cfg`，内容：

	lang en_US.UTF-8
	langsupport en_US.UTF-8
	keyboard us
	mouse
	timezone Asia/Shanghai
	rootpw --disabled
	#明文密码,不能少于6位,不然会中断自动应答,问你要强密码
	#密文可以执行echo 1234567890 | openssl passwd -1 -stdin
	#user www --fullname="www" --iscrypted --password $1$YKmaOIb5$/13bs7gCjaoH./ohFT0A7/
	user ubuntu --fullname="ubuntu" --password ubuntu
	install
	url --url http://192.168.33.1/ubuntu
	bootloader --location=mbr
	zerombr yes
	clearpart --all --initlabel  --drives=sda
	#size单位为M
	part / --fstype ext4 --size 51200
	part swap --size 4048
	network --bootproto=dhcp --device=eth0
	#静态ip
	#network --bootproto=static --ip=192.168..168 --netmask=255.255.255.0 --	gateway=192.168.5.112 \ nameserver=221.5.88.88 --device=eth0
	firewall --disabled
	skipx
	%packages
	@openssh-server

### 安装 apt-cacher

	apt-get install apt-cacher

修改配置文件，`/etc/apt-cacher/apt-cacher.conf` 修改内容：
	
	cache_dir = /var/cache/apt-cacher
	log_dir = /var/log/apt-cacher
	daemon_port = 3142
	group = www-data
	user = www-data
	allowed_hosts = *
	path_map = ubuntu cn.archive.ubuntu.com/ubuntu ; ubuntu-updates cn.archive.ubuntu.com/	ubuntu ; security cn.archive.ubuntu.com/ubuntu security.debian.org/debian-security ; swift 	ppa.launchpad.net/swift-core/release/ubuntu ; swauth gholt.github.com/swauth/lucid ; 	grizzly ubuntu-cloud.archive.canonical.com/ubuntu ; rabbitmq www.rabbitmq.com/debian ; 	10gen downloads-distro.mongodb.org/repo/ubuntu-upstart
	

至此 , **PXE-Server** 安装完毕.

## Swift 储存服务器安装

安装Swift1.4.6 , 建立文件 `/etc/apt/sources.list.d/swift-core-release-lucid.list` 内容为：

	deb http://13.141.43.17:3142/swift lucid main
	

安装Swift1.8.0 , 建立文件/etc/apt/sources.list.d/grizzly.list，内容：

	deb http://13.141.43.17:3142/swift precise-updates/grizzly main
	deb http://13.141.43.17:3142/swift precise-proposed/grizzly main
	
安装Swift1.10.0 , 建立文件/etc/apt/sources.list.d/grizzly.list，内容：

	deb http://13.141.43.17:3142/swift precise-updates/havana main
	deb http://13.141.43.17:3142/swift precise-proposed/havana main

安装
	
	apt-get update
	apt-get install swift
	
建立 `swift` 配置目录

	mkdir -p /etc/swift
	chown -R swift:swift /etc/swift/

### Swift 储存代理

	apt-get install swift-proxy memcached	
	
配置 `memcached`
	
	perl -pi -e "s/-l 127.0.0.1/-l $PROXY_LOCAL_NET_IP/" /etc/memcached.conf
	service memcached restart #启动memcached服务
	
	

建立配置文件 `/etc/swift/proxy-server.conf`：

	export PROXY_LOCAL_NET_IP=192.168.33.206
	
	cat >/etc/swift/proxy-server.conf <<EOF
	[DEFAULT]
	cert_file = /etc/swift/cert.crt
	key_file = /etc/swift/cert.key
	bind_port = 443
	workers = 8
	user = swift
 
	[pipeline:main]
	pipeline = healthcheck cache tempauth proxy-server
 
	[app:proxy-server]
	use = egg:swift#proxy
	allow_account_management = true
	account_autocreate = true
 
	[filter:tempauth]
	use = egg:swift#tempauth
	user_system_root = testpass .admin https://$PROXY_LOCAL_NET_IP/v1/AUTH_system
 
	[filter:healthcheck]
	use = egg:swift#healthcheck
 
	[filter:cache]
	use = egg:swift#memcache
	memcache_servers = $PROXY_LOCAL_NET_IP:11211
	EOF
	
生成密钥Key文件
	
	cd /etc/swift
	openssl req -new -x509 -nodes -out cert.crt -keyout cert.key


建立配置文件，创建 `account`, `container` 和 `object` `rings`。

	cd /etc/swift
	swift-ring-builder account.builder create 18 3 1
	swift-ring-builder container.builder create 18 3 1
	swift-ring-builder object.builder create 18 3 1
	
参数的值：

+ `18` 代表2 ^第十八，价值的分区大小。这个“分区”的价值基础上的存储总量你期望你的整个环使用。
+ `3`  表示每份文件(`object`)所备份的数量
+  最后被数个小时的限制移动分区超过 `1` 次。
	

具体的步骤是例如有 `3` 个 `ZONE`，每个 `ZONE` 有一台服务器，每台服务器有`二块硬盘`作为存储使用

首先是第一个zone

	export ZONE=1                    # 设置ZONE序号 ZONE 的值要从1开始，每次 +1
	export STORAGE_LOCAL_NET_IP=192.168.33.201    # 存储节点的IP地址
	export WEIGHT=100               # 存储的配重
	export DEVICE=sdb1
	swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
	swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
	swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

第二个zone

	export ZONE=2                    # 设置ZONE序号
	export STORAGE_LOCAL_NET_IP=192.168.33.202    # 存储节点的IP地址
	export WEIGHT=100               # 存储的配重
	export DEVICE=sdc1
	swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
	swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
	swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

第三个zone

	export ZONE=3                    # 设置ZONE序号
	export STORAGE_LOCAL_NET_IP=192.168.33.203    # 存储节点的IP地址
	export WEIGHT=100               # 存储的配重
	export DEVICE=sdd1
	swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
	swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
	swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT


确认 `ZONE` 的配置

	swift-ring-builder account.builder
	swift-ring-builder container.builder
	swift-ring-builder object.builder
	
调整 `ZONE` 的负载配置

	swift-ring-builder account.builder rebalance
	swift-ring-builder container.builder rebalance
	swift-ring-builder object.builder rebalance


复制存储代理节点到每个存储节点。

	cd /etc/swift
	scp account.ring.gz container.ring.gz object.ring.gz 13.141.43.236:/etc/swift/

确认目录的权限

	chown -R swift:swift /etc/swift

启动存储代理节点的代理进程

	swift-init proxy start
 

### Swift 储存节点

	apt-get install swift-account swift-container swift-object xfsprogs	

	
建立rsyncd配置文件/etc/rsyncd.conf：

	cat >/etc/rsyncd.conf <<EOF
	uid = swift
	gid = swift
	log file = /var/log/rsyncd.log
	pid file = /var/run/rsyncd.pid
	 
	address = $STORAGE_LOCAL_NET_IP
	 
	[account]
	max connections = 2
	path = /srv/node/
	read only = false
	lock file = /var/lock/account.lock
	 
	[container]
	max connections = 2
	path = /srv/node/
	read only = false
	lock file = /var/lock/container.lock
	 
	[object]
	max connections = 2
	path = /srv/node/
	read only = false
	lock file = /var/lock/object.lock
	EOF

修改/etc/default/rsync文件内容

	perl -pi -e 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/' /etc/default/rsync

启动rsync服务

	service rsync start
	
将硬盘格式化为 `XFS volume` 格式，挂载到 `/srv/node` 目录下

	fdisk /dev/sdb  (set up a single partition)
	mkfs.xfs -i size=1024 /dev/sdb1
	echo "/dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab
	mkdir -p /srv/node/sdb1
	mount /srv/node/sdb1
	chown -R swift:swift /srv/node
	
### parted 分区

大容量磁盘分区时, 使用 `fdisk` 可能会遇到问题 , 此时可以使用 `parted` 工具操作:

	root@taskserver:/home/david# parted /dev/sdb
	GNU Parted 2.3
	Using /dev/sdb
	Welcome to GNU Parted! Type 'help' to view a list of commands.
	(parted) print
	Error: /dev/sdb: unrecognised disk label
	(parted) mklabel
	New disk label type? gpt
	(parted) print
	Model: SEAGATE ST4000NM0023 (scsi)
	Disk /dev/sdb: 4001GB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	 
	Number  Start  End  Size  File system  Name  Flags
	 
	(parted) mkpart
	Partition name?  []?
	File system type?  [ext2]? xfs
	Start? 0
	End? 4TB
	Warning: The resulting partition is not properly aligned for best performance.
	Ignore/Cancel? i
	(parted) print
	Model: SEAGATE ST4000NM0023 (scsi)
	Disk /dev/sdb: 4001GB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt
	 
	Number  Start   End     Size    File system  Name  Flags
	 1      17.4kB  4001GB  4001GB
	 
	(parted)
	
### 配置 Swift

设置变量参数

	export STORAGE_LOCAL_NET_IP=192.168.33.201~205
	export PROXY_LOCAL_NET_IP=192.168.33.206

在第一个节点，建立配置文件
	

	cat >/etc/swift/swift.conf <<EOF
	[swift-hash]
	# random unique string that can never change (DO NOT LOSE)
	swift_hash_path_suffix = `od -t x8 -N 8 -A n </dev/random`
	EOF

将配置文件分发到其他节点
	
	scp /etc/swift/swift.conf 192.168.33.201:/etc/swift/


建立配置文件/etc/swift/account-server.conf:

	cat >/etc/swift/account-server.conf <<EOF
	[DEFAULT]
	bind_ip = $STORAGE_LOCAL_NET_IP
	workers = 2
	 
	[pipeline:main]
	pipeline = account-server
	 
	[app:account-server]
	use = egg:swift#account
	 
	[account-replicator]
	 
	[account-auditor]
	 
	[account-reaper]
	EOF

以及 `/etc/swift/container-server.conf`:

	cat >/etc/swift/container-server.conf <<EOF
	[DEFAULT]
	bind_ip = $STORAGE_LOCAL_NET_IP
	workers = 2
	 
	[pipeline:main]
	pipeline = container-server
	 
	[app:container-server]
	use = egg:swift#container
	 
	[container-replicator]
	 
	[container-updater]
	 
	[container-auditor]
	EOF

最后 `/etc/swift/object-server.conf`:

	cat >/etc/swift/object-server.conf <<EOF
	[DEFAULT]
	bind_ip = $STORAGE_LOCAL_NET_IP
	workers = 2
	 
	[pipeline:main]
	pipeline = object-server
	 
	[app:object-server]
	use = egg:swift#object
	 
	[object-replicator]
	 
	[object-updater]
	 
	[object-auditor]
	EOF

都完成后启动服务

	swift-init all start
	 
### 建立管理员权限

进入存储代理服务器，建立管理员，用户名是 `system:root`，密码 `testpass`
	
	curl -k -v -H 'X-Storage-User: system:root' -H 'X-Storage-Pass: testpass' https://$PROXY_LOCAL_NET_IP/auth/v1.0
	
会得到用户的 `X-Auth-Token` 和 `X-Storage-Url`

检查用户权限

	curl -k -v -H 'X-Auth-Token: <token-from-x-auth-token-above>' <url-from-x-storage-url-above>
 
### 安装 `swauth` 认证


下载 [swauth](https://github.com/gholt/swauth) 包，解压缩 , 运行 `python setup.py install` 命令安装
          
编辑配置文件 `/etc/swift/proxy-server.conf` ,修改 `[pipeline:main]` 节中的
	
	pipeline = healthcheck cache tempauth proxy-server
 
将 `tempauth` 修改为 `swauth` 并添加
	 
	[filter:swauth]
	use = egg:swauth#swauth
	set log_name = swauth
	super_admin_key = swauthkey
	default_swift_cluster = local#https://$PROXY_LOCAL_NET_IP/v1
	 
到配置文件内 , 运行 `swift-init proxy reload` 命令使配置生效

检测 `swauth` ：

	swauth-prep -K swauthkey -A https://13.141.43.80/auth，##没有错误输出表示正常。
	
添加 `swauth` 用户：

	swauth-add-user -K swauthkey -A https://13.141.43.80/auth -a test tester testing

查看用户信息：
	
	swauth-list -K swauthkey -A https://13.141.43.80/auth



### Swift升级

复制源1.4.6服务器中每个服务器的 `/etc/swift` 文件到新的1.8.0服务器中，确认 `proxy` 服务器中所有IP地址相关的配置，包括文件：

	/etc/swift/proxy-server.conf
	/etc/memcached.conf

确认存储节点服务器中所有IP地址相关的配置，包括文件：

	/etc/rsyncd.conf
	/etc/swift/account-server.conf
	/etc/swift/container-server.conf
	/etc/swift/object-server.conf

在 `proxy` 使用 `swift-ring-builder` 的 `set_info` 命令修改 `ring` 中的每个存储设备的信息，例如：

	swift-ring-builder account.builder set_info d0 13.141.43.81:6002/sdb1

完成后重新生成 `ring.gz` 文件：
	
	swift-ring-builder account.builder write_ring
	swift-ring-builder container.builder write_ring
	swift-ring-builder object.builder write_ring
	
重启 `swift`：

	swift-init all restart

确认 `swauth` 账户 `test` 正常
	
	swauth-list -K swauthkey -A https://13.141.43.80/auth test

如果 `proxy` 修改了IP地址，需要修改 `swauth` 用户的IP地址信息：
	
	swauth-set-account-service -K swauthkey -A https://13.141.43.80/auth test storage local https://13.141.43.80/v1/AUTH_121195c0-99a2-4d17-b127-fea144b6a58b
	

此处的local要和之前proxy-server.conf中default_swift_cluster一项中的值相对应。

### 常用的swift命令：

检查 `swift` 的状态

	swift -A https://$PROXY_LOCAL_NET_IP/auth/v1.0 -U system:root -K testpass stat

上传 `swift` 的配置文件到名为 `builders` 的 `container` 中
	
	swift -A https://$PROXY_LOCAL_NET_IP/auth/v1.0 -U system:root -K testpass upload builders /etc/swift/*.builder
	
列出 `swift` 的所有 `container`
	
	swift -A https://$PROXY_LOCAL_NET_IP/auth/v1.0 -U system:root -K testpass list

下载 `builders`的 `container` 中的所有文件

	swift -A https://$PROXY_LOCAL_NET_IP:8080/auth/v1.0 -U system:root -K testpass download builders 

    




## 查询服务器的部署

修改 `sudo` 不用输入密码

	sudo visudo

在文件结尾增加一行：
	
	%david ALL=(ALL) NOPASSWD: ALL

### 安装所需的软件

	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get install build-essential curl git-core zlib1g-dev libssl-dev imagemagick cifs-	utils keyutils smbfs vsftpd makepasswd libaio1 libdbi-perl libnet-daemon-perl libplrpc-	perl ttf-wqy-microhei ttf-wqy-zenhei
	sudo apt-get install libcurl4-openssl-dev p7zip-full libtool inoticoming ghostscript unzip

### 安装数据库

系统使用percona封装的mysql，系统使用的是5.1版本，从percona网站下载文件：
	
	mkdir percona
	cd percona
	wget http://www.percona.com/redir/downloads/Percona-Server-5.1/LATEST/deb/oneiric/x86_64/libmysqlclient16-dev_5.1.63-rel13.4-443.oneiric_amd64.deb
	wget http://www.percona.com/redir/downloads/Percona-Server-5.1/LATEST/deb/oneiric/x86_64/libmysqlclient16_5.1.63-rel13.4-443.oneiric_amd64.deb
	wget http://www.percona.com/redir/downloads/Percona-Server-5.1/LATEST/deb/oneiric/x86_64/percona-server-client-5.1_5.1.63-rel13.4-443.oneiric_amd64.deb
	wget http://www.percona.com/redir/downloads/Percona-Server-5.1/LATEST/deb/oneiric/x86_64/percona-server-common_5.1.63-rel13.4-443.oneiric_amd64.deb
	wget http://www.percona.com/redir/downloads/Percona-Server-5.1/LATEST/deb/oneiric/x86_64/percona-server-server-5.1_5.1.63-rel13.4-443.oneiric_amd64.deb
	sudo dpkg -i *.deb
	sudo apt-get -f install
	
#### MySQL主从配置

主服务器地址: `13.141.43.230`
从服务器地址: `13.141.43.225`

数据库名称 `dms_development`

**主数据库端：**

修改配置文件 `/etc/mysql/my.cnf`，注释以下内容：

	#bind-address           = 127.0.0.1

重启 `mysql`
	
	/etc/init.d/mysql restart
	netstat -tap | grep mysql，确认mysql可以监听远程端口调用。
	tcp        0      0 *:mysql                 *:*                     LISTEN      -

登陆 `mysql`，建立备份账号

	GRANT REPLICATION SLAVE ON *.* TO 'slave'@'13.141.43.225' IDENTIFIED BY '123456';
	FLUSH PRIVILEGES;

修改 `mysql` 配置文件 `/etc/mysql/my.cnf`，修改以下内容
	
	server-id               = 1
	log_bin                 = /var/log/mysql/mysql-bin.log
	expire_logs_days        = 10
	max_binlog_size         = 100M
	binlog_do_db            = dms_development

重启 `mysql` 服务，登陆 `mysql`，将数据库锁住

	USE dms_development;

	FLUSH TABLES WITH READ LOCK;   //不要退出这个终端，否则这个锁就不生效了。从服务器的数据库建好后。在主
	服务器执行解锁

查看主服务器数据库状态
	
	mysql> SHOW MASTER STATUS;
	
	+------------------+----------+--------------+------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
	+------------------+----------+--------------+------------------+
	| mysql-bin.000001 |    19467 |dms_development|                 |
	+------------------+----------+--------------+------------------+
	1 row in set (0.00 sec)

不要退出 `mysql`，另外一个进程登陆服务器，备份 `mysql`数据库
	
	mysqldump -uroot -pXXXXXX --opt dms_development > snapshot.sql

将 `snapshot.sql` 复制到从服务器上。

**转到从服务器**

在上面锁住mysql数据库的界面，给数据库解锁

	UNLOCK TABLES; 

修改 `mysql` 配置文件 `/etc/mysql/my.cnf` 添加以下内容：

	server-id=2
	#master-connect-retry=60                      #加上这句mysql无法启动，先注释掉
	replicate-do-db= dms_development

重启mysql数据库服务，`CREATE DATABASE dms_development;` 退出数据库，恢复数据库：

	mysqladmin --user=root --password=XXXXXX stop-slave
	mysql -uroot -pXXXXX dms_development < snapshot.sql

登录服务器，设置主服务器信息：

	CHANGE MASTER TO MASTER_HOST='dms-server', MASTER_USER='slave', MASTER_PASSWORD='123456', 	MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=19467;

注意 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 要和主服务器相同。

启动从服务器模式

	START SLAVE;

查看从数据库状态：
	
	SHOW SLAVE STATUS \G，其中存在两项：
	Slave_IO_Running: Yes
	Slave_SQL_Running: Yes

### 安装 WebServer

**安装 `RVM`**

	bash -s stable < <(curl -s https://raw.github.com/wayneeseguin/rvm/master/binscripts/rvm-installer )

**安装 `ruby1.9.3`**
	
	rvm install ruby-1.9.3
	gem install rails

在线安装 `passenger、nginx`

	gem install passenger
	sudo passenger-install-nginx-module

离线安装RVM、ruby

[Rvm](https://github.com/wayneeseguin/rvm)
[Ruby](ftp://ftp.ruby-lang.org/pub/ruby/)
[Rubygem](http://rubygems.org/pages/download)
	
	tar –xvf rvm-1.16.13.tar.gz
	./install –auto
	source ~/.rvm/scripts/rvm
 
	cp ruby-1.9.3-p286.tar.bz2 rubygems-1.8.24.tgz yaml-0.1.4.tar.gz $rvm_path/archives/
	echo rubygems_version=1.8.24 >> $rvm_path/user/db
	echo "" > ~/.rvm/gemsets/default.gems
	echo "" > ~/.rvm/gemsets/global.gems
	sh install_sh.sh
	rvm install 1.9.3-p286
 
	cd gems/vendor/cache/
	gem install bundler-1.2.1.gem
	bundle install --local
 
	rvm implode
	passenger离线安装nginx

下载网址	<http://nginx.org/download/>
首先确认nginx解压的目录，在安装nginx中需要例如：

	/home/david/setup-package/irrgrmpn/nginx-1.3.7
	rvmsudo passenger-install-nginx-module

在选择1、2时选择2离线安装。


Nginx开机自启动

写文件/etc/init.d/nginx
		
	#! /bin/sh
	# /etc/init.d/nginx
	#
	 
	# Some things that run always
	#touch /var/lock/blah
	 
	# Carry out specific functions when asked to by the system
	case "$1" in
	  start)
	    echo "Starting script nginx "
	        /opt/nginx/sbin/nginx
	    echo "start success."
	    ;;
	  stop)
	    echo "Stopping script nginx"
	        /opt/nginx/sbin/nginx -s stop
	    echo "stoped."
	    ;;
	  restart)
	    echo "Restaring script nginx"
	        /opt/nginx/sbin/nginx -s reload
	    echo "reload ok"
	    ;;
	  *)
	    echo "Usage: /etc/init.d/nginx {start|stop|restart}"
	    exit 1
	    ;;
	esac
	 
	exit 0

**安装node**

下载网址http://nodejs.org/dist/
	
	cd node-v0.6.14/
	./configure
	make
	sudo make install

**安装jpeg支持库**

下载文件jpegsrc.v6b.tar.gz
	
	tar –xvf jpegsrc.v6b.tar.gz
	./configure
	make
	sudo make install

**安装spawning**

下载spawning文件。
	
	wget http://pypi.python.org/packages/source/S/Spawning/	Spawning-0.9.7.tar.gz#md5=d9e517ab33bba1961a4733aec8e1c92e

执行安装命令

	sudo apt-get install python-dev python-setuptools
	pip install eventlet（离线安装使用）
	python setup.py install

### 项目部署

安装查询项目目录~/query-server/、~/tcollector/和~/bin/。
检录目录：

	mkdir ~/var
	mkdir ~/var/run
	mkdir ~/var/log
	mkdir ~/datastore
	mkdir ~/incoming_json

修改项目目录 `~/query-server/public/docimages/` 为真实目录。

服务器启动

### 挂载密钥盘

首先确认系统挂载了RAM盘，在/etc/fstab中确认存在以下内容：

	tmpfs /home/david/bin/conf tmpfs uid= david,gid= david,size=1M 0       0

查看磁盘信息状态

	sudo fdisk –l

假设u盘盘符是/dev/sdb1，并且系统存在/media/u目录
	
	sudo mount -o uid=david,gid=david,rw /dev/sdb1 /media/u
	cp -r /media/u/conf/* bin/conf/

将密码本解密
	
	./recover.sh
	
	
All done .