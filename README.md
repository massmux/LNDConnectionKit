# NodeConnectionKit

This is an howto and configuration files in order to best connect and use an external LND node (installed on a different server or on a cloud service like Voltage). What you need is:

- an external LND install or a Voltage node LND (not based on neutrino)
- a small external VPS in order to install what you need. This may be a fresh ubuntu install with docker and docker-compose
- a domain name you own which must point to the VPS IP address. let's say my.domain.com

## Setup firewall

on the VPS. Enable the firewall

```
sudo ufw allow to any proto tcp port 22
sudo ufw allow to any proto tcp port 443
sudo ufw enable
sudo ufw status
```

Allow connection only with SSH key authentication. Disallow password authentication and root login


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
snap install --classic certbot
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
LND_REST_ENDPOINT=https://xxx.voltageapp.io:8080/
LND_REST_CERT=""
LND_REST_MACAROON="ADMINMACAROONXXXX"
```
copy the docker-compose.yml and the .env file inside the lnbits directory. then:

```
cd lnbits
docker build -t lnbits .
```
finally run

```
docker-compose up -d
```

### Configure nginx to proxy to LNBITS to my.domain.com

in /etc/nginx/sites-available/default

- empty the file, meaning that the file must be present but completely blank
- now copy the example file my.domain.com.conf in the directory /etc/nginx/sites-enabled/ and make modifications you need

create symbolic links

```
ln -s /etc/nginx/sites-available/my.domain.com.conf /etc/nginx/sites-enabled/my.domain.com.conf
```
enable nginx

```
sudo systemctl enable nginx
sudo systemctl start nginx
```

now https://my.domain.com  will proxy the local lnbits install (lnbits must be running)

## Install and configure rebalance-lnd

```
docker pull rebalancelnd/rebalance-lnd:latest
```

Now put on the .bashrc file of your user (not root), the following:

```
alias rebalance-lnd="docker run --rm --network=host --add-host=host.docker.internal:host-gateway -it -v /home/dev/lnd:/root/.lnd rebalancelnd/rebalance-lnd --grpc XXXX.voltageapp.io:10009"
```

## Install and configure balance of satoshi

```
docker pull alexbosworth/balanceofsatoshis
```

Now put on the .bashrc file of your user (not root), the following:
```
alias bos="docker run -it --rm -v /home/dev/.bos:/home/node/.bos alexbosworth/balanceofsatoshis  --node=voltage"
```

### Run the telegram daemon

First of all create a bot on telegram with botfather and get the api. First time you run the command, you will be asked of the api key and then a connection conde is returned on the bot in order to connect to your node.

For definitely run, go in tmux

```
bos telegram --connect CONNECT_CODE
```

then exit the tmux with CTRL+b+d

to reenter tmux attach -t 0
