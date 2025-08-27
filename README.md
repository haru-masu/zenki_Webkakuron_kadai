## AWS手順書
1. 事前準備
•	ブラウザ: FireFoxを使用してください。（Chromeのほうが良いです。）
•	AWSアカウント: 未作成の場合は、先に作成してください。
2. AWS Academy へのアクセス
1.	AWS Academy にアクセスしログイン。
2.	右上の 「Start Lab」 をクリック。
→ 数秒後、AWSロゴ横の赤丸が 緑色 に変わります。
3.	緑に変わったら、AWSロゴをクリックしてAWSコンソールへ遷移。
※作業終了時は 「End Lab」 を押してください。

3. EC2 インスタンス作成
1.	AWSコンソール右上から 「インスタンス起動」 をクリック。
2.	以下を設定:
o	名前: 任意（例: kadai_zenki）
o	OSイメージ: Amazon Linux
o	キーペア: 新規作成
	名前: 任意（例: kadai12）
	タイプ: RSA
	形式: .pem
	作成後、秘密鍵ファイルをダウンロードし安全に保存
o	ネットワーク: 「HTTPトラフィックを許可」 にチェック
o	ストレージ: 20GB
3.	「インスタンス起動」 をクリック。
4.	「すべてのインスタンスを表示」から対象インスタンスを選択し、パブリックIPv4アドレスをコピー。
4. SSH 接続
1.	Windowsで PowerShell を開く。
2.	ダウンロードした .pem ファイルのアクセス権を修正:
o	ファイルを右クリック → プロパティ → セキュリティ → 継承の無効化
o	「継承されたアクセス許可を明示的なアクセス許可に変換」を選択
o	ktc 以外のユーザーを削除 → 適用
3.	以下コマンドで接続（IPアドレスとpemファイルのパスを各自に合わせて変更）:
4.	ssh ec2-user@98.87.163.216 -i C:\Users\ktc\Downloads\kadai_zenki12.pem
5.	初回接続時の確認メッセージには yes と入力。
→ プロンプトが ec2-user に変われば接続成功。
5. Screen の導入
•	複数ターミナルを管理するために screen を使用。
1.	screen を起動。インストールされていない場合は以下を実行:
2.	sudo yum install screen -y
3.	screen
4.	設定ファイルを編集（見やすいように）:
5.	vim ~/.screenrc
以下を入力して保存:
hardstatus alwayslastline "%{= bw}%-w%{= wk}%n%t*%{-}%+w"
•	操作方法:
o	Ctrl + c: 新規ウィンドウ
o	Ctrl + p: ウィンドウ切替

6. Docker環境構築
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
Docker Compose インストール
sudo mkdir -p /usr/local/lib/docker/cli-plugins/
sudo curl -SL https://github.com/docker/compose/releases/download/v2.36.0/docker-compose-linux-x86_64 \
-o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version

7. プロジェクトディレクトリ作成
mkdir dockertest
cd dockertest
mkdir public kadai
kadai.php 作成
（本文略。画像投稿機能付きBBS を実装するPHPコードが含まれる）

8. Docker Compose 設定
compose.yml の例:
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
      mysqld --character-set-server=utf8mb4
             --collation-server=utf8mb4_unicode_ci
             --max_allowed_packet=4MB

volumes:
  mysql:
  image:
9. Nginx 設定
nginx/conf.d/default.conf:
server {
    listen       0.0.0.0:80;
    server_name  _;
    charset      utf-8;
    client_max_body_size 6M;
    root /var/www/public;

    location ~ \.php$ {
        fastcgi_pass  php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }

    location /image/ {
        root /var/www/upload;
    }
}

10. Dockerfile 作成
FROM php:8.4-fpm-alpine AS php
RUN docker-php-ext-install pdo_mysql
RUN install -o www-data -g www-data -d /var/www/upload/image/
RUN echo -e "post_max_size = 5M\nupload_max_filesize = 5M" >> ${PHP_INI_DIR}/php.ini

11. コンテナ起動
docker compose build
docker compose up
→ ブラウザで http://[EC2のIPアドレス] にアクセスし、
「Welcome to nginx!」 が表示されれば成功。

12. データベース準備
docker compose exec mysql mysql example_db
テーブル作成:
CREATE TABLE bbs_entries (
  id INT AUTO_INCREMENT PRIMARY KEY,
  body TEXT NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  image_filename VARCHAR(255) DEFAULT NULL
);
13. 再起動と動作確認
1.	ここまでの入力や設定がすべて完了したら、一度Dockerを停止します。
o	Ctrl + C で現在の docker compose up を終了してください。
2.	次のコマンドを順番に実行します:
3.	docker compose build
4.	docker compose up
o	docker compose build : コンテナを再構築
o	docker compose up : コンテナを起動
5.	ブラウザで以下のURLにアクセスします。
6.	http://[自分のパブリックIPアドレス]/[phpファイルまでのパス]
例:
http://54.92.192.95/kadai/kadai.php
7.	サイトに無事アクセスでき、アプリケーションが表示されれば 構築成功 です。

