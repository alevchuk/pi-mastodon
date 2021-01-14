# ln-mastodon
Mastodon on Lightning


## Build Mastodone

Similar to https://docs.joinmastodon.org/admin/install/ - yet more paranoid

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
