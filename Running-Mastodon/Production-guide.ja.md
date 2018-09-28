# Mastodon Production Guide 日本語版

**Disclaimer:**

このインストールガイドは、 [Ubuntu Server 18.04](https://www.ubuntu.com/server)用に作成されています。 他のOSを利用している場合、予期せぬ問題が発生するかもしれません。 他の環境用のインストールガイドの作成もお待ちしています。

また、このドキュメントは、十分に高度なLinuxサーバーの管理技術を有していることを期待して作成されています。

もしインスタンスのセットアップに協力が必要なら、 [#MastoAdmins](https://mastodon.social/tags/mastoadmins) タグをつけてトゥートしてみてください。

**日本語版ドキュメント作者より:**

この翻訳には個人的な感覚に基づく意訳が含まれます。また、最新のドキュメントとは内容が異なる恐れがありますので、[本家ドキュメント](https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Production-guide.md)も一緒に確認してください。

翻訳などに誤りがあると思われる場合は、プルリクエストを送るか、 [@ars42525@odakyu.app](https://odakyu.app/@ars42525)にリプライを送ってください。

## このガイドは何？

このガイドは、 [Mastodon](https://github.com/tootsuite/mastodon/) のインスタンスをセットアップする手順を説明しています。

このガイドでは example.com をドメイン(サブドメイン)の代わりとして使用しています。 example.comをあなたのインスタンスのドメイン(サブドメイン)に適宜置き換えて作業してください。

## 前提環境

このガイドには次のものが必要です。

- [Ubuntu Server 18.04](https://www.ubuntu.com/server) のサーバー
- サーバーへのrootアクセス権限
- インスタンスに使用するドメイン(サブドメイン)

## DNS

作業前に、次のDNSレコードが追加されているとよいです。

追加するレコードは、

-  example.com宛のAレコード (IPv4アドレス)
-  example.com宛のAAAAレコード (IPv6アドレス)

> ### (任意)知っておくと便利なもの
>
> 作業中に `tmux` を使うとに便利です。
>
>
> `tmux`は、サーバーから切断されてしまった時だけでなく、複数のターミナル画面を開いてrootユーザーとmastodonユーザーを切り替えるためなどにも使用できます。
>
> [tmux](https://github.com/tmux/tmux/wiki) はパッケージマネージャーからインストールできます。
>
> ```sh
> apt -y install tmux
> ```

## 依存ソフトウェアのインストール

全ての依存ソフトウェアは、rootアカウントでインストールします。
```
sudo -i
```

## Ubuntu 18.04.1 LTS向けのUbuntuリポジトリの拡張

Ubuntu 18.04.1 LTS (not 18.04) Canonicalの.1-releaseから、multiverse,universe,
restrictedのリポジトリが/etc/apt/のsources.listから削除されました。これらのリポジトリを追加しないと、インストールに失敗します。追加するためには、次のコマンドを入力してください。

```add-apt-repository universe
add-apt-repository multiverse
add-apt-repository restricted
apt update
```

### node.jsのリポジトリ

要求されたバージョンの[node.js](https://nodejs.org/en/)をインストールするには、追加でリポジトリを追加する必要があります。

リポジトリを追加するためには、このスクリプトを実行します

```sh
apt -y install curl
curl -sL https://deb.nodesource.com/setup_8.x | bash -
```

これで[node.js](https://nodejs.org/en/)のリポジトリが追加されました。

###  Yarnのリポジトリ

[Mastodon](https://github.com/tootsuite/mastodon/)で使用される[Yarn](https://yarnpkg.com/en/)をインストールするためには、別のリポジトリを追加する必要があります。

これがリポジトリを追加する方法です。

```sh
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt update
```

### 他のいろいろな依存ソフトウェア

[Yarn](https://yarnpkg.com/en/)を、その他のソフトウェアをインストールします。

#### それぞれのソフトウェアの説明

- imagemagick - 画像の処理に利用されています
- ffmpeg - GIFをmp4に変換するのに利用されています
- libprotobuf-dev and protobuf-compiler - 言語の検出に利用されています
- nginx - nginxはフロントエンドのWebサーバーです
- redis-* - redisはインメモリデータ構造サーバーを提供します
- postgresql-* - SQLデータベースとして利用されています
- nodejs - NodeはストリーミングAPIに利用されています
- yarn - YarnはNode.js用のパッケージマネージャーです
- 他の-devとg++のパッケージは、ruby-buildを利用したRubyのビルドに必要です

```sh
apt -y install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm5 libgdbm-dev nginx redis-server redis-tools postgresql postgresql-contrib certbot yarn libidn11-dev libicu-dev
```

### 非rootユーザーでインストールする依存ソフトウェア

まずユーザーを作成します。

```sh
adduser mastodon
```

`mastodon`ユーザーとしてログインします。


```sh
sudo su - mastodon
```

[`rbenv`](https://github.com/rbenv/rbenv)と[`ruby-build`](https://github.com/rbenv/ruby-build)をセットアップします。

```sh
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
# shellを再起動
exec bash
# rbenvがインストールされているか確認する
type rbenv
# ruby-buildをrbenvのプラグインとしてインストールする
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

[`rbenv`](https://github.com/rbenv/rbenv)と[`ruby-build`](https://github.com/rbenv/ruby-build)がインストールされたので、[Mastodon](https://github.com/tootsuite/mastodon/)で要求されるバージョンの[Ruby](https://www.ruby-lang.org/en/)をインストールして、それを有効化します。

[Ruby](https://www.ruby-lang.org/en/)を有効化するには、このコマンドを使用します。

```sh
rbenv install 2.5.1
rbenv global 2.5.1
```

**このコマンドにはしばらくかかります。終わるまで軽くストレッチでもして水でも飲んでいてください。**

### node.jsとRubyの依存ソフトウェア

[Ruby](https://www.ruby-lang.org/en/)がインストールできたので、[MastodonのGitリポジトリ](https://github.com/tootsuite/mastodon/)をクローンして、[Ruby](https://www.ruby-lang.org/en/)と[node.js](https://nodejs.org/en/)の依存ソフトウェアをインストールします。

このコマンドでクローンとインストールをします。

```sh
# mastodonユーザーのホームディレクトリに戻ります
cd ~
# mastodonのGitリポジトリを~/liveにクローンします
git clone https://github.com/tootsuite/mastodon.git live
# ~/liveに移動します
cd ~/live
# 最新の安定版にcheckoutします
git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)
# bundlerをインストールします
gem install bundler
# bundlerを利用してRubyの依存ソフトウェアをインストールします
bundle install -j$(getconf _NPROCESSORS_ONLN) --deployment --without development test
# yarnを利用してnode.jsの依存ソフトウェアをインストールします
yarn install --pure-lockfile
```

これで`mastodon`ユーザーで行う作業は終わりです。`exit`でrootに戻ります。

## PostgreSQLデータベースの作成

[Mastodon](https://github.com/tootsuite/mastodon/)は[PostgreSQL](https://www.postgresql.org)インスタンスへのアクセス権が必要です。

[PostgreSQL](https://www.postgresql.org)インスタンスのユーザーを作成します。

```
# postgresユーザーでpsqlを起動します
sudo -u postgres psql

# 表示されるプロンプトに入力してください
CREATE USER mastodon CREATEDB;
\q
```

**Note** パスワードを設定していないのは、ident認証を使用しているからです。これによってローカルユーザーはパスワードなしでデータベースにアクセスできます。

## nginx Configuration

[Mastodon](https://github.com/tootsuite/mastodon/)インスタンス用に、[nginx](http://nginx.org)を設定します。

**注意: example.comを全てあなたのドメイン(サブドメイン)に置き換えてください**

`cd`で`/etc/nginx/sites-available`に移動して、次のコマンドで新規ファイルを開きます。

`nano /etc/nginx/sites-available/example.com.conf`

これをコピペして、必要に応じて適宜改変してください。

```nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name example.com;
  root /home/mastodon/live/public;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name example.com;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  keepalive_timeout    70;
  sendfile             on;
  client_max_body_size 80m;

  root /home/mastodon/live/public;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location ~ ^/(emoji|packs|system/accounts/avatars|system/media_attachments/files) {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri @proxy;
  }
  
  location /sw.js {
    add_header Cache-Control "public, max-age=0";
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://127.0.0.1:3000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  location /api/v1/streaming {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";

    proxy_pass http://127.0.0.1:4000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}
```

追加した[nginx](http://nginx.org)のコンフィグを有効化します。

```sh
cd /etc/nginx/sites-enabled
ln -s ../sites-available/example.com.conf
```

このコンフィグは、TLS証明書プロバイダとして[Let's Encrypt](https://letsencrypt.org)を利用することを前提にしています。

**もしTLS証明書プロバイダとしてLet's Encryptを利用するなら、次の項を読んでください。もしそうでなければ`ssl_certificate`と`ssl_certificate_key`の値を変更してください。**

## Let's Encrypt

この項は[Let's Encrypt](https://letsencrypt.org/)をTLS証明書プロバイダとして利用する時のみ実行してください

### 証明書の作成

Let's Encryptの証明書を作成します。

**全ての'example.com'をあなたのMastodonのドメインと置き換えてください。**

この時点で[nginx](http://nginx.org)を停止します。

```sh
systemctl stop nginx
```

ここでは、スタンドアローンモードでTLS SNI認証を利用して1回、webrootモードを利用してもう1回の計2回証明書を作成します。これは[nginx](http://nginx.org)と[Let's Encrypt](https://letsencrypt.org/)のツールを利用するために必要です。

```sh
certbot certonly --standalone -d example.com
```

これが完了したら、次はwebrootモードで実行します。これには[nginx](http://nginx.org)が起動している必要があります。

```sh
systemctl start nginx
# certbotツールが証明書を再生成するか質問してきたら、再生成を選択してください
certbot certonly --webroot -d example.com -w /home/mastodon/live/public/
```

### Let's Encrypt証明書の更新の自動化

[Let's Encrypt](https://letsencrypt.org/)の証明書は90日で期限切れになります。

期限切れになる前に証明書を更新する必要があります。更新しないと、インスタンスにアクセスできなくなり、他のインスタンスのユーザーがあなたのインスタンスと連合できなくなります。

このコマンドで毎日実行されるcronジョブを設定します。

```sh
nano /etc/cron.daily/letsencrypt-renew
```

このスクリプトをファイルにコピペします。

```sh
#!/usr/bin/env bash
certbot renew
systemctl reload nginx
```

保存してエディターを出てください。

スクリプトを実行可能にして、cronデーモンを再起動すると、スクリプトが毎日実行されるようになります。

```sh
chmod +x /etc/cron.daily/letsencrypt-renew
systemctl restart cron
```

これでサーバーは自動で[Let's Encrypt](https://letsencrypt.org/)の証明書を更新します。

## Mastodonの設定

次に、Mastodonの設定をします。

これは、`mastodon`ユーザーに切り替えて作業します。


```sh
sudo su - mastodon
```

ディレクトリ`~/live`に移動して、[Mastodon](https://github.com/tootsuite/mastodon/)セットアップウィザードを実行します。

```sh
cd ~/live
RAILS_ENV=production bundle exec rake mastodon:setup
```

インタラクティブなセットアップウィザードが基本的な設定や必要な設定をガイドして、シークレットキーを生成し、データベースをセットアップしてアセットファイルをプリコンパイルします。

**アセットのプリコンパイルにはしばらくかかるので、少し休憩しましょう。**

## Mastodonのsystemd用サービス設定ファイル

Mastodonには3つの[systemd](https://github.com/systemd/systemd)のサービスファイルが必要です。

ここでrootユーザーに戻ります。

[Mastodon](https://github.com/tootsuite/mastodon/)のwebワーカー用サービスを`/etc/systemd/system/mastodon-web.service`に作成します。

```
[Unit]
Description=mastodon-web
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="RAILS_ENV=production"
Environment="PORT=3000"
ExecStart=/home/mastodon/.rbenv/shims/bundle exec puma -C config/puma.rb
ExecReload=/bin/kill -SIGUSR1 $MAINPID
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

[Mastodon](https://github.com/tootsuite/mastodon/)のバックグラウンドサービスを `/etc/systemd/system/mastodon-sidekiq.service`に作成します。

```
[Unit]
Description=mastodon-sidekiq
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="RAILS_ENV=production"
Environment="DB_POOL=5"
ExecStart=/home/mastodon/.rbenv/shims/bundle exec sidekiq -c 5 -q default -q push -q mailers -q pull
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

[Mastodon](https://github.com/tootsuite/mastodon/)のストリーミングAPIのサービスを`/etc/systemd/system/mastodon-streaming.service`に作成します。

```
[Unit]
Description=mastodon-streaming
After=network.target

[Service]
Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment="NODE_ENV=production"
Environment="PORT=4000"
ExecStart=/usr/bin/npm run start
TimeoutSec=15
Restart=always

[Install]
WantedBy=multi-user.target
```

ここで、すべてのファイルを有効化します。

```sh
systemctl enable /etc/systemd/system/mastodon-*.service
```

サービスを起動します。

```sh
systemctl start mastodon-*.service
```

サービスがきちんと動作しているか確認します。

```sh
systemctl status mastodon-*.service
```

## リモートのメディアキャッシュの自動削除

Mastodonは他のインスタンスのメディアをダウンロードしてローカルにキャッシュします。このキャッシュは定期的に削除しないととても肥大化し、ディスク容量不足や、S3バケットの肥大化を引き起こします。

リモートのメディアキャッシュを削除するには、毎日cronジョブを実行することが推奨されます。 (これを、`crontab -e`でmastodonユーザーのcrontabに設定します。)

```sh
RAILS_ENV=production
@daily cd /home/mastodon/live && /home/mastodon/.rbenv/shims/bundle exec rake mastodon:media:remove_remote
```

このrakeタスクは、NUM_DAYSよりも古いキャッシュされた添付メディアを削除します。NUM_DAYSは特に指定しなければ7日(1週間)に設定されています。NUM_DAYSは環境変数なので、このように指定することも出来ます。

```sh
RAILS_ENV=production
NUM_DAYS=14
@daily cd /home/mastodon/live && /home/mastodon/.rbenv/shims/bundle exec rake mastodon:media:remove_remote
```

## メールサービス

もし、通知メールを受け取ったり、シングルユーザーモード以外で運用する予定なら、メールプロバイダを設定した方がよいでしょう。

いくつか無料のメールプロバイダが存在します。クレジットカードを登録すると10,000通のメールを無料で送信できる Mailgun.com や、.spaceのTLDを保有していればクレジットカードなしで15,000通のメールを送信できる Sparkpost.com などです。

このようなプロバイダを利用するなら、サブドメインを利用すると簡単です。この場合、あなたのドメインをプロバイダに登録するときに、"mail.domain.com"のようなドメインで登録します。

アカウントを作成したら、プロバイダの指示に従ってDNSレコードを変更します。全てのDNSの設定が完了して、プロバイダがDNSの設定を検証したら、コンフィグファイルを編集します。これらの設定は既にコンフィグの中に存在するはずですが、ここに設定のサンプルがあるので、置き換えて利用してください。

```
SMTP_SERVER=smtp.mailgun.org
SMTP_PORT=587
SMTP_LOGIN=anAccountThatIsntPostmaster@mstdn.domain.com
SMTP_PASSWORD=HolySnacksAPassword
SMTP_FROM_ADDRESS=Domain.com Mastodon Admin <notifications@domain.com>
```

最後に、メールをテストするためには、Railsのコンソールに入って(詳細は[管理ガイド](https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Administration-guide.md)を確認してください)、このコマンドを実行します。

```ruby
m = UserMailer.new.mail to:'email@address.com', subject: 'test', body: 'awoo'
m.deliver
```

これで完了です！全てが正常に完了したら、ブラウザで`https://example.com`にアクセスすると、[Mastodon](https://github.com/tootsuite/mastodon/)インスタンスが表示されます。

おめでとうございます！Fediverseへようこそ！
