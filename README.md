# Kubernetes deploy
 -  -  - 
In this short guideline I will explain how to setup kubernetes with dashboard and private registry for it.

## Docker
Docker is "container" technology. It helps you run virtual servers with preinstalled programs, configurations etc.
On your computer you can run multi docker images with different OS in it. If you want you can forward ports from each contaner to your externa port. In this example I will show you how to expose docker port 80(nginx) to external custom port.

### Registry
If we want to install ubuntu in docker container we need to give him a path to the repository where to download the image. if we have our cusotm code in it or some private configuration we can but private registry on https://hub.docker.com/ or install our private one.

### Installation
In this tutorial all OS will be running ubuntu 18.04. 
First we install OS to one of our servers. Than we need to install docker to run registry
```
$ apt-get update && apt-get install -y apt-transport-https
$ curl -s https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ add-apt-repository [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
$ apt update && apt install -qy docker-ce
```

#### Running on https
If we want to have registry on HTTPS install create registry.key and registry.crt file.
Node: for key you have to have full chain certificate: 
```
cat cert.crt interm.crt > registry.crt
```

#### Running registry under username/password
Please do the security first. Add new htpasswd for registry. To create new user run command like:
```
docker run --entrypoint htpasswd registry:2 -Bbn Username Oasswird123 > htpasswd
```

To run registry you need to provide all paths to auth directory, certificate and so on. Some of the parameters are for port forwarding
```
sudo docker run -d \
  --restart=always \
  --name mySuperRegistry \
  -v /data/certs:/certs \
  -v /data/registry:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
  -v /data/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -p 443:443 \
  registry:2
```

This will download the image map all drives and started the image.
To check the results run:
```
sudo docker ps -a
```

### Registry frontend
Ofcourse we can't see if everything is ok wo if you want you can setup frontend to see what is happening. 
I used  konradkleine/docker-registry-frontend:v2 It's good enough for me
```
sudo docker run \
  -d \
  --name frontendOfMySuperRegistry \
  -e ENV_DOCKER_REGISTRY_HOST=dockerhub.mypage.com \
  -e ENV_DOCKER_REGISTRY_PORT=443 \
  -e ENV_DOCKER_REGISTRY_USE_SSL=1 \
  -e ENV_MODE_BROWSE_ONLY=false \
  -e ENV_USE_SSL=yes \
  -v /data/certs/registry.crt:/etc/apache2/server.crt:ro \
  -v /data/certs/registry.key:/etc/apache2/server.key:ro \
  -p 8080:80 \
  konradkleine/docker-registry-frontend:v2
```

At this point we can push the docker images to the registry. If you use the command line to create&push the image you could have get some error because of the authorization.
To login in private registry use:
```
docker login dockerhub.mypage.com
```
My running the command above you will need to insert username/password for the page. 

Some docker comamnds:
```
docker ps -a 
docker rm ImageName -f (-f is for force)
docker run ubuntu:18.04 
```

### Create test docker image for mongo nginx and php fpm
```
FROM ubuntu:18.04

RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get install -y vim software-properties-common
RUN add-apt-repository ppa:ondrej/php -y
RUN apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y php7.2 php7.2-cli  php7.2-fpm php7.2-dev php7.2-xml libcurl3-openssl-dev autoconf pkg-config libssl-dev nginx git
RUN pecl install mongodb-1.5.3


COPY ./nginxconfig.conf /etc/nginx/sites-available/default
RUN ln -sf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default

RUN echo "extension=mongodb.so" >> /etc/php/7.2/fpm/php.ini
RUN echo "extension=mongodb.so" >> /etc/php/7.2/cli/php.ini

RUN update-rc.d nginx enable
RUN update-rc.d php7.2-fpm enable


WORKDIR /var/www/

COPY ./data/ /var/www/
RUN php composer.phar update

CMD /etc/init.d/nginx start && \ 
	/etc/init.d/php7.2-fpm start && tail -f /dev/null
```

What it does:
 - Install ubuntu
 - Upgrade OS
 - Add php 7.2 repository (7.3 its not working in kubernetes :) )
 - Install php, fpm, nginx, git,..
 - From working directory copy config file to docker image 
 - Enable mongo drive in PHP
 - add nginx and fpm on startup
 - Set working dir as /var/www
 - Copy data from working directory to /var/www in docker
 - run php composer
 - start all service and keep it running (tail -f /dev/null)
   - In kubernetes the worker just shutted down my node because it thought it's done with it.  


If you want build the image use:
```
docker build -t mysuperimage .
```

To push to your registry:
```
docker push dockerhub.superdomain.com/mysuperimage:1
```

And to run it with mapping port 80 to my 80 port.
```
docker run -i  -p 80:80 -i -t mysuperimage:1 /bin/bash
```
If you want to can map you own directory to docker:
```
 docker run --rm -v c:/work/data:/var/www/ -i  -p 80:80 -i -t dockerhub.superdomain.com/mysuperimage:1 /bin/bash 
```

Note: If you are running windows there could be some problem if you are using Azure AD.
Folow this to fix this: https://tomssl.com/2018/01/11/sharing-your-c-drive-with-docker-for-windows-when-using-azure-active-directory-azuread-aad/
basicly you need local account for it.


## Install kubernetes
Soo finaly the kubernetes. On simple solution to give you a headache. Kubernetes is a "docker image for controlling other dockers". Not. But Yes. It has one master server who is connected to multiple nodees which are running different docker images. 
Let's first install docker master. 
```
sudo apt-get update && apt-get install -y apt-transport-https
sudo curl -s https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update && apt install -qy docker-ce
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
    > /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && apt-get install -y kubeadm kubelet kubectl
```

You need to install this kubeadm/kubelet and kubectl on every node you want to have. But for now you don't need this. You can install it later.

To create the master for the kubernetes run command like this:
```
sudo kubeadm init --apiserver-advertise-address=192.168.5.135
```
For apiserver pust your IP of the server. After a while you will get join command. Please copy it to somewhere or if you lose it run this command to print it again:
```
kubeadm token create --print-join-command
```

Now we have running kubernetes. To be able to communicate with it you need to set your private config:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Install the nodes
Nodes are just simple servers who will run the image which kubernetes told them to.
Install all necessary libraries and then run the join command from command above:
```
kubeadm join 192.168.5.135:6443 --token psncol.ghljaklsjdhfsjfbn --discovery-token-ca-cert-hash sha256:85835a9d6faf06eljdfgkljdfklfgsdnlajsd6eb32c0d83ba193b181ac07d6
```

WIP. how to configure network etc

