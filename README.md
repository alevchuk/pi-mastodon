# pi-mastodon
Mastodon on a Pi via Tor

Based to official [Mastodon instructions](https://docs.joinmastodon.org/admin/install/) - yet more paranoid, setup on Raspberry Pi, and made to work over Tor without SSL

Table of contents
=================

  * [1. Get hardware](#1-get-hardware)
  * [2. Install operating system and check temperature](#2-install-operating-system-and-check-temperature)
  * [3. Setup 64-bit capability](#3-setup-64-bit-capability)
  * [4. Get a Tor .onion address](#4-get-a-tor-onion-address)
  * [5. Install Mastodon dependencies inside schroot](#5-install-mastodon-dependencies-inside-schroot)
  * [6. Build node.js and yarn ](#6-build-nodejs-and-yarn)
  * [7. Install Ruby and Bundler](#7-install-ruby-and-bundler)
  * [8. Install PostgreSQL](#8-install-postgresql)
  * [9. Install Redis](#9-install-redis)
  * [10. Setup Mastodon](#10-setup-mastodon)
  * [11. Setup Nginx](#11-setup-nginx)
 
## 1. Get hardware

Total **91 USD** as of 2021-01-19

* Pi 4 kit (2GB RAM, heat sinks, power supply): [CanaKit Raspberry Pi 4 Basic Kit 2GB RAM](https://camelcamelcamel.com/product/B07TYK4RL8)
* FLIRC Passive cooling case [Flirc Raspberry Pi 4 Case](https://camelcamelcamel.com/Flirc-Raspberry-Pi-Case-Silver/product/B07WG4DW52)
* Micro SD card 32G (for operating system) [SanDisk-Extreme-microSD-UHS-I-Adapter](https://camelcamelcamel.com/product/B06XWMQ81P)
* Card Reader (for 1 time setup) [Transcend-microSDHC-Reader-TS-RDF5K-Black](https://camelcamelcamel.com/Transcend-microSDHC-Reader-TS-RDF5K-Black/product/B009D79VH4)

If you want a Raid mirror for data protection follow https://github.com/alevchuk/minibank/blob/first/README.md#hardware

## 2. Install operating system and check temperature

External links to minibank wiki:
1. [Operating System](https://github.com/alevchuk/minibank/blob/first/README.md#operating-system)
2. [First time login](https://github.com/alevchuk/minibank/blob/first/README.md#first-time-login)
3. [Heat](https://github.com/alevchuk/minibank/blob/first/README.md#heat)
4. [Netwrok](https://github.com/alevchuk/minibank/blob/first/README.md#network)
5. [Convenience Stuff](https://github.com/alevchuk/minibank/blob/first/README.md#convenience-stuff) - to make it comfortable

## 3. Setup 64-bit capability

For Mast to work you'll need 64-bit dependency binaries so lets setup a 64-bit Kernel and schroot (if you need to know what this does, read
https://medium.com/for-linux-users/how-to-make-your-raspberry-pi-4-faster-with-a-64-bit-kernel-77028c47d653):

1. Update the kernel and enable 64 bit mode:

First check if you aleady have this step done, run:
```
uname -a  # if you see "aarch64 GNU/Linux" then this step is done and you can skip this setep, and go to intalling debootstrap
```

Run:
```
sudo rpi-update  # there will be interactive prompt, press "y" to proceed
```

Reboot #1:
```
sudo reboot
```

Edit kernel parameters (use vi or if unfamiliar, use nano):
```
sudo vi /boot/config.txt
```

In the `[pi4]` section add:
```
arm_64bit=1
```

Reboot #2:
```
sudo reboot
```

Check:
```
uname -a  # you should you see "aarch64 GNU/Linux" at the end of the line
```

2. Install debootstrap and schroot
```
sudo apt install -y debootstrap schroot
```

3. Create mastodon user:
```
sudo adduser --disabled-password mastodon  # when prompted press and hold Enter
```

4. Form "admin" account (that has `sudo`) run:
```
sudo mkdir /mnt/mastodon
sudo chown -R mastodon /mnt/mastodon

cat << EOF | sudo tee /etc/schroot/chroot.d/mastodon64
[mastodon64]
description=builds that need 64-bit environment
type=directory
directory=/mnt/mastodon/pi64
users=mastodon
root-groups=root
profile=desktop
personality=linux
preserve-environment=true
EOF

sudo debootstrap --arch arm64 buster /mnt/mastodon/pi64

sudo schroot -c mastodon64 -- apt update
sudo schroot -c mastodon64 -- apt upgrade -y

sudo mkdir -p /mnt/mastodon/pi64/mnt/mastodon
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/src
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/gocode
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/bin
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/live

sudo chown -R mastodon /mnt/mastodon/pi64/mnt/mastodon
```

## 4. Get a Tor .onion address

1. Install Tor
```
sudo apt install -y tor
```

2. Edit /etc/tor/torrc (use vi if familiary, otherwise nano):
```
sudo vi /etc/tor/torrc
```
* at the end of the file add:
```
HiddenServiceDir /var/lib/tor/hidden_service_tmp/
HiddenServicePort 80 127.0.0.1:80
```

3. Run the following, you can run it multiple times - until you see an address that you like:
```
sudo rm -rf /var/lib/tor/hidden_service_tmp/ &&  sudo service tor restart && sleep 4 && sudo cat /var/lib/tor/hidden_service_tmp/hostname
```

Other options:
* [simple automation to look for onion addresses with words of 4 or more letters](https://github.com/alevchuk/ln-mastodon/blob/main/misc/onion-address-with-words.md)
* [various tools to generate pretty .onion addresses](https://security.stackexchange.com/questions/29772/how-do-you-get-a-specific-onion-address-for-your-hidden-service) - here your taking on a larger scurity risk because your using extra software that you technically don't need

4. Persist
```
sudo mv /var/lib/tor/hidden_service_tmp /var/lib/tor/hidden_service_mastodon

```

5. Change tor config to use the persisted version:
```
sudo vi /etc/tor/torrc
```
* at the end of the file change:
```
HiddenServiceDir /var/lib/tor/hidden_service_tmp/
```
* to:
```
HiddenServiceDir /var/lib/tor/hidden_service_mastodon/
```

6. Restart Tor and print your new hostname
```
sudo service tor restart

sudo service tor status  # check that it's running
sudo cat /var/lib/tor/hidden_service_mastodon/hostname  # print your .onion address

```

## 5. Install Mastodon dependencies inside schroot

1. From "admin" account run
```
sudo schroot -c mastodon64 -- apt install -y imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git \
  g++ libprotobuf-dev protobuf-compiler pkg-config gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  redis-tools \
  certbot python-certbot-nginx yarn libidn11-dev libicu-dev libjemalloc-dev \
  python3.7 python3-distutils \
  curl
```

2. Setup symlinks
```
sudo su -l mastodon
schroot -c mastodon64

ln -s /mnt/mastodon/src ~/src
ln -s /mnt/mastodon/gocode ~/gocode
ln -s /mnt/mastodon/bin ~/bin
ln -s /mnt/mastodon/live ~/live
```

3. Setup convenience

We already did convenience in the admin account (host operating system), now it's time to do the same inside the schroot

```
sudo schroot -c mastodon64

```

and go thru [Convenience Stuff](https://github.com/alevchuk/minibank/blob/first/README.md#convenience-stuff) - to make it comfortable inside the schroot. Yet:
* Skip "Name your Pi" and "Timezone"
* Don't include `sudo` in the commands


## 6. Build node.js and yarn 
* Prerequisit: you need to be logged in as "mastodon" followed by going into schroot:
```
sudo su -l mastodon
schroot -c mastodon64
```

1. Build node.js (includes NPM)

```
git clone https://github.com/nodejs/node.git ~/src/node
cd ~/src/node
git fetch
git checkout $(git tag | grep v12 | sort -V | grep -v  rc | tail -n1)  # latest minor version of 12
./configure --prefix $HOME/bin
make  # negtive (-): this will take all day; postitive (+): building from source has transparency advantages
make install

```

2. Add the following to `~/.profile`

```
export PATH=$HOME/bin/bin:$PATH

```

Load ~/.profile
```
. ~/.profile

```

2. Install Yarn:
```
npm install -g yarn
```


## 7. Install Ruby and Bundler

1. Install rbenv and rbenv-build:

```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.profile
echo 'eval "$(rbenv init -)"' >> ~/.profile
. ~/.profile
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

2. Install ruby
```
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 2.7.2
rbenv global 2.7.2
```

3. Install bundler
```
gem install bundler --no-document
```

4. Exist out of schroot
```
exit  # or press Ctrl-d
```

5. Return to admin user:
```
exit  # or press Ctrl-d
```


## 8. Install PostgreSQL

1. On "admin" account (not inside schroot), install default PostgreSQL version 11:
```
sudo apt install -y postgresql postgresql-contrib
```

2. Get PGTune parameters for you're RAM / Cores 
https://pgtune.leopard.in.ua/#/
* put PG Version 11
* 2 GB RAM (if you bought what's linked above)
* 4 CPU cores (if you bought what's linked above)

3. Add the tune parameters at the end of:
```
sudo vi /etc/postgresql/11/main/postgresql.conf
```

4. Restart PstgreSQL
```
sudo systemctl restart postgresql
```

5. Generate a random DB_PASSWORD:
```
openssl rand -base64 32 | sed 's/+//g' | tr '[A-Z]' '[a-z]' | tr -cd '[0-9a-z\n]'
```

6. Add DB user:

```
sudo -u postgres psql
```

and when prompted, paste the following line-by-line:
* replace DB_PASSWORD with the password you generated in setp 5
```
CREATE USER mastodon CREATEDB;
ALTER USER mastodon PASSWORD 'DB_PASSWORD';
\q
```

## 9. Install Redis

1. On "admin" account (not inside schroot), install default system Redis:
```
sudo apt install -y redis-server
```

## 10. Setup Mastodon
* Prerequisit: you need to be logged in as "mastodon" followed by going into schroot:
```
sudo su -l mastodon
schroot -c mastodon64
```

1. Get Mastodon source code:
```
git clone https://github.com/tootsuite/mastodon.git ~/live
cd ~/live
git fetch
git checkout $(git tag | grep v3.3 | sort -V | tail -n1)  # latest minor version of v3.3
```


2. Install Ruby and JavaScript dependencies
```
cd ~/live

bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN) 
yarn install --pure-lockfile
```

3. Run the setup wizard
* this will take a long time and interactively ask questions
```
RAILS_ENV=production bundle exec rake mastodon:setup  # if you are re-running this command AND want to destory current data and create an empty database, add DISABLE_DATABASE_ENVIRONMENT_CHECK=1

```
* Domain name: put your onion address from earlier step
* Single user mode: No
* Docker: No
* PostgreSQL host: localhost
* Port: Enter (uses the default)
* Name of PostgreSQL database: press Enter
* Name of PostgreSQL user: press Enter
* Password of PostgreSQL user: DB_PASSWORD from earlier step (password does not echo back, so just pasted it and press Enter)
* Redis host: press Enter
* Redis port: 6379
* Redis password: press Enter
* Do you want to store uploaded files on the cloud?:  press Enter
* Do you want to send e-mails from localhost? press Enter
* press Enter for many email related questions
* Send a test e-mail with this configuration right now? no
* press Enter for the rest of the questions

4. Write down your admin E-mail and password. Ok if you loose it - it's easy to re-create like this:
```
sudo su -l mastodon
schroot -c mastodon64
cd ~/live
RAILS_ENV=production ./bin/tootctl accounts create admin2 --role admin --email admin2@mast.com

```

5. Update mastodon config:
```
vi ~/.env.production
```
on top add:
```
HTTPS_KEY=off
SERVER_PROTOCOL=http
PORT=3001
BIND=127.0.0.1

LOCAL_DOMAIN=ONION_SITE_GOES_HERE
STREAMING_API_BASE_URL=http://ONION_SITE_GOES_HERE
CDN_HOST=http://ONION_SITE_GOES_HERE

```
* replace ONION_SITE_GOES_HERE with the onion address you generated earlier (e.g. a1b2c3.onion)

6. Start 3 mastodon services. Later, we'll setup these as systemd services that get restarted automatically if they crash. Yet, at this stage you'll need to learn how to use [multiple virtual windowns in Screen](https://www.youtube.com/watch?v=HomIzLB-HBc) and run all 3 services in parallel:

```
# in screen window 1
sudo su -l mastodon
schroot -c mastodon64
cd ~/live
PORT=3001 RAILS_ENV=production bundle exec rails s

# in sceen window 2
sudo su -l mastodon
schroot -c mastodon64
cd ~/live
RAILS_ENV=production DB_POOL=25 MALLOC_ARENA_MAX=2 /home/mastodon/.rbenv/shims/bundle exec sidekiq -c 25

# in screen window 3
sudo su -l mastodon
schroot -c mastodon64
cd ~/live
NODE_ENV=production PORT=4000 /home/mastodon/bin/bin/node ./streaming

```

## 11. Setup Nginx

1. On "admin" account (not inside schroot), install nginx
```
sudo apt install -y nginx

```

2. Create new config
```
sudo vi /etc/nginx/sites-available/mastodon

```

Paste the following, yet replace **ONION_SITE_GOES_HERE** with your .onion address generated at an earlier step (e.g. a1b2c3.onion)

```
upstream backend {
    server 127.0.0.1:3001 fail_timeout=0;
}

upstream streaming {
    server 127.0.0.1:4000 fail_timeout=0;
}

map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name ONION_SITE_GOES_HERE;

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

  add_header X-Frame-Options "DENY";

  add_header Content-Security-Policy "default-src 'none'; script-src 'self'; object-src 'self'; style-src 'self'; img-src 'self' data: blob: http://ONION_SITE_GOES_HERE; media-src 'self' data: http://ONION_SITE_GOES_HERE; frame-src 'none'; font-src 'self' data: http://ONION_SITE_GOES_HERE; frame-ancestors 'self'; form-action 'self'; base-uri 'self'; connect-src 'self' blob: wss://ONION_SITE_GOES_HERE";

  location / {
    try_files $uri @proxy;
  }
  
  location ~ ^/(emoji|packs|system/accounts/avatars|system/media_attachments/files) {
    try_files $uri @proxy;
  }

  location /sw.js {
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://backend;
    proxy_buffering on;
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
    proxy_set_header X-Forwarded-Proto http;
    proxy_set_header Proxy "";

    proxy_pass http://streaming;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }
  
  error_page 500 501 502 503 504 /500.html;
  access_log /var/log/nginx/mastodon_access.log;
  error_log /var/log/nginx/mastodon_error.log warn;
}
```

3. Edit top-level config:
```
sudo vi /etc/nginx/nginx.conf

```

Add:
```
    server_names_hash_bucket_size 65;
    
```

And comment out all setting that start with "ssl":
```

    #ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    #ssl_prefer_server_ciphers on;
```

4. Enable config:
```
sudo ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon

```

5. Restart nginx:
```
sudo systemctl restart nginx

```

## 12. Remove HTTPS from Mastodon

Tor does not need HTTPS. Moreover it does damage, signing SSL Certrficates is costly and cetralized (most authorities will not even sign a .onion address). Self-singned cettificated generate warnings that are very hard to bypass and get users acistomed to ignoring warnings that are imporatnt on clearnet.

Yet Mastadon is hardcoded to use HTTPS. So lets de-hardcode it.

1. Check that everything was installed correctly:
```
cd ~/live
git log -n1 # should say "commit 444b21b55ff5768e4cbbaf7cfa8285c65a4b54f9 (HEAD, tag: v3.3.0rc3)"

git status  # should say "nothing to commit, working tree clean"

sha256sum vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_controller/metal/redirecting.rb
# shuld say "da60d1e6315e4ef7e88ebb08a30b283cfcea588c0df3f610cd898f6b5fbd7ad9"

sha256sum vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb
# should say "cccb04f6a65890672fffc1b7a6fd7f9d55367e7e0bfc55521a2e5f334db7b06d"
```

2. If step 1 does not produce the correct hashes then the following step is probably not going to work. I encoruage you to reach out to me, send by the hashes you get, and I'll help you debug and produce a differnet patch if needed.

Copy the following lage command (all the way to, and including "EOF"), and run it: 
```diff

sudo su -l mastodon
schroot -c mastodon64
cd ~/live

patch --ignore-whitespace -p1 << 'EOF'
From cb5188b1c5146eb5cacd6b99a695c006b0fa7381 Mon Sep 17 00:00:00 2001
From: Your Name <alevchuk@example.com>
Date: Sat, 23 Jan 2021 14:21:59 +0000
Subject: [PATCH] HTTP patch

---
 app/controllers/accounts_controller.rb                      | 6 ++++++
 app/controllers/api/web/base_controller.rb                  | 2 +-
 app/controllers/application_controller.rb                   | 2 +-
 app/controllers/auth/sessions_controller.rb                 | 3 +++
 app/controllers/settings/sessions_controller.rb             | 3 ---
 app/models/user.rb                                          | 3 ++-
 config/environments/development.rb                          | 3 +++
 config/environments/production.rb                           | 2 ++
 config/initializers/1_hosts.rb                              | 2 +-
 config/initializers/devise.rb                               | 4 ++--
 config/initializers/session_store.rb                        | 2 +-
 config/navigation.rb                                        | 2 +-
 .../lib/action_controller/metal/redirecting.rb              | 2 +-
 .../gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb | 2 +-
 14 files changed, 25 insertions(+), 13 deletions(-)

diff --git a/app/controllers/accounts_controller.rb b/app/controllers/accounts_controller.rb
index b902ada09..4d9c1e2ae 100644
--- a/app/controllers/accounts_controller.rb
+++ b/app/controllers/accounts_controller.rb
@@ -1,6 +1,12 @@
 # frozen_string_literal: true

 class AccountsController < ApplicationController
+  force_ssl if: :ssl_configured?
+
+  def ssl_configured?
+    false
+  end
+
   PAGE_SIZE     = 20
   PAGE_SIZE_MAX = 200

diff --git a/app/controllers/api/web/base_controller.rb b/app/controllers/api/web/base_controller.rb
index 8da549b3a..6bb80f857 100644
--- a/app/controllers/api/web/base_controller.rb
+++ b/app/controllers/api/web/base_controller.rb
@@ -2,7 +2,7 @@

 class Api::Web::BaseController < Api::BaseController
   protect_from_forgery with: :exception
-
+
   rescue_from ActionController::InvalidAuthenticityToken do
     render json: { error: "Can't verify CSRF token authenticity." }, status: 422
   end
diff --git a/app/controllers/application_controller.rb b/app/controllers/application_controller.rb
index 44616d6e5..38865c16b 100644
--- a/app/controllers/application_controller.rb
+++ b/app/controllers/application_controller.rb
@@ -43,7 +43,7 @@ class ApplicationController < ActionController::Base
   private

   def https_enabled?
-    Rails.env.production? && !request.path.start_with?('/health')
+    false
   end

   def authorized_fetch_mode?
diff --git a/app/controllers/auth/sessions_controller.rb b/app/controllers/auth/sessions_controller.rb
index 13d158c67..1f0b2765d 100644
--- a/app/controllers/auth/sessions_controller.rb
+++ b/app/controllers/auth/sessions_controller.rb
@@ -9,6 +9,9 @@ class Auth::SessionsController < Devise::SessionsController
   skip_before_action :require_functional!
   skip_before_action :update_user_sign_in

+  # here123
+  skip_before_action :verify_authenticity_token
+
   include TwoFactorAuthenticationConcern
   include SignInTokenAuthenticationConcern

diff --git a/app/controllers/settings/sessions_controller.rb b/app/controllers/settings/sessions_controller.rb
index ee2fc5dc8..8a4804328 100644
--- a/app/controllers/settings/sessions_controller.rb
+++ b/app/controllers/settings/sessions_controller.rb
@@ -3,9 +3,6 @@
 class Settings::SessionsController < Settings::BaseController
   skip_before_action :require_functional!

-  before_action :require_not_suspended!
-  before_action :set_session, only: :destroy
-
   def destroy
     @session.destroy!
     flash[:notice] = I18n.t('sessions.revoke_success')
diff --git a/app/models/user.rb b/app/models/user.rb
index b4508c2eb..621e73931 100644
--- a/app/models/user.rb
+++ b/app/models/user.rb
@@ -131,7 +131,8 @@ class User < ApplicationRecord
   attr_writer :external, :bypass_invite_request_check

   def confirmed?
-    confirmed_at.present?
+    # No email confirmations on Tor, need lighting to make spam uneconomical
+    true
   end

   def invited?
diff --git a/config/environments/development.rb b/config/environments/development.rb
index 0791b82ab..8f3ecc73d 100644
--- a/config/environments/development.rb
+++ b/config/environments/development.rb
@@ -1,6 +1,9 @@
 Rails.application.configure do
   # Settings specified here will take precedence over those in config/application.rb.

+  config.force_ssl = false
+  config.x.use_https = false
+
   # In the development environment your application's code is reloaded on
   # every request. This slows down response time but is perfect for development
   # since you don't have to restart the web server when you make code changes.
diff --git a/config/environments/production.rb b/config/environments/production.rb
index aaad2449f..2e4eacfa0 100644
--- a/config/environments/production.rb
+++ b/config/environments/production.rb
@@ -1,5 +1,7 @@
 Rails.application.configure do
   # Settings specified here will take precedence over those in config/application.rb.
+  config.force_ssl = false
+  config.x.use_https = false

   # Code is not reloaded between requests.
   config.cache_classes = true
diff --git a/config/initializers/1_hosts.rb b/config/initializers/1_hosts.rb
index 757f1f735..159295095 100644
--- a/config/initializers/1_hosts.rb
+++ b/config/initializers/1_hosts.rb
@@ -7,7 +7,7 @@ web_host = ENV.fetch('WEB_DOMAIN') { host }
 alternate_domains = ENV.fetch('ALTERNATE_DOMAINS') { '' }

 Rails.application.configure do
-  https = Rails.env.production? || ENV['LOCAL_HTTPS'] == 'true'
+  https = false

   config.x.local_domain = host
   config.x.web_domain   = web_host
diff --git a/config/initializers/devise.rb b/config/initializers/devise.rb
index ef612e177..2c6258c71 100644
--- a/config/initializers/devise.rb
+++ b/config/initializers/devise.rb
@@ -9,7 +9,7 @@ Warden::Manager.after_set_user except: :fetch do |user, warden|
     value: session_id,
     expires: 1.year.from_now,
     httponly: true,
-    secure: (Rails.env.production? || ENV['LOCAL_HTTPS'] == 'true'),
+    secure: false,
     same_site: :lax,
   }
 end
@@ -20,7 +20,7 @@ Warden::Manager.after_fetch do |user, warden|
       value: warden.cookies.signed['_session_id'] || warden.raw_session['auth_id'],
       expires: 1.year.from_now,
       httponly: true,
-      secure: (Rails.env.production? || ENV['LOCAL_HTTPS'] == 'true'),
+      secure: false,
       same_site: :lax,
     }
   else
diff --git a/config/initializers/session_store.rb b/config/initializers/session_store.rb
index e5d1be4c6..4c30b7851 100644
--- a/config/initializers/session_store.rb
+++ b/config/initializers/session_store.rb
@@ -2,6 +2,6 @@

 Rails.application.config.session_store :cookie_store, {
   key: '_mastodon_session',
-  secure: (Rails.env.production? || ENV['LOCAL_HTTPS'] == 'true'),
+  secure: false,
   same_site: :lax,
 }
diff --git a/config/navigation.rb b/config/navigation.rb
index 4a56abe18..16c8b3029 100644
--- a/config/navigation.rb
+++ b/config/navigation.rb
@@ -54,6 +54,6 @@ SimpleNavigation::Configuration.run do |navigation|
       s.item :pghero, safe_join([fa_icon('database fw'), 'PgHero']), pghero_url, link_html: { target: 'pghero' }, if: -> { current_user.admin? }
     end

-    n.item :logout, safe_join([fa_icon('sign-out fw'), t('auth.logout')]), destroy_user_session_url, link_html: { 'data-method' => 'delete' }
+    n.item :logout, safe_join([fa_icon('sign-out fw'), t('auth.logout')]), destroy_user_session_url(protocol: 'http'), link_html: { 'data-method' => 'delete' }
   end
 end
diff --git a/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_controller/metal/redirecting.rb b/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_controller/metal/redirecting.rb
index 2804a06a5..3b0048cd7 100644
--- a/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_controller/metal/redirecting.rb
+++ b/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_controller/metal/redirecting.rb
@@ -103,7 +103,7 @@ module ActionController
       when /\A([a-z][a-z\d\-+\.]*:|\/\/).*/i
         options
       when String
-        request.protocol + request.host_with_port + options
+        "http://" + request.host_with_port + options
       when Proc
         _compute_redirect_to_location request, instance_eval(&options)
       else
diff --git a/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb b/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb
index 35ba44005..9e76a4ac5 100644
--- a/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb
+++ b/vendor/bundle/ruby/2.7.0/gems/actionpack-5.2.4.4/lib/action_dispatch/http/url.rb
@@ -205,7 +205,7 @@ module ActionDispatch
       #   req = ActionDispatch::Request.new 'HTTP_HOST' => 'example.com', 'HTTPS' => 'on'
       #   req.protocol # => "https://"
       def protocol
-        @protocol ||= ssl? ? "https://" : "http://"
+        @protocol ||= "http://"
       end

       # Returns the \host and port for this request, such as "example.com:8080".
--
2.20.1
EOF

```
