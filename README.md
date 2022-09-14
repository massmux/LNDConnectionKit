# LNDConnectionKit

This is an howto with example configuration files which goal is to best connect and use an external LND node (installed on a different server or on a cloud service like Voltage). In this howto we will suppose we are connecting to a Voltage node, but everything is similar in case of a connection to an external LND, no matter where and how it is configured. What you need is:

- An external LND install or a Voltage node LND (not those based on neutrino);
- A small VPS in order to install what you need. This may be a fresh ubuntu install with docker and docker-compose;
- A domain name you own which must point to the VPS IP address. let's say my.domain.com . You need to have the possiblity to modify DNS records in order to later authentify the letsencrypt certificate;

## Basic security on the VPS

### Enable the firewall

Allow Only SSH and https traffic.

```
sudo ufw allow to any proto tcp port 22
sudo ufw allow to any proto tcp port 443
sudo ufw enable
sudo ufw status
```

### Modify SSH configuration

- Allow connection only with SSH key authentication
- Disallow password authentication
- Disallow root login

### User configuration

if you are root then:

- add a standard user

run the following commands

```
adduser YOURUSER docker
adduser YOURUSER sudo
```

from now on, do everything as user YOURUSER. This user will be also able to start the docker instances.

## Node configuration files

in the VPS as unpriviledged user (into the docker group and sudoers)

```
mkdir -p ~/lnd/data/chain/bitcoin/mainnet/
```

download from voltage the following files

- tls.cert
- admin.macaroon

```
cp tls.cert ~/lnd/
cp admin.macaroon ~/lnd/data/chain/bitcoin/mainnet/
```

## Install prerequisites

```
sudo apt-get install nginx
sudo apt-get remove certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
ll /usr/bin/certbot
sudo certbot certonly --manual --preferred-challenges dns
```

With last command above you run the certbot in order to get the certs for your domain my.domain.com . At the end of process, you will have the cert installed in /etc/letsencrypt


## LNBITS: Install and configure

in ~ clone the repo

```
git clone https://github.com/lnbits/lnbits.git
```

modify the .env configuration file, adding the following lines. Please checkout the included example file

```
# BVB configuration
LNBITS_BACKEND_WALLET_CLASS=LndRestWallet
LND_REST_ENDPOINT=https://xxxx.voltageapp.io:8080/
LND_REST_CERT=""
LND_REST_MACAROON="ADMINMACAROONXXXX"
```
copy the docker-compose.yml and the .env file inside the lnbits directory.

Build the image

```
cd lnbits
docker build -t lnbits .
```

finally run the containers

```
docker-compose up -d
```

### Configure nginx to proxy to LNBITS to my.domain.com

This is in order to call the lnbits instance from a customized domain name on the clear net.

Consider the file: /etc/nginx/sites-available/default

- empty this file, meaning that the file must be present but completely blank;
- now copy the example file my.domain.com.conf in the directory /etc/nginx/sites-enabled/ and make modifications you need;

Create symbolic links

```
ln -s /etc/nginx/sites-available/my.domain.com.conf /etc/nginx/sites-enabled/my.domain.com.conf
```

Enable nginx

```
sudo systemctl enable nginx
sudo systemctl start nginx
```

Now https://my.domain.com  will proxy the local lnbits install (lnbits must be running)

## Install and configure rebalance-lnd

```
docker pull rebalancelnd/rebalance-lnd:latest
```

Now put on the .profile file of your user (not root), the following (for external LND or Voltage LND):

```
alias rebalance-lnd="docker run --rm --network=host --add-host=host.docker.internal:host-gateway -it -v /home/dev/lnd:/root/.lnd rebalancelnd/rebalance-lnd --grpc xxxx.voltageapp.io:10009"
```

## Install and configure Balance of Satoshis

A reminder: In this example the user we are using is `dev`, the home is then `/home/dev`.

Create bos directory:

```
mkdir ~/.bos
```

Now pull balanceofsatoshis' image from docker repository


```
docker pull alexbosworth/balanceofsatoshis
```

Now put on the .profile file of your user (not root), the following (in case of Voltage install):

```
alias bos="docker run -it --rm -v /home/dev/lnd:/home/node/.lnd -v /home/dev/.bos:/home/node/.bos alexbosworth/balanceofsatoshis  --node=voltage"
```


In case of Umbrel, the full command is different:

```
alias bos="docker run -it --rm --network=umbrel_main_network --add-host=bvb.local:10.21.21.9 -v /mnt/bvb-volume/umbrel/.bos:/home/node/.bos -v /mnt/bvb-volume/umbrel/app-data/lightning/data/lnd:/home/node/.lnd alexbosworth/balanceofsatoshis"
```

where the IP, is the IP you can find as LND_IP in file .env in the main dir of your Umbrel installation. Please also pay attention to the path to full data directory. In our case it was a VPS so it is `/mnt/bvb-volume/umbrel`. In case of raspberry it is different. Run `df -m` command to see the mounted partition's directory, this is the correct path


### Run the telegram daemon

install tmux

```
 sudo apt-get install tmux 
```

First of all create a bot on telegram with botfather and get the Api. First time you run the command, you will be asked of the api key and then a connection conde is returned on the bot in order to connect to your node.

For definitely run, go in tmux

```
bos telegram --connect CONNECT_CODE
```

- to exit the tmux with CTRL+b+d
- to reenter tmux attach -t 0


## Install and configure charge-lnd

```
docker pull accumulator/charge-lnd
```

Now create configuration file dir. Please bear in mind that in our case, the user is `dev` so ~ is `/home/dev`

```
mkdir ~/charge-lnd
```

Copy inside the just created dir the config file (charge.config). An example is in this repository

Now put on the .profile file of your user (not root), the following:

```
alias charge-lnd="docker run --rm -e LND_DIR=/data/.lnd -v /home/dev/charge-lnd:/app -v /home/dev/lnd:/data/.lnd -e GRPC_LOCATION=xxxx.voltageapp.io:10009 accumulator/charge-lnd"
```

## Install and configure LNDg


- Clone respository git clone https://github.com/cryptosharks131/lndg.git
- cd into the repo cd lndg
- set a docker-compose.yml like this:

```
version: "3.7"


services:
  lndg:
    build: .
    volumes:
      - /home/dev/lnd:/root/.lnd:ro
      - /home/dev/lndg/data:/lndg/data:rw
    command:
      - sh
      - -c
      - python initialize.py -net 'mainnet' -server 'XXXX.voltageapp.io:10009' -d && supervisord && python manage.py runserver 0.0.0.0:8889 
    network_mode: "host"

```

- Deploy your docker image: docker-compose up -d
- LNDg should now be available on port http://localhost:8889
- Open and copy the password from output file: nano data/lndg-admin.txt
- Use the password from the output file and the username lndg-admin to login

NB: to run in https, configure a nginx proxy to pass to 8889

TO BE CONTINUED ..
