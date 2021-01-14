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
sudo schroot -c mastodon64 -- apt install -y mesa-utils sudo

sudo mkdir -p /mnt/mastodon/pi64/mnt/mastodon
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/src
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/gocode
sudo mkdir /mnt/mastodon/pi64/mnt/mastodon/bin

sudo chown -R mastodon /mnt/mastodon/pi64/mnt/mastodon
```

## Build Mastodone

This is similar to https://docs.joinmastodon.org/admin/install/ - yet more paranoid

1. Get a Pi and Setup network:
https://github.com/alevchuk/minibank/blob/first/README.md#network

2. Create user:
```
sudo adduser --disabled-password mastodo  # when prompted press and hold Enter
```

3. Log-in as Mastodon:
```
sudo su -l mastodon
```

4. Build node.js and yarn
Build node.js (includes NPM)

```
mkdir ~/src
git clone https://github.com/nodejs/node.git ~/src/node
cd ~/src/node
git fetch
git checkout v13.7.0  # version higher then this will not build on the 32-bit rasbian
./configure --prefix $HOME/bin
make
make install

```

5. Install Yarn:
```
npm install -g yarn
```


6. Get source code:
```
mkdir ~/src
git clone https://github.com/tootsuite/mastodon.git ~/src/mastodon
git checkout v3.3.0
```
