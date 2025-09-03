# EC2 + Docker + Nginx + PHP + MySQL セットアップ手順

```bash
# ▼ EC2インスタンス作成
# キーペアが保存されているファイルを右クリックしてプロパティを開く
# セキュリティの欄の継承の無効化を押す
# 「継承されたアクセス許可をこのオブジェクトの明示的なアクセス許可に変換します。」を選択
# その後 ktc 以外のプリンシパルを削除し、適用する

ssh ec2-user@{IPアドレス} -i {秘密鍵ファイルのパス}

# ▼ docker および docker compose などのインストール
sudo yum install vim -y
sudo yum install screen -y
exit   # 再ログイン
screen
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
再起動

sudo mkdir -p /usr/local/lib/docker/cli-plugins/
sudo curl -SL https://github.com/docker/compose/releases/download/v2.36.0/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version

# ▼ 作業ディレクトリ作成
mkdir dockertest
cd dockertest

# ▼ Dockerfile 作成
cat << 'EOF' > Dockerfile
FROM php:8.4-fpm-alpine AS php
RUN docker-php-ext-install pdo_mysql
RUN install -o www-data -g www-data -d /var/www/upload/image/
RUN echo -e "post_max_size = 5M\nupload_max_filesize = 5M" >> ${PHP_INI_DIR}/php.ini
EOF

docker compose up --build

# ▼ compose.yml 作成
cat << 'EOF' > compose.yml
services:
  web:
    image: nginx:latest
    ports:
      - 80:80
    volumes:
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/
    depends_on:
      - php
  php:
    container_name: php
    build:
      context: .
      target: php
    volumes:
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/
  mysql:
    container_name: mysql
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: example_db
      MYSQL_ALLOW_EMPTY_PASSWORD: 1
      TZ: Asia/Tokyo
    volumes:
      - mysql:/var/lib/mysql
   command: >
      mysqld
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --max_allowed_packet=4MB
volumes:
  mysql:
  image:
EOF

docker compose up -d --build

# ▼ Nginx 設定ファイル作成
vim nginx/conf.d/default.conf

server {
    listen 80;
    server_name _;
    root /var/www/public;
    index index.html index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass php:9000;
    }
}
EOF

# ▼ public ディレクトリ作成
mkdir -p public/kadai
mkdir -p public/images

# ▼ kadai.php 作成
cat << 'EOF' > public/kadai/kadai.php
<?php
echo "Hello, Kadai!";
?>
EOF

# ▼ MySQL 接続してDB・テーブル作成
docker compose exec mysql mysql -u root example_db << 'EOSQL'
CREATE DATABASE IF NOT EXISTS example_db
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE example_db;
DROP TABLE IF EXISTS bbs_entries;
CREATE TABLE bbs_entries (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  body TEXT NOT NULL,                         -- 本文
  image_filename VARCHAR(255) DEFAULT NULL,   -- 画像ファイル名（任意）
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
EOSQL

docker compose up --build

# ▼ 動作確認
# 最後に以下をブラウザで開く
# http://パブリックIPアドレス/kadai/kadai.php
