# Chapter 12.3: Adding SSH Hidden Services

In this chapter we will show you how to add a ssh hidden service to login remotelly using Tor. 

## Create SSH Hidden Services

To create new service you need to add some lines in your torrc file.

This should be under /etc/tor/torrc

Add this lines:

```
HiddenServiceDir /var/lib/tor/hidden-service-ssh/
HiddenServicePort 22 127.0.0.1:22
HiddenServiceAuthorizeClient stealth hidden-service-ssh
```

* HiddenServiceDir: indicates tor that you have a hidden service directory with the necessary configuration on path.
* HiddenServicePort:  indicates tor port to be used,  in SSH case is 22, if you want use other port you can change here.
* HiddenServiceAuthorizeClient: As it's name indicates authorize a client to connect to the hidden service. 

After add lines to tor file you need to restart tor service

```
sudo /etc/init.d/tor restart
```

After restart you should have three new files like this:

```
total 24
drwx--S--- 3 debian-tor debian-tor 4096 jul  1 18:39 ./
drwx--S--- 5 debian-tor debian-tor 4096 jul  1 18:39 ../
drwx--S--- 2 debian-tor debian-tor 4096 jul  1 18:39 authorized_clients/
-rw------- 1 debian-tor debian-tor   63 jul  1 18:39 hostname
-rw------- 1 debian-tor debian-tor   64 jul  1 18:39 hs_ed25519_public_key
-rw------- 1 debian-tor debian-tor   96 jul  1 18:39 hs_ed25519_secret_key
```
The file hostname contains your id onion.

Use this address to connect to your ssh hidden service like this:

```
torify ssh <your-username>@your_new_onion_id.onion
```




