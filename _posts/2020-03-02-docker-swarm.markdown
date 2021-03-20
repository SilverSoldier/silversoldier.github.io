---
layout: post
title:  "Deploying Hyperledger Fabric on Multiple Nodes using docker swarm"
description: Deploying Hyperledger Fabric on Multiple Nodes using docker swarm
img:
date: 2020-03-02  +1900
---

Recently, I had to deploy a Hyperledger Fabric setup on multiple machines. I struggled quite a bit with it and until I hit upon the correct option, I was trying out too many paths and backtracking everytime I hit a wall or an unsurmountable obstacle. I am writing down the issues I faced and other details for posterity, in case (who am I kidding, I definitely will) have to) do this again.

At a very high level, I had 3 options/paths I could go down (from what I understood):
1. Deploy the containers on multiple physical machines. Setup the DNS on all machines such that they can discover each other. 
	The DNS settings could either be written to the /etc/hosts on each machine or we could setup a single local DNS server and add it to the /etc/hosts which seems somewhat neater.

	On the plus side, it sounded like the exact definition of multi-machine deployment of a blockchain network, where independant machines were communicating and running Fabric.

	On the minus side, setting up DNS will be nasty (read require root permissions, need hardcoding, be hard to scale etc.), deployment will have to be manual and fault tolerance (if any, will have to be handled by us, and every computer scientist avoids reimplementing something already polished to perfection at this point of time.)

	Though, in all honesty, I did start trying this method. I added grpcs://orderer.example.com to the /etc/resolv.conf file, but there seemed to be some issue with the ports which I avoided debugging to try out other methods. Option 2 worked (thankfully!) and this section ended up being a stub.

2. Use docker stack deploy on a docker swarm

	This is the method I ended up employing and have described in detail below.

3. Use docker stack deploy on a Kubernetes cluster

	Kubernetes! Run!

## Docker stack deploy on docker swarm

### Docker swarm
Docker swarm allows one to add multiple machines (managers or workers) to a swarm (duh!). Using docker stack deploy on a manager in a swarm load balances and starts the containers on all the other nodes in the swarm.

The manager machines run Raft for consensus for maintaining the swarm, it needs a quorum of the managers to be alive. (Read before on re-implementing, this is exactly what we would have to have re-implemented).

Yet to understand how fault tolerance is provided, what the managers specifically do etc., basically everything under the hood.

` docker swarm init ` initializes the swarm. `docker swarm join-token <manager|worker` gives the token for any machine to join the swarm.

`docker node ls` on any manager node lists the nodes in the swarm.

### Docker stack deploy
There are a bunch of issues in using `docker stack deploy` directly with a Fabric deployment script.

I am assuming the Fabric deployment script consists majorly of a docker-compose.yaml which is responsible for the ca, orderer and peer containers.

Listing them now:
1. `docker stack deploy` needs version `>3` of compose.
2. `docker stack deploy` cannot have the container/volume names contain special characters. This is especially a problem for Fabric, since all the containers are named xyz.orgX.example.com etc.
	There were 2 ways to solve this problem:
	1. The Long Route: Change the container names to not "contain" a ".", by replacing all occurences of "." with an "_" or "-". Unfortunately, this is not as simple as it sounds. It will involve changing all the crypto material generated since the URL on the certificate will not match the request URL. For this, we need to use the specs, CommonName options in the `crypto-config.yaml` used to generate the certificates for the CA, Peers and Orderers or disable TLS altogether (not sure if it will work, but we cannot disable TLS if we need to use Raft consensus for the orderers).
	2. The Short Route: Name the container/volume without special characters. Use the network.alias field in docker-compose to set the network alias of the container to the required xyz.OrgX.example.com
3. Volume mounts cannot be relative paths, otherwise containers will not be deployed on multiple machines (due to dependencies). Thus, all the crypto material (TLS certs and private keys) and network config (Genesis block) needs to be `scp`'d to all the swarm machine and should be located in the same location in each machine so that a single absolute path to the config can be specified.

## Docker network
After much experimentation, it seemed best to use an external attachable network. This is because the dev-peer containers created by each peer need to be attachable to the same network which is not allowed, if the network is created in the docker-compose file itself (unless you provide the attachable config there which is basically just a choice at this point).

`docker network create --driver overlay --attachable <nw_name>`

Then use the network as external network in all the compose files needed. I say **all** since there might be additional containers such as **cli**, **backend**, **frontend** etc. as needed for your setup.

```
networks:
	default:
		external:
			name: <nw_name>
```

Within the container/volume definition:

```
networks:
	<nw_name>:
		aliases:
			- xyz.orgX.example.com
```

Sometimes, the `docker network` throws the world's most frustrating error messages, such as cannot create a network since it already exists while `rm` throws no such network error. One fix for one of the errors is :

```
docker inspect <nw_name>
```

There will be an endpoint container still running - <nw-endpoint>

```
docker network disconnect -f nw_name <nw-endpoint>
```

Do not miss the `-f`, it is what saves your life.

## Deployment
```
docker stack deploy -c docker-compose.yaml fabric
```

Finally, you can deploy the one time cli, backend etc. on a single machine using 

```
docker-compose -f setup.yaml up -d
```

Just remember to set the external network as mentioned before and everything works like magic.

### Fault Tolerance test
- Yet to perform fault tolerance tests on this setup.yaml
- However, for a 2 (1 manager + 1 worker) setup, when the worker is disconnected from the manager, all the containers in the worker exit and are killed. It is probably something got to do with the low number of managers I think (1 Ha...Ha...Ha... , that system is waiting to fail).

### References
Indebted to the following links which have provided deployment scripts etc. for deploying Fabric on docker swarm. I had to use a mix of the 3 repos.
1. [Balance Transfer Docker Swarm](https://github.com/sebastianpaulp/Balance_Transfer_Docker_Swarm)
2. [fabric-network-on-swarm](https://github.com/dineshrivankar/fabric-network-on-swarm)
3. [hlf-docker-swarm](https://github.com/skcript/hlf-docker-swarm)
