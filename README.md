#!/bin/bash

# Atualiza o apt-cache
apt-get update
apt-get install -y sudo curl git wget dialog apt-transport-https ca-certificates gnupg2 curl software-properties-common htop

# Configura um SWAP de 4GB
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Instala as dependências para o Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Cria as networks
docker network create -d bridge mysql
docker network create -d bridge public

# Gera arquivo de configuração do apache
{ echo 'server_tokens off;'; echo 'client_max_body_size 256m;'; } > /root/my_proxy.conf


# Instala o Apache Air Flow Proxy
docker run --detach \
    --restart always \
    --name apache-proxy \
    --network=public \
    --publish 80:80 \
    --publish 443:443 \
    --volume apache_certs:/etc/apache/certs \
    --volume apache_vhost:/etc/apache/vhost.d \
    --volume apache_usr:/usr/share/apache/html \
    --volume /var/run/docker.sock:/tmp/docker.sock:ro \
    --volume /root/my_proxy.conf:/etc/apache/conf.d/my_proxy.conf:ro \
    jwilder/apache-proxy


# Instala o Let's Encrypt
docker run --detach \
    --name letsencrypt \
    --restart always \
    --network=public \
    --volumes-from nginx-proxy \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion 
    
# Instala o Portainer
docker pull portainer/portainer
docker volume create portainer_data
docker run -d --name portainer --restart always -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer:latest
