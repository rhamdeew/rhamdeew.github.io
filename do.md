Всем привет!
Сегодня я расскажу вам как сделать простой и удобный хостинг статичных сайтов на базе дроплета от DigitalOcean.
Итак, начнем

### Приготовления

1) Первым делом мы регистрируем [VPS на DigitalOcean](https://www.digitalocean.com/?refcode=02102c6f64b8). 
Выбираем образ Ubuntu 14.04.3 и регион ближайший к вам и вашим клиентам.

2) Далее мы начинаем настраивать нашу VPS

	ssh root@ip_addr (где ip_addr адрес нашей VPS)

Создаем пользователя у которого будут храниться наши статичные сайты:

	useradd -m -d /home/static static
	usermod -s /bin/bash static

Устанавливаем необходимые нам программы

	apt-get update
	apt-get install nginx git nano rsync

Разрешаем пользователю static перезагружать веб-сервер

	nano /etc/sudoers

И добавляем в конце файла строчку:

	static ALL=(ALL) NOPASSWD: /usr/sbin/service nginx reload


3) Делаем наш сервер чуточку безопаснее

	nano /etc/ssh/sshd_config

Меняем стандартный порт ssh

	Port 20002

Запрещаем логин от пользователя root

	PermitRootLogin no

Перезапускаем ssh

	service ssh restart

Задаем пароль пользователю static

	passwd static

*Пароль не должен совпадать с паролем пользователя root!*

Редактируем стандартный конфиг Nginx

	nano /etc/nginx/nginx.conf

Раскомментируем 
	
	server_tokens off;
	
	gzip types ...;
	
	gzip_comp_level 6;

Удаляем строки
	
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

Добавляем в конце файла
	
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


Удаляем стандартные конфиги

	rm -rf /etc/nginx/sites-available
	rm -rf /etc/nginx/sites-enabled

Добавляем общий конфиг

	mkdir /etc/nginx/global
	
	nano /etc/nginx/global/drop.conf

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

Создаем пользователю static нужные директории

	su static
	mkdir ~/nginx
	mkdir ~/git
	mkdir ~/bin
	mkdir ~/www
	mkdir ~/.ssh

Не закрываем терминал

4) Пробуем подключаться

**На локальной машине**

Генерируем ssh-ключ. Как это делать подробно описано [здесь](https://help.github.com/articles/generating-ssh-keys/)

	cat ~/.ssh/id_rsa.pub

**На сервере**

	nano ~/.ssh/authorized_keys

И вставляем скопированную публичную часть ключа в этот файл

**На локальной машине** 

Пробуем подключиться

	ssh static@ip_addr -p20002

Если подключение проходит нормально то добавляем конфигурацию для удобства

	nano ~/.ssh/config

	Host static
	User static
	Port 20002
	Hostname 188.166.69.147

Теперь добавляем локальную часть скрипта

	mkdir ~/.bin
	nano ~/.bin/static.sh


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


	chmod +x ~/.bin/static.sh
	
	nano ~/bashrc

И добавляем последней строчкой alias

	alias static="~/.bin/static.sh"

Запускаем bash заново

	bash -l


**На сервере** 
От пользователя static:

	nano ~/bin/static.sh

И вставляем содержимое

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

Делаем скрипт исполняемым

	chmod +x ~/bin/static.sh


### Все, настройка готова!

Для корректной работы на нашей локальной машине должны быть установлены утилиты git и rsync.

Теперь используя любимый static-генератор мы создаем сайт.

Опишу на примере [Hexo](http://hexo.io)

Создаем новый сайт и генерируем статику

	cd ~/site1
	hexo init
	npm install
	hexo generate

Теперь в директории ~/site1 у нас есть директория public со сгенерированным блогом.
Пробуем добавить сайт на наш сервер

	static

Отвечаем на вопросы...

В итоге у нас должен появиться скрипт deploy.sh 
Запускаем его:

	./deploy.sh

Вот и все, теперь сервер настроен на отдачу сайта site1.ru
В следующий раз после обновления статики и генерации контента нам остается только запустить deploy.sh

Также на сервере хранится git-репозиторий сайта который мы можем склонировать в любой момент командой

	git clone static:~/git/site1.ru
