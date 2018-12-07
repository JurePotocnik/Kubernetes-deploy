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
