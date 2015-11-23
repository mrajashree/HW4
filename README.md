# Homework 4 - Advanced Docker

Digital Ocean droplets have been used for all three parts. On every droplet, latest version of docker needs to be installed using this command:
```
curl -sSL https://get.docker.com/ | sudo bash
```

##File IO
On the new droplet, after installing docker, copy the Dockerfile from the repo in folder File_IO.
Contents of Dockerfile:
```
FROM ubuntu:14.04
MAINTAINER Rajashree

RUN DEBIAN_FRONTEND=noninteractive sudo apt-get -y update

RUN DEBIAN_FRONTEND=noninteractive sudo apt-get install -y socat
```
Using the Dockerfile create a new image:
```
sudo docker build -t img_socat .
```
1. These are the steps to run the 1st container, which will run a command that outputs to a file
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
2. Using socat: This socat command maps file access to read file container and expose over port 9001
```
socat tcp-l:9001,reuseaddr,fork system:'cat out.txt',nofork
```
3. These are the steps to create the 2nd container or the linked container, which will link to first and access data (out.txt) on it.
```
sudo docker run -td --name socat_cont2 --link socat_cont1:readlink img_socat
sudo docker exec -it socat_cont2 bash
sudo apt-get install curl
curl readlink:9001 result.txt
cat result.txt
```
After this, the file_io part is finished.

## Ambassador
1. We'll be using two hosts, that is two digital ocean droplets.
Install docker & docker compose on both hosts:
```
curl -sSL https://get.docker.com/ | sudo bash
sudo apt-get -y install python-pip
sudo pip install docker-compose
```
2. Docker-compose is being used to configure containers as follows:
Contents of docker-compose.yml on redis server side to configure the containers
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
Following command runs both the containers described in the docker-compose.yml file:
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
3. Four containers on two different hosts are running
4. Now, set/get operations can be performed. This is shown in the screencast.

## Deploy docker
Install git on the new droplet:
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

Start private registery on port 5000
```
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

Use these commands to create bare repo:
```
cd deploy/green.git
git init --bare
cd ..
cd blue.git
git init --bare
```

Add the following in App repo:
```
git remote add blue file:///root/deploy/blue.git
git remote add green file:///root/deploy/green.git
```

1. A git commit is building a new image (new_img) using the Dockerfile from the App repo (which gets copied to /root/deploy/blue-www/ & /root/deploy/blue-www/)
2. The image is pushed to local registry
3. App is being deployed on green_slice & blue_slice depending on the push
4. Docker pull, stop, rm and run commands are pulling from registery, stopping, and restarting containers.
All these commands are shown below as contents of the post-receive hooks
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
Change message in src/main.js in /App to see change in browser and then do git push again.

Link to screencast:
https://youtu.be/6sq4MzWp4BY
