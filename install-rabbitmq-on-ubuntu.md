# Installing RabbitMQ on Ubuntu 12.04 LTS

	#添加 RabbitMQ 的singing-key-public.	wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc	sudo apt-key add rabbitmq-signing-key-public.asc
	echo "deb http://www.rabbitmq.com/debian/ testing main" >> /etc/apt/sources.list
	
	sudo apt-get update
    sudo apt-get install rabbitmq-server

或者

    dpkg -i http://www.rabbitmq.com/releases/rabbitmq-server/v3.2.2/rabbitmq-server_3.2.2-1_all.deb

安装控制插件，查看列表：
```
sudo rabbitmq-plugins listsudo rabbitmq-plugins enable rabbitmq_management
```一旦设成enable，你将看到以下几个选中：
```[e] amqp_client                       3.2.2
[ ] cowboy                            0.5.0-rmq3.2.2-git4b93c2d
[ ] eldap                             3.2.2-gite309de4
[e] mochiweb                          2.7.0-rmq3.2.2-git680dba8
[ ] rabbitmq_amqp1_0                  3.2.2
[ ] rabbitmq_auth_backend_ldap        3.2.2
[ ] rabbitmq_auth_mechanism_ssl       3.2.2
[ ] rabbitmq_consistent_hash_exchange 3.2.2
[ ] rabbitmq_federation               3.2.2
[ ] rabbitmq_federation_management    3.2.2
[ ] rabbitmq_jsonrpc                  3.2.2
[ ] rabbitmq_jsonrpc_channel          3.2.2
[ ] rabbitmq_jsonrpc_channel_examples 3.2.2
[E] rabbitmq_management               3.2.2
[e] rabbitmq_management_agent         3.2.2
[ ] rabbitmq_management_visualiser    3.2.2
[ ] rabbitmq_mqtt                     3.2.2
[ ] rabbitmq_shovel                   3.2.2
[ ] rabbitmq_shovel_management        3.2.2
[ ] rabbitmq_stomp                    3.2.2
[ ] rabbitmq_tracing                  3.2.2
[e] rabbitmq_web_dispatch             3.2.2
[ ] rabbitmq_web_stomp                3.2.2
[ ] rabbitmq_web_stomp_examples       3.2.2
[ ] rfc4627_jsonrpc                   3.2.2-git5e67120
[ ] sockjs                            0.3.4-rmq3.2.2-git3132eb9
[e] webmachine                        1.10.3-rmq3.2.2-gite9359c7
```重启服务后，改变将生效。
`sudo service rabbitmq-server restart`
首先创建一个用户
	sudo rabbitmqctl add_user user-name your-password
	
	#Give new user administration privileges
	sudo rabbitmqctl set_user_tags admin administrator`
	
	#Lets assign permissions to admin user
	sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
	#Delete the default guest account
	sudo rabbitmqctl delete_user guest
	#Restart the server for the changes to take effect.
	sudo service rabbitmq-server restart

Lastly, navigate to <http://localhost:55672> to view the web interface for your RabbitMQ server##Celery


    |- https://pypi.python.org/packages/source/c/celery/celery-3.1.7.tar.gz
    |-- https://pypi.python.org/packages/source/k/kombu/kombu-3.0.8.tar.gz
      |- https://pypi.python.org/packages/source/a/anyjson/anyjson-0.3.3.tar.gz
      |- https://pypi.python.org/packages/source/a/amqp/amqp-1.3.3.tar.gz
    |-- https://pypi.python.org/packages/source/p/pytz/pytz-2013.9.tar.gz
    |-- https://pypi.python.org/packages/source/b/billiard/billiard-3.3.0.13.tar.gz

##Redis server

    https://pypi.python.org/packages/source/r/redis/redis-2.9.0.tar.gz


