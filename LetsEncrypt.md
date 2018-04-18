# Let's Encryptを使う

* nginxに手作業で Let's Encrypt を設定します。
* サーバは Ubuntu 16.04.1 LTS です。
* サーバには事前にドメインを割り当てて、名前解決ができるようにしておく必要があります。(正引きのみで可)
* サーバ証明書を発行するドメイン名は **example.domain.dom** とします。LE_TARGET_DOMAIN環境変数としてexportします。

## インストール

	$ sudo mkdir /usr/local/letsencrypt
	$ sudo chown `whoami` /usr/local/letsencrypt
	$ cd /usr/local/letsencrypt
	$ git clone --depth 1 https://github.com/letsencrypt/letsencrypt .
	$ sudo ./letsencrypt-auto --help

## ドメイン名の設定

	$ export LE_TARGET_DOMAIN=example.domain.dom

## acme-challenge用の公開ディレクトリを用意する

	$ sudo mkdir -p -m 0755 /usr/local/letsencrypt/var/$LE_TARGET_DOMAIN/webroot
	$ ls -al /usr/local/letsencrypt/var

nginx の設定

	$ sudo sh -c "cat > /etc/nginx/conf.d/$LE_TARGET_DOMAIN.conf" << EOS
	server {
	   listen               80;
	   server_name          $LE_TARGET_DOMAIN;
	   location /.well-known {
	        root /usr/local/letsencrypt/var/$LE_TARGET_DOMAIN/webroot;
	   }
	}
	EOS

	$ sudo service nginx restart

## 証明書の作成・取得

	$ cd /usr/local/letsencrypt

	$ sudo ./letsencrypt-auto certonly --webroot \
	  --webroot-path /usr/local/letsencrypt/var/$LE_TARGET_DOMAIN/webroot \
	  -d $LE_TARGET_DOMAIN  

	$ sudo openssl dhparam -out /etc/nginx/dhparams.pem 2048

## https領域の設定

Forward Secrecy, HTTP Strict Transport Security, http2 に対応させ、192.168.1.1:80 を Proxyする。

	$ sudo sh -c "perl -nle \"s/###TARGET_DOMAIN###/$LE_TARGET_DOMAIN/g; print\" > /etc/nginx/conf.d/$LE_TARGET_DOMAIN.conf" << 'EOS'
	
	server {
	   listen               80;
	   server_name          ###TARGET_DOMAIN###;
	   server_tokens        off;
	   location /.well-known {
	      root /usr/local/letsencrypt/var/###TARGET_DOMAIN###/webroot;
	   }
	   location / {
	      rewrite ^/(.+)$ https://###TARGET_DOMAIN###/$1 permanent;
	   }
	}
	
	server {
	   listen                    443 ssl http2;
	   server_name               ###TARGET_DOMAIN###;
	   server_tokens             off;
	
	   ssl                       on;
	   ssl_prefer_server_ciphers on;
	   ssl_session_cache         shared:SSL:10m;
	   ssl_session_timeout       10m;
	   ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
	   ssl_stapling              on;
	
	   ssl_ciphers 'kEECDH+ECDSA+AES128 kEECDH+ECDSA+AES256 kEECDH+AES128 kEECDH+AES256 kEDH+AES128 kEDH+AES256 DES-CBC3-SHA +SHA !aNULL !eNULL !LOW !kECDH !DSS !MD5 !EXP !PSK !SRP !CAMELLIA !SEED';
	
	   add_header Strict-Transport-Security "max-age=15768000";
	   ssl_certificate     /etc/letsencrypt/live/###TARGET_DOMAIN###/fullchain.pem;
	   ssl_certificate_key /etc/letsencrypt/live/###TARGET_DOMAIN###N/privkey.pem;
	   ssl_dhparam         /etc/nginx/dhparams.pem;
	
	   location / {
	      proxy_set_header Host                 $http_host;
	      proxy_set_header X-Real-IP            $remote_addr;
	      proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for;
	      proxy_set_header X-Forwarded-Proto    $scheme;
	      proxy_set_header X-Forwarded-Protocol $scheme;
	      proxy_pass http://192.168.1.1:80;
	
	   }
	}
	
	EOS

	$ sudo service nginx restart

## 証明書の自動更新

証明書は90日で失効するので、月に一回自動更新仕掛けを作る。月のランダムな1日の3時の何分かに実行。

	$ perl -e "printf('%02d %02d %02d * * root /usr/local/letsencrypt/letsencrypt-auto renew --force-renew && service nginx restart'.qq{\n},int(rand()*59),3,int(rand()*30)+1)" > /tmp/letsencrypt

	$ cat /tmp/letsencrypt
	28 03 12 * * root /usr/local/letsencrypt/letsencrypt-auto renew --force-renew && service nginx restart

	$ sudo mv /tmp/letsencrypt /etc/cron.d/
	$ sudo chown root:root /etc/cron.d/letsencrypt
	$ sudo chmod 644 /etc/cron.d/letsencrypt

