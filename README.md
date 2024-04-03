# DockerSwarm set up Common NFS server for multinode

* 1. **Set up NFS Server**
   

First, you need an NFS server that will host the shared volume. You can set up NFS on a separate server or one of the swarm nodes.


* Install NFS on the chosen server:

  
```
sudo apt update
sudo apt install nfs-kernel-server
```
* Create and Export the NFS Share:

```
sudo mkdir -p /var/nfs/general
```
* Configure NFS to share this directory by editing /etc/exports and adding:
```
/var/nfs/general    *(rw,sync,no_subtree_check)
```
* Apply the changes by restarting NFS service:
```
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

* 2. **Prepare Docker Swarm Nodes**

Each node in the swarm needs to be able to mount the NFS share.

* Install NFS Client on each swarm node:

```
sudo apt update
sudo apt install nfs-common
```

* Mount the NFS Share on each node (ensure it's mounted on the same mount point, e.g., /mnt/nfs/general):

```
sudo mkdir -p /mnt/nfs/general
sudo mount <NFS-Server-IP>:/var/nfs/general /mnt/nfs/general
```

* 3. **Deploy the Application with a Common Volume**

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


