## What is Ganglia

[Ganglia][1] is a grid monitoring system.

## Overview/Concepts

Major components.

1.  The stats-generators. The nodes you want to monitor. These need the package ganglia-monitor.
2.  The stats-collector(s). The machine(s) that collect the stats send by gmond. These need the package gmetad.
3.  The web-interface. This will display the data. These need the package ganglia-webfrontend.

One of the tricky things about Ganglia is how it connects to itself. It isn't that tricky, but it trips people up, so lets spell it out.:

*   stats-generators send their stats on to a stats collector box. The stats-gen and stats-col box run the same daemon (gmond, from ganglia-monitor) to do this. Which means they have different configs.
*   The stats-col boxes provide the data to the web front end, so it is also listening on it a port (8652 default) as well as making connections to its local gmond
*   The web frontend connects to the stats-col box (gmetad is the daemon) and gets the info
*   You can also talk to the ganglia-monitor and dump XML directly for troubleshooting purposes simply with nc localhost 8649

## Installation

*   On Ubuntu nodes: `sudo apt-get install ganglia-monitor` (this installs the service ganglia-monitor, which is gmond)
*   On Ubuntu web interface machine: `sudo apt-get install ganglia-webfrontend` (this installs gmetad and the web frontend)

## Configuration

### Stats Generation Boxes

#### ganglia-monitor (gmond)

<pre>
#/etc/ganglia/gmond.conf
cluster {
  #这个名字要个 `gmetad` 中的 `data_source` 配置保持一致
  name = "swift-node" 
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}

udp_send_channel {
  #默认是广播,监听此地址的服务都可以收到来自这个地址的广播消息,如果存在多个 ganglia 实例可以修改此地址
  #广播消息不能跨越不同的网段, 如果是用多网卡并且第一块网卡地址与 `gmetad` 服务的地址不在同一网段,数据无法传递
  #有两个解决办法 :
  #1. 修改成单播通讯方式 , 即将下面的IP地址选项设置为本机的IP地址(修改成本机IP地址后每个节点会单独显示, 分组可能会遇到问题)
  #   或者
  #   本组的IP地址 , (修改成本组的IP地址后,gmetad中的data_source 地址只写这个地址就可以收集所有节点数据,但是当这个地址的服务器下线后当前节点数据也收不到了)
  #2. 执行 `ip route add 239.2.11.71 dev eth2` eth2 替换成与  `gmetad` 服务相同IP段的网卡 , 使 `239.2.11.71` 地址的数据通过 `eth2` 传递 (推荐).
  mcast_join = 239.2.11.71 
  port = 8649
  ttl = 1
}

udp_recv_channel {
  port = 8649
  #多播通讯下面两个地址要个上面的 udp_send_channel#mcast_join 地址保持一致
  #单播通讯要注释或删除下面两个配置
  mcast_join = 239.2.11.71
  bind = 239.2.11.71

}
</pre>

##### 配置 `diskpart` 模块监控磁盘分区信息
	
<pre>
# line 88
module {
  name = "python_module"
  path = "/usr/lib/ganglia/modpython.so"
  params = "/usr/lib/ganglia/python_modules/"
}
# line 91
include ('/etc/ganglia/conf.d/*.conf')
</pre>

安装模块	

<pre>
sudo mkdir -p /etc/ganglia/conf.d
sudo mkdir -p /usr/lib/ganglia/python_modules

#模块
tar xzvf diskpart.tar.gz
sudo mv diskpart.py  /usr/lib/ganglia/python_modules/
</pre>  

配置磁盘监控

<pre>
 #/etc/ganglia/conf.d/diskpart.conf
modules {
  module {
    name     = "diskpart"
    language = "python"
  }
}

collection_group {
  collect_every  = 20
  time_threshold = 90

  metric {
    name  = "diskpart-root-total"
    title = "total partition space: root"
  }
  metric {
    name  = "diskpart-root-used"
    title = "partition space used: root"
  }
  metric {
    name  = "diskpart-root-inode-total"
    title = "total number of inode: root"
   }
  metric {
    name  = "diskpart-root-inode-used"
    title = "number of inode used: root"
  }
}
</pre>
>说明
>
>1. `name` 中的 `root` 表示磁盘的挂载点 , 例如: `/srv/node/sda1` => `diskpart-srv_node_sda1-total`
>2. `metric` 的数量没有约束 , 也就是说可以只监控每个磁盘的 `total` 和 `used` 信息 ,多个磁盘酌情添加 

`sudo /etc/init.d/ganglia-monitor restart`

#### ganglia-webfrontend (gmetad)

<pre>
#/etc/ganglia/gmetad.conf
data_source "ganglia" localhost
data_source "haproxy" 13.141.43.105
data_source "swift-node" 13.141.43.171 13.141.43.172 13.141.43.173
gridname "MyServer"
</pre>

They default will be set server to apache2 .

##### There is Nginx:
`ln -s /usr/share/ganglia-webfrontend /var/www/ganglia`

<pre>
#/etc/nginx/conf.d/ganglia.conf
server{
	listen 80;	
	server_name ganglia.dev;
	root /var/www/ganglia;

	location / {
		index index.html index.php;
		root /var/www/ganglia;
	}
	location ~ \.php$ {
    root /var/www/ganglia;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  /var/www/ganglia/$fastcgi_script_name;
    include        fastcgi_params;
  }
}
</pre>

restart nginx 

## Usage

enjoy it!

## FAQ

+ 为什么配置好后有些机器没有出现在监控界面 ?
>1. 可能是由于多网卡导致服务监听了第一块网卡 , 而第一块网卡的网段和 `gmetad` 的IP段不一致
>执行 `ip route add 239.2.11.71 dev eth2` eth2 替换成与  `gmetad` 服务相同IP段的网卡
>2. 另外要注意 `gmetad` 启动后要确保所有节点处于在线状态,否则可能无法显示该节点 .


## References

[Ganglia on Sourceforge][2]  
[Hadoop Metrics][3]  
[Setting up Hadoop Metrics][4]  
<http://www.ibm.com/developerworks/wikis/display/WikiPtype/ganglia>  
<http://agiletesting.blogspot.com/2010/09/quick-note-on-installing-and.html>  
<http://sourceforge.net/apps/trac/ganglia/wiki/ganglia_quick_start>  
<http://blog.stlhadoop.org/2010/11/ganglia-hadoop-monitoring.html> <http://docs.hortonworks.com/HDPDOCS/HDPv1.0.1.14/index.htm#Monitoring_HDP/>

 [1]: http://sourceforge.net/projects/ganglia/
 [2]: http://ganglia.sourceforge.net/
 [3]: http://wiki.apache.org/hadoop/GangliaMetrics
 [4]: http://www.ryangreenhall.com/2010/10/monitoring-hadoop-clusters-using-ganglia/
