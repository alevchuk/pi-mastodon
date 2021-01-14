# ln-mastodon
Mastodon on Lightning

## Get hardware

The powerful Pi 4 with plenty of RAM removing the need for swap.

Total **296 USD** as of 2020-12-09

* Pi 4 kit (8GB RAM, heat sinks, power supply): [CanaKit Raspberry Pi 4 Basic Kit 8GB RAM](https://camelcamelcamel.com/product/B08DJ9MLHV)
* FLIRC Passive cooling case [Flirc Raspberry Pi 4 Case](https://camelcamelcamel.com/Flirc-Raspberry-Pi-Case-Silver/product/B07WG4DW52)
* Micro SD card 32G (for operating system) [SanDisk-Extreme-microSD-UHS-I-Adapter](https://camelcamelcamel.com/product/B06XWMQ81P)
* Card Reader (for 1 time setup) [Transcend-microSDHC-Reader-TS-RDF5K-Black](https://camelcamelcamel.com/Transcend-microSDHC-Reader-TS-RDF5K-Black/product/B009D79VH4)

If you want a Raid mirror for data protection follow https://github.com/alevchuk/minibank/blob/first/README.md#hardware

# Install operating system and check temperature

1. https://github.com/alevchuk/minibank/blob/first/README.md#operating-system
2. https://github.com/alevchuk/minibank/blob/first/README.md#first-time-login
3. https://github.com/alevchuk/minibank/blob/first/README.md#heat

## Install prereqisits
```
sudo apt install -y debootstrap schroot
```

## Get 64-bit environment

You'll need 64-bit dependency binaries so lets setup schroot

For account that has `sudo` run:
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

sudo schroot -c mastodon64 -- apt install git

sudo mkdir -p /mnt/mastodon/pi64/mnt/mastodon
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/src
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/gocode
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/bin

sudo chown -R mastodon /mnt/mastodon/pi64/mnt/mastodon
```

## Setup server

This is similar to https://docs.joinmastodon.org/admin/install/ - yet more paranoid

1. Get a Pi and Setup network:
https://github.com/alevchuk/minibank/blob/first/README.md#network

2. Create user:
```
sudo adduser --disabled-password mastodon  # when prompted press and hold Enter
```

3. Install Mastodon dependencies
```
sudo schroot -c mastodon64 -- apt install -y imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git \
  g++ libprotobuf-dev protobuf-compiler pkg-config gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  nginx redis-server redis-tools postgresql postgresql-contrib \
  certbot python-certbot-nginx yarn libidn11-dev libicu-dev libjemalloc-dev \
  python3.7 python3-distutils
```


4. Setup symlinks
```
sudo su -l mastodon
schroot -c mastodon64

ln -s /mnt/mastodon/src ~/src
ln -s /mnt/mastodon/gocode ~/gocode
ln -s /mnt/mastodon/bin ~/bin
```


## Build node.js and yarn 
* Prerequisit: you need to be logged in as "mastodon" and in `schroot -c mastodon64`

1. Build node.js (includes NPM)

```
git clone https://github.com/nodejs/node.git ~/src/node
cd ~/src/node
git fetch
git checkout $(git tag | grep v12 | sort -V | tail -n1)  # latest minor version of 12
./configure --prefix $HOME/bin
make  # negtive (-): this will take all day; postitive (+): building from source has transparency advantages
make install

```

2. Install Yarn:
```
npm install -g yarn
```


## Build Mastodone

1. Get source code:
* Prerequisit: you need to be logged in as "mastodon" and in `schroot -c mastodon64`

```
mkdir ~/src
git clone https://github.com/tootsuite/mastodon.git ~/src/mastodon
git checkout v3.3.0
```
