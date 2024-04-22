# DockerSwarm set up Common NFS server for multinode

* Prepared minimum 3 server 
* `manager` with public IP `0.0.0.0`

```
manager ->	192.168.12.101
worker1 ->	192.168.12.103
worker2	-> 192.168.12.105
```

* Zone Domains with manager nodes public IP `0.0.0.0`

```
etldocker.arcapps.org
etltraefik.arcapps.org
etlportainer.arcapps.org
etlerp.arcapps.org
```


# Install Ubuntu Server with Docker Swarm Mode

```
//Use root user 
sudo su
apt update && apt upgrade -y
apt install openssh-server
ufw allow ssh
systemctl status ssh
init 6
```

# Starting the docker swarm installation for manager node

```
export USE_HOSTNAME=etldocker.arcapps.org
echo $USE_HOSTNAME > /etc/hostname
hostname -F /etc/hostname
apt update && apt upgrade -y
apt install curl -y
curl -fsSL get.docker.com -o get-docker.sh
CHANNEL=stable sh get-docker.sh
rm get-docker.sh
docker swarm init --advertise-addr=192.168.12.101
docker ps
//Post-installation steps for Linux: Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker $USER
//Log out and log back in so that your group membership is re-evaluated.
```


# Allow firewall port 

```
sudo ufw allow 2377/tcp
sudo ufw reload
```

# Get the join token 

```
docker swarm join-token worker
```

# Label the node for manager node

```
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add manager.data=true $NODE_ID

# Extract worker node IDs into variables
export WORKER_ID1=$(docker node ls --filter "role=worker" --format "{{.ID}}" | sed -n 1p)
export WORKER_ID2=$(docker node ls --filter "role=worker" --format "{{.ID}}" | sed -n 2p)

# Update each worker node with a specific label
docker node update --label-add worker1.data=true $WORKER_ID1
docker node update --label-add worker2.data=true $WORKER_ID2

```

# Add following section to run manager

```
    deploy:
      placement:
        constraints:
          - node.labels.manager.data == true
          # - node.role==manager

```          

# Add following section to run worker1

```
    deploy:
      placement:
        constraints:
          - node.labels.worker1.data == true
          # - node.role==worker1
```          

# Add following section to run worker1

```
    deploy:
      placement:
        constraints:
          - node.labels.worker2.data == true
          # - node.role==worker2
```  

# Traefik Proxy with HTTPS

```
docker network create --driver=overlay traefik-public
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
export EMAIL=azmin@excelbd.com
export DOMAIN=etltraefik.arcapps.org
export USERNAME=admin
//AVOID all sorts of characters [~!@#$%, etc.] as a pass
export PASSWORD=0Xuk8MhkJWLwGj3
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
//Deploy the stack
curl -L dockerswarm.rocks/traefik-host.yml -o traefik-host.yml
docker stack deploy -c traefik-host.yml traefik
docker stack ps traefik
```


# Portainer web user interface

```
export DOMAIN=etlportainer.arcapps.org
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add portainer.portainer-data=true $NODE_ID
//Create the Docker Compose file
curl -L dockerswarm.rocks/portainer.yml -o portainer.yml
//Deploy the stack
// to update/install the portainer to the community edition, edit the yaml and change portainer image tag to portainer/portainer-ce and deploy the stack.
docker stack deploy -c portainer.yml portainer
docker stack ps portainer
```


# Starting the docker swarm installation for worker1, worker2

```
apt update && apt upgrade -y
apt install curl -y
curl -fsSL get.docker.com -o get-docker.sh
CHANNEL=stable sh get-docker.sh
rm get-docker.sh
docker ps
//Post-installation steps for Linux: Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker $USER
//Log out and log back in so that your group membership is re-evaluated.
```


* 1. **Set up NFS Server `worker2`**
   

First, you need an NFS server that will host the shared volume. You can set up NFS on a separate server or one of the swarm nodes.


* Install NFS on the chosen server:

  
```
sudo apt update
sudo apt install nfs-kernel-server -y
```
* Create and Export the NFS Share:

```
sudo mkdir -p /var/nfs/docker  # Create a directory for Docker volumes
sudo chown nobody:nogroup /var/nfs/docker
```
* Edit the NFS export file to add the directory you want to share:

```
sudo nano /etc/exports

/var/nfs/docker 192.168.12.101(rw,sync,no_subtree_check) 192.168.12.103(rw,sync,no_subtree_check)

```
* Apply the changes by restarting NFS service:

```
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

* 2. **Prepare Docker Swarm Nodes**

Install NFS client utilities on manager and worker1 so they can mount NFS shares.

* Install NFS Client on each swarm node:

```
sudo apt update
sudo apt install nfs-common -y
```

* 3. **Deploy the Application with a Common Volume**


* We are going to deploy a mariadb stack, go to the worker2 | NFS node and run the command 

```
sudo mkdir -p /var/nfs/docker/mariadb
sudo chown nobody:nogroup /var/nfs/docker/mariadb
sudo chmod 777 /var/nfs/docker/mariadb
```

* Restart the NFS server 

```
sudo systemctl restart nfs-kernel-server
sudo exportfs -ra
```


* Allow port for `worker2`

```
sudo ufw allow 2049/tcp
sudo ufw allow 2049/udp
sudo ufw allow 111/tcp
sudo ufw allow 111/udp
sudo ufw allow 1110/tcp
sudo ufw allow 1110/udp
sudo ufw allow 20048/tcp
sudo ufw allow 20048/udp
sudo ufw reload
```

* Allow port `manager` and `worker1`

```
sudo ufw allow out 2049/tcp
sudo ufw allow out 2049/udp
sudo ufw allow out 111/tcp
sudo ufw allow out 111/udp
sudo ufw allow out 1110/tcp
sudo ufw allow out 1110/udp
sudo ufw allow out 20048/tcp
sudo ufw allow out 20048/udp
sudo ufw reload
```

Create a docker-compose.yml file for your application. Use the driver_opts to specify the NFS as the source for your volume.


```
version: '3.8'

services:
  your-service:
    image: your-image
    volumes:
      - general-volume:/path/in/container

volumes:
  general-volume:
    driver_opts:
      type: "nfs"
      o: "addr=<NFS-Server-IP>,nolock,soft,rw"
      device: ":/var/nfs/general"
```

* 4. **Deploy the Stack**
  
Initialize your swarm (if not already done) and deploy your stack:

* Initialize Swarm (if not already a swarm):

```
docker swarm init
```
* Deploy your application:
```
docker stack deploy -c docker-compose.yml your_app_name
```



This setup will allow your service to use a volume that is common across all nodes in the swarm, backed by NFS. Any service that mounts the general-volume will be able to read and write to the NFS share, providing a shared filesystem.

Please note that using NFS might not be suitable for all applications, especially those requiring high IOPS or low latency. For such cases, consider other solutions like Portworx or other cloud-native storage solutions designed for Kubernetes and Docker Swarm environments.


