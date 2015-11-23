#
##file io
install docker using docker.io:
```
curl -sSL https://get.docker.com/ | sudo bash
```

Contents of Dockerfile:
```
FROM ubuntu:14.04
MAINTAINER Rajashree

RUN DEBIAN_FRONTEND=noninteractive sudo apt-get -y update

RUN DEBIAN_FRONTEND=noninteractive sudo apt-get install -y socat
```

Using the Dockerfile
```
sudo docker build -t img_socat .
```
1st container(which will contain op file)
```
sudo docker run -td --name socat_cont1 img_socat
sudo docker exec -it socat_cont1 bash
sh socat_version.sh > out.txt
```
contents of socat_version.sh file:
```
#!/bin/bash

socat -V
```
Socat command to map access & expose on port 9001
```
socat tcp-l:9001,reuseaddr,fork system:'cat out.txt',nofork
```

2nd container (which will link to first)
```
sudo docker run -td --name socat_cont2 --link socat_cont1:readlink img_socat
sudo docker exec -it socat_cont2 bash
sudo apt-get install curl
curl readlink:9001 result.txt
cat result.txt
```

## ambassador
Install docker & docker compose on both hosts:
```
curl -sSL https://get.docker.com/ | sudo bash
sudo apt-get -y install python-pip
sudo pip install docker-compose
```

Contents of docker-compose.yml on redis server side:
```
redis:
  image: redis
redis_ambassador_server:
  image: svendowideit/ambassador
  links:
    - redis
  ports:
    - "6379:6379"
```
then run:
```
docker-compose up
```

Contents of docker-compose.yml on client side:
```
redis_ambassador_client:
  container_name: redis_ambassador_client
  image: svendowideit/ambassador
  environment:
    - "REDIS_PORT_6379_TCP=tcp://<server_IP>:6379"
  expose:
    - "6379"
  ports:
    - "6379:6379"
```
then run:
```
docker-compose up -d
sudo docker run -it --link redis_ambassador_client:redis --name redis_client relateiq/redis-cli
```

## deploy docker
```
sudo apt-get install git
```
Create directory structure:
deploy/
  /App (clone repo: git clone https://github.com/CSC-DevOps/App.git)
  /blue.git
  /blue-www
  /green.git
  /green-www
put main.js, package.json in src/ folder in /App

Start private registery on port 5000
```
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

```
cd deploy/green.git
git init --bare
cd ..
cd blue.git
git init --bare
```
post-receive in blue.git/hooks:
```
GIT_WORK_TREE=/root/deploy/blue-www/ git checkout -f

sudo docker rm new_img
sudo docker build -t new_img /root/deploy/blue-www/

sudo docker tag -f new_img localhost:5000/deploy_img:latest
sudo docker push localhost:5000/deploy_img:latest

sudo docker pull localhost:5000/deploy_img:latest
sudo docker stop $(sudo docker ps -a -f 'name=blue*' -q)
sudo docker rm $(sudo docker ps -a -f 'name=blue*' -q)

sudo docker run -p 8080:8080 -td --name blue_slice localhost:5000/deploy_img:latest
sudo docker exec -td blue_slice sh -c "node /src/main.js 8080"
```
Then do:
```
chmod +x post-receive
```

post-receive in green.git/hooks:
```
GIT_WORK_TREE=/root/deploy/green-www/ git checkout -f

sudo docker rm new_img
sudo docker build -t new_img /root/deploy/green-www/

sudo docker tag -f new_img localhost:5000/deploy_img:latest
sudo docker push localhost:5000/deploy_img:latest

sudo docker pull localhost:5000/deploy_img:latest
sudo docker stop $(sudo docker ps -a -f 'name=green*' -q)
sudo docker rm $(sudo docker ps -a -f 'name=green*' -q)

sudo docker run -p 8090:8080 -td --name green_slice localhost:5000/deploy_img:latest
sudo docker exec -td green_slice sh -C "node /src/main.js 8080"
```
Then do:
```
chmod +x post-receive
```
Change message in src/main.js in /App to see change in browser and then do commit

Dockerfile from the workshop:
```
FROM    centos:centos6

# Enable EPEL for Node.js
RUN     rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
# Install Node.js and npm
RUN     yum install -y npm

# Bundle app source
COPY . /src
# Install app dependencies
RUN cd /src; npm install

EXPOSE  8080
CMD ["node", "/src/main.js", "8080"]
```

then:
```
git remote add blue file:///root/deploy/blue.git
git remote add green file:///root/deploy/green.git
```

Link to screencast:
https://youtu.be/6sq4MzWp4BY
