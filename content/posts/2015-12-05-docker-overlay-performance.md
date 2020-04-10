---
title: Overlay Network Performance of Docker
date: 2015-12-05
---

Since Docker 1.9, the multi-host networks can be used very easily. All you have to do is just setup your Swarm cluster and use `docker network create -d mynet` and voila: Your multi host network is ready. I want to do a benchmark of this new feature, however, the official multi host networking examples are in Virtualbox and local performance can be misleading. I created an image called `mustafaakin/alpine-iperf` just has the Alpine image plus iperf network benchmarking tool installed. Resulting image is very tiny, around 7 MB. I measured the speed of both a  Virtualbox and DigitalOcean Swarm cluster. Just ensure docker-machine and latest docker client is installed in your machine.


Creating Swarms
---------------

To create a Swarm using VirtualBox:

```bash
# Create Key Store Node
docker-machine create -d virtualbox mh-keystore-vb
docker $(docker-machine config mh-keystore-vb) run -d \
    -p "8500:8500" \
    -h "consul" \
    progrium/consul -server -bootstrap

# Create Primary Node
docker-machine create \
	-d virtualbox \
	--swarm --swarm-master \
	--swarm-discovery="consul://$(docker-machine ip mh-keystore-vb):8500" \
	--engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore-vb):8500" \
	--engine-opt="cluster-advertise=eth1:2376" \
	mhs-demo0-vb

# Create Secondary Node
docker-machine create -d virtualbox \
    --swarm \
    --swarm-discovery="consul://$(docker-machine ip mh-keystore-vb):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore-vb):8500" \
    --engine-opt="cluster-advertise=eth1:2376" \
  	mhs-demo1-vb
```

To do in DigitalOcean: (we need a fairly new kernel for VXLAN, default value is Ubuntu 14.04 but its kernel is fairly old)

```bash
// replace it with your own token
TOKEN=aaa12a00f2f264727a4c5436c070d58477cd6d5f13a0379955901c4a962e9ee2 

# Create Key Store Node
docker-machine create --driver digitalocean \
	--digitalocean-access-token=$TOKEN \
	--digitalocean-region=ams3 mh-keystore-do		
docker $(docker-machine config mh-keystore-do) run -d \
    -p "8500:8500" \
    -h "consul" \
    progrium/consul -server -bootstrap    

# Create Primary Node
docker-machine create \
	--driver digitalocean \
	--digitalocean-access-token=$TOKEN \
	--digitalocean-region=ams3  \
	--digitalocean-image=debian-8-x64 \
	--swarm --swarm-master \
	--swarm-discovery="consul://$(docker-machine ip mh-keystore-do):8500" \
	--engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore-do):8500" \
	--engine-opt="cluster-advertise=eth0:2376" \
	mhs-demo0-do

# Create Secondary Node
docker-machine create \
	--driver digitalocean \
	--digitalocean-access-token=$TOKEN \
	--digitalocean-region=ams3 \
	--digitalocean-image=debian-8-x64 \
	--swarm \
    --swarm-discovery="consul://$(docker-machine ip mh-keystore-do):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore-do):8500" \
    --engine-opt="cluster-advertise=eth0:2376" \
  	mhs-demo1-do
```

With above configs, we create 3 machines, 1 key-value store (consul) for discovery and 1 Swarm master, and 1 additional Swarm node. To use them, you just point to the right swarm master. If you want to point to local Swarm master, all you have to do is the following:

```bash
$ eval $(docker-machine env --swarm mhs-demo0-vb)
$ docker info
Containers: 60
Images: 7
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 2
 mhs-demo0-vb: 192.168.99.101:2376
  └ Containers: 16
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
 mhs-demo1-vb: 192.168.99.102:2376
  └ Containers: 44
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.021 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.1.13-boot2docker, operatingsystem=Boot2Docker 1.9.1 (TCL 6.4.1); master : cef800b - Fri Nov 20 19:33:59 UTC 2015, provider=virtualbox, storagedriver=aufs
CPUs: 2
Total Memory: 2.043 GiB
Name: 8d66756b3509
```

And also for DigitalOcean the following is enough:

```bash
$ eval $(docker-machine env --swarm mhs-demo0-do)
$ docker info
Containers: 3
Images: 2
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 2
 mhs-demo0-do: 188.166.113.176:2376
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 519.2 MiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.16.0-4-amd64, operatingsystem=Debian GNU/Linux 8 (jessie), provider=digitalocean, storagedriver=aufs
 mhs-demo1-do: 178.62.230.66:2376
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 519.2 MiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.16.0-4-amd64, operatingsystem=Debian GNU/Linux 8 (jessie), provider=digitalocean, storagedriver=aufs
CPUs: 2
Total Memory: 1.014 GiB
Name: 61b80edc00a0
```

**Note:** The reason I have chosen Debian Jessie is proper VXLAN functionality. We need a recent kernel and default value of DigitalOcean image uses 14.04 which has kernel 3.13 and updating it requires a reboot.


Running iperf network testing
-----------------------------

Steps from now on are not specific to DigitalOcean or Virtualbox (Besides the constraints, since I pin the containers to nodes!). You can use the above commands to switch your Swarm clusters. Now, lets try the network testing! To assess the limits, I have attached the host network directly to the containers, so we have an idea about the maximum performance we expect to achieve. 

```bash
docker run --name iperf_host -d -ti --net host \
	--env="constraint:node==mhs-demo0-do" \
	mustafaakin/alpine-iperf iperf -s 

# Wait a little
sleep 1

for run in {1..5}; do 
	docker run --net host --env="constraint:node==mhs-demo1-do" -ti \
	mustafaakin/alpine-iperf \
	iperf -c $(docker-machine ip mhs-demo0-do) -P 10 | grep SUM
done
``` 

On DigitalOcean it gives 932 MBits/sec on average of 5 runs, and in VirtualBox it is approximately 1.72 GBit/sec. 

To test Overlay networking, we need to create it through Swarm master, so it can setup proper bridges/interfaces on each node. Then the process is very similar to previous one, we just subsitute the host network for our network as a parameter, and we give the private IP address of iperf server container to iperf client container. A better solution would be a DNS embedded to Swarm, which is not out yet but some people are working on [their solutions](https://github.com/ahmetalpbalkan/wagl)

```bash
docker network create -d overlay mynet
docker run --name iperf_overlay -d -ti --net mynet \
	--env="constraint:node==mhs-demo0-do" \
	mustafaakin/alpine-iperf iperf -s 

# Wait a little
sleep 1

IP=$(docker inspect -f "{ { .NetworkSettings.Networks.mynet.IPAddress }}" iperf_overlay)
for run in {1..5}; do 
	docker run --net mynet --env="constraint:node==mhs-demo1-do" -ti \
	mustafaakin/alpine-iperf \
	iperf -c $IP -P 10 | grep SUM
done
```

It should be working! The speed of overlay network on VirtualBox slows down significantly, and it is approximately 896 Mbit/sec. On the other hand, speed on DigitalOcean is approximately 882 MBits/sec for 5 runs. As I have said before, the numbers in VirtualBox might be misleading of too many overheads and inefficient virtualization. 

So, we look at performance at DigitalOcean and it is approximately %5 slower compared to native networking on DigitalOcean! Pretty impressive. Also note that this is merely a full bandwith benchmark, in real applications the difference would probably will not be noticable. However it is another blog post topic :)