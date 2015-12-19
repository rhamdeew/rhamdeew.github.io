Hello!
Today I'll show you how to make simple and convenient hosting for static websites based on Linode VPS.
Begin:

### Prepare

1) We register [VPS on Linode](http://linode.com). 
Choose image Ubuntu 14.04.3 and suiting your region.

2) Adjust our VPS

	ssh root@ip_addr (где ip_addr адрес нашей VPS)

Create user for static sites:

	useradd -m -d /home/static static
	usermod -s /bin/bash static

Instal additional software

	apt-get update
	apt-get install nginx git nano rsync

Give persmissions to user "static" for reload Nginx

	nano /etc/sudoers

Add line to end:

	static ALL=(ALL) NOPASSWD: /usr/sbin/service nginx reload

Save file

3) Secure server

	nano /etc/ssh/sshd_config

Change standart SSH-port

	Port 20002

Deny root login

	PermitRootLogin no
	
Save file

Restart SSH daemon

	service ssh restart

Set password for user "static"

	passwd static

*The password should not match the root password!*

Edit default Nginx config

	nano /etc/nginx/nginx.conf

Uncomment 
	
	server_tokens off;
	
	gzip types ...;
	
	gzip_comp_level 6;

Delete
	
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

Add line to end
	
	include /home/static/nginx/*.conf;

	server {
	        listen 80 default_server;
	        listen [::]:80 default_server ipv6only=on;

	        root /usr/share/nginx/html;
	        index index.html index.htm;

	        server_name localhost;

	        location / {
	                try_files $uri $uri/ =404;
	        }
	}

Save file

Delete default configuration files

	rm -rf /etc/nginx/sites-available
	rm -rf /etc/nginx/sites-enabled

Add shared config

	mkdir /etc/nginx/global
	
	nano /etc/nginx/global/drop.conf
	
Insert

	# log access to drop file in /etc/nginx/ but don't log 404
	location = /robots.txt {
	        access_log drop;
	        log_not_found off;
	}

	# log access to drop file in /etc/nginx/ but don't log 404
	location = /favicon.ico {
	        access_log drop;
	        log_not_found off;
	}

	# log access to denied file in /etc/nginx/ but don't log 404 and also deny all to dot files
	location ~ /\. {
	        access_log denied;
	        log_not_found off;
	        deny all;
	}

	# log access to drop file in /etc/nginx/ but don't log 404 and also deny all to files starting with a dollar sign ($temp.config.php)
	location ~ ~$ {
	        access_log denied;
	        log_not_found off;
	        deny all;
	}
	
Save file

Create directories need for user "static"

	su static
	mkdir ~/nginx
	mkdir ~/git
	mkdir ~/bin
	mkdir ~/www
	mkdir ~/.ssh

Do not close the terminal

4) Trying connection

**On local machine**

Generate ssh-keys. Example [here](https://help.github.com/articles/generating-ssh-keys/)

	cat ~/.ssh/id_rsa.pub

**On server**

	nano ~/.ssh/authorized_keys

And paste id_rsa.pub content here

**On local machine** 

Trying connection

	ssh static@ip_addr -p20002

If success add local ssh config

	nano ~/.ssh/config

	Host static
	User static
	Port 20002
	Hostname 188.166.69.147

Now add local part of script

	mkdir ~/.bin
	
	nano ~/.bin/static.sh
	
Insert


		#!/bin/bash
		echo "Site domain (without www.)?"
		read domain

		echo "Site data directory?"
		read directory

		ssh static -t "~/bin/static.sh $domain $directory"


		echo "$directory/" >> .gitignore
		git init
		git add -A
		git commit -m \"first\"
		git remote add origin static:~/git/$domain

		echo "
		#!/bin/bash
		git add -A && git commit -m 'revision' && git push origin master
		rsync -avz $directory/ static:~/www/$domain
		" > deploy.sh
		chmod +x deploy.sh

Save file and make it executable

	chmod +x ~/.bin/static.sh
	
Edit our bashrc
	
	nano ~/bashrc

And add alias to end of file

	alias static="~/.bin/static.sh"
	
Save file

Execute bash

	bash -l


**On server** 
For user "static":

	nano ~/bin/static.sh

Insert

		#!/bin/bash
		domain=$1
		directory=$2

		echo "Configure git";
		echo "---------------";
		mkdir ~/git/$domain
		cd ~/git/$domain
		git --bare init

		mkdir ~/www/$domain
		cd ~/www/$domain
		echo "---------------";

		echo "Configure Nginx";
		echo "---------------";

		echo "
		server {
		        listen 80;
		        listen [::]:80;
		        server_name www.$domain;
		        rewrite ^ \$scheme://$domain\$request_uri redirect;
		}

		server {
		    listen       80;
		    listen [::]:80;
		    server_name  $domain;

		    set \$root_path /home/static/www/$domain;

		    include global/drop.conf;
		    location ~* ^.+\.(jpg|jpeg|gif|png|svg|js|css|mp3|ogg|mpe?g|avi|zip|gz|bz2?|rar|swf|woff|eot|ttf)$ {
		        root   \$root_path;
		        expires 30d;
		        access_log off;
		    }

		    location / {
		        root   \$root_path;
		        index  index.html;
		    }

		    error_page  404  /404.html;
		}
		" > ~/nginx/$domain.conf
		echo "---------------";


		echo "Reloading Nginx\'s";
		echo "---------------";

		sudo service nginx reload

		echo "---------------";

Save and make executable

	chmod +x ~/bin/static.sh


### Ready to deploy!

Please install git and rsync locally!

Now we make static-site with lovely static-generator.

Here is example with [Hexo](http://hexo.io)

Make new site and generate

	mkdir ~/site1.ru
	cd ~/site1.ru
	hexo init
	npm install
	hexo generate

Now in ~/site1.ru isset "public" directory with content.

Trying to add site on server

	static

Answer to questions... (site.ru, public)

Now we should see "deploy.sh" 

Execute:

	./deploy.sh

It's all!  Now server configured to site1.ru

For update site simply execute script ./deploy.sh

Also on server exists git-repository with site code. Clone it later if you need:

	git clone static:~/git/site1.ru

### Result

We make self hosting for static sites and get easy script to add new hosts to server.
