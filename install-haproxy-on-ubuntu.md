##Installing HAProxy
Use the `apt-get` command to install HAProxy.

	sudo apt-get update
	sudo apt-get install haproxy

We need to enable HAProxy to be started by the init script.

	vim /etc/default/haproxy

Set the ENABLED option to 1

	ENABLED=1

##Configuring HAProxy


We'll move the default configuration file and create our own one.


	sudo mv /etc/haproxy/haproxy.cfg{,.bak}
	
Create and edit a new configuration file:

	sudo vim /etc/haproxy/haproxy.cfg

```
global
    log 127.0.0.1 local0 notice
    maxconn 2000
    user haproxy
    group haproxy
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  10000
    timeout server  10000
listen appname 0.0.0.0:80
    mode http
    stats enable
    stats uri /haproxy
    stats realm Strictly\ Private
    stats auth admin:admin
    balance roundrobin
    option httpclose
    option forwardfor
    server query-server 13.141.43.227:80 check
    #rabbitmq status
    acl rabbitmq hdr_end(host) -i rabbitmq.query-server.dev
    use_backend rabbitmq_backend if rabbitmq
    #ganglia
    #acl ganglia hdr_end(host) -i ganglia.query-server.dev
    #use_backend ganglia_backend if ganglia

backend rabbitmq_backend
    server query-server 127.0.0.1:15672 check

```
    
```
sudo service haproxy start
```
