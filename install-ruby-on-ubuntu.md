#Install ruby on ubuntu

安装环境依赖

	sudo apt-get install build-essential libssl-dev zlib1g-dev

下载源代码

	wget http://cache.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p247.tar.gz
	
解压缩

	tar xzvf ruby-2.0.0-p247.tar.gz
	cd ruby-2.0.0-p247
	./configure
	make
	sudo make install

更新 GEM

	gem update --system
	gem update
	
	

	sudo gem install rails