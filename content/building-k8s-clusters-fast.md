+++
title = "Building K8s Clusters Fast"
date = 2024-05-17
+++

Spinning up a k8s cluster is easier than ever, but there are a ton of options and there are some trade offs. I tend nowadays to focus on the [k3s](https://k3s.io/) stack as there are a number of ways to use it.

* [Generic Install](https://k3s.io/) which works everywhere except Mac
* Dev focused on your notebook using [k3d](https://k3d.io/v5.6.3/) 
* Building a cluster [k3sup](https://github.com/alexellis/k3sup) which is what I'm going to cover here

## Spinning up cloud instances

The gcloud command makes things a snap, typically I use the console to build a command before adding it to a script to spin up several nodes, but roughly something like this will work

```bash
  #!/bin/bash

  if [ "$1" = "" ]
  then
    echo "Usage: spawn-node.sh <node name>"
    exit
  fi

  NODE=$1
  gcloud compute instances create \
      $NODE --project=project-abc\
      --zone=us-west1-c \
      --machine-type=e2-standard-4 \
      --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=MY_SUBNET\
      --maintenance-policy=MIGRATE \
      --provisioning-model=STANDARD \
      --service-account=<SERVICE_ACCOUNT>-developer.gserviceaccount.com \
      --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
      --create-disk=auto-delete=yes,boot=yes,device-name=$NODE,image=projects/debian-cloud/global/images/debian-12-bookworm-v20240415,mode=rw,size=100,type=projects/project-abc/zones/us-west1-c/diskTypes/
      --no-shielded-secure-boot \
      --shielded-vtpm \
      --shielded-integrity-monitoring \
      --reservation-affinity=any
```

Then go ahead and spin up 3 nodes:

```bash
spawn-node.sh k8s-master
spawn-node.sh k8s-worker-1
spawn-node.sh k8s-worker-2
```

### Install k3sup 

Since this documentation may not age well check out [the k3sup docs](https://github.com/alexellis/k3sup) for the latest information, but this was correct as of May 21st, 2024.
 
```bash
# install k3sup
curl -sLS https://get.k3sup.dev | sh
sudo chmod +x k3sup
sudo mv k3sup /usr/local/bin/
```

Then write the following script to install the cluster

```bash
#!/bin/bash


if [ "$1" = "" ]
then
  echo "Usage: launch-k3s.sh <master ip> <comma separated list of node ips> <node user>"
  exit
fi

if [ "$2" = "" ]
then
  echo "Usage: launch-k3s.sh <master ip> <comma separated list of node ips> <node user>"
  exit
fi

if [ "$2" = "" ]
then
  echo "Usage: launch-k3s.sh <master ip> <comma separated list of node ips> <node user>"
  exit
fi

MASTER_IP=$1
NODE_IPS=$2
USER=$3

k3sup install --ip $MASTER_IP --user $USER
IFS=', ' read -r -a array <<< "$NODE_IPS"

for NODE_IP in "${array[@]}"
do
    k3sup join --ip $NODE_IP --server-ip $MASTER_IP --user $USER
done
```
Then once done, you can simply install your cluster with a simple call (assuming master of 192.168.1.100 and workers of 192.168.1.101, 192.168.1.102)

```bash
launch-k3s.sh 192.168.1.100 192.168.1.101,192.168.1.102 admin-user  
```
