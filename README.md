
1. Create a VM instance on GCP with following
  
Requirement  | Specification  
------------ | -------------
CPU | 4
Memory | 15 GB
OS | Ubuntu 18.04 LTS
Disk | 30 GB

2. ssh to the machine.

3. Install wetty and set up user "trainee" with password "P@ssword"
```bash 

sudo su
apt -y update

#install wetty - Terminal over HTTP and HTTPS
apt -y install npm
npm install -g wetty


adduser --disabled-password --gecos "" trainee
echo -e "PASSWORD\nPASSWORD" | sudo passwd trainee 
usermod -aG sudo trainee 
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
systemctl restart sshd

```
4. install docker-ce 

```bash
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

apt-key fingerprint 0EBFCD88
   
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update

apt-get -y install docker-ce

```

5. Set the Docker training env

```bash

# create a docker subnet

echo 'nameserver 172.18.0.20' >  /opt/redislabs/resolv.conf
docker network create --subnet=172.18.0.0/16 redislabs

# create the DNS 
docker run --name bind -d -v /opt/redislabs/resolv.conf:/etc/resolv.conf  --net redislabs --restart=always -p 10000:10000/tcp   --ip 172.18.0.20 rahimre/redislabs-training-bind


# create the north cluster

sudo docker run -d  --cap-add=ALL --name n1  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/n1/:/var/opt/redislabs/log -p 21443:8443 -p 41443:9443 --restart=always  --hostname  n1.north.redislabs-training.org --net redislabs --ip 172.18.0.21  redislabs/redis
sudo docker run -d  --cap-add=ALL --name n2  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/n2/:/var/opt/redislabs/log -p 22443:8443 -p 42443:9443 --restart=always  --hostname  n2.north.redislabs-training.org  --net redislabs --ip 172.18.0.22   redislabs/redis
sudo docker run -d  --cap-add=ALL --name n3  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/n3/:/var/opt/redislabs/log -p 23443:8443 -p 43443:9443 --restart=always  --hostname  n3.north.redislabs-training.org  --net redislabs --ip 172.18.0.23    redislabs/redis

sudo docker exec --user root n1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sudo docker exec --user root n2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sudo docker exec --user root n3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"


# create the south cluster

sudo docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/s1/:/var/opt/redislabs/log --name s1 -p 31443:8443 -p 51443:9443 --restart=always --hostname  s1.south.redislabs-training.org   --net redislabs --ip 172.18.0.31  redislabs/redis
sudo docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/s2/:/var/opt/redislabs/log --name s2 -p 32443:8443 -p 52443:9443 --restart=always --hostname s2.south.redislabs-training.org   --net redislabs --ip 172.18.0.32  redislabs/redis
sudo docker run -d  --cap-add=ALL  -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /opt/redislabs/log/s3/:/var/opt/redislabs/log --name s3 -p 33443:8443 -p 53443:9443 --restart=always  --hostname s3.south.redislabs-training.org   --net redislabs --ip 172.18.0.33   redislabs/redis

sudo docker exec --user root s1 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sudo docker exec --user root s2 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"
sudo docker exec --user root s3 bash -c "iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 5300"


#set up the ftp server
docker run -d -v /my/data/directory:/home/vsftpd -e FTP_USER=redis \
   -e FTP_PASS=redis --net redislabs  --ip 172.18.0.19 --name vsftpd  --restart=always \
   fauria/vsftpd
   
```



6. Set the default editor to vim

```
 update-alternatives --config editor
 ```
 
7. add the following line to /etc/sudoers using "sudo visudo" 

```
trainee ALL=(ALL) NOPASSWD:ALL

```

8. add the following to .bashrc

```
alias n1="sudo docker exec -it n1 bash "
alias n2="sudo docker exec -it n2 bash "
alias n3="sudo docker exec -it n3 bash "
alias s1="sudo docker exec -it s1 bash "
alias s2="sudo docker exec -it s2 bash "
alias s3="sudo docker exec -it s3 bash "
alias ns="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/dnsutils bash"
alias crl="sudo docker run -it --net redislabs   -v /opt/redislabs/resolv.conf:/etc/resolv.conf   tutum/curl bash"
alias redis="sudo docker run -v /opt/redislabs/resolv.conf:/etc/resolv.conf -v /home/trainee/redis:/data --net redislabs  -it redis bash "
```


9. Take a snapshot of the running instance and create training image called "redislabs-admin-training" from the snapshot

10. Go to Images on the GCP menu, find image "redislabs-admin-training" and create a new training instance.

11. You can log into the instance using ssh client 
```
  ssh trainee@publicip
```  
  
12. After login to ssh console, start the wetty deamon
```
   screen -d  -m wetty -p 21000 >/dev/null 2>&1
```   
   
   Now you can access the instance from browser 
   http://publicip:21000
   



