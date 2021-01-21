# Onion address with words

To look for an onion address with words to the following



1. Install Tor and a dictionary
```
sudo apt install -y tor wamerican-small
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

3. Run
```
grep '....' /usr/share/dict/words > /tmp/words  # 4 letter words, anything larger will take a very long time
while :; do sudo rm -rf /var/lib/tor/hidden_service_tmp/ &&  sudo service tor restart && sleep 2 && sudo cat /var/lib/tor/hidden_service_tmp/hostname && sleep 2 && sudo cat /var/lib/tor/hidden_service_tmp/hostname | sed 's/.onion$//' | grep -Ff/tmp/words && break; done

```
