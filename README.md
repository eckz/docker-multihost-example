# Docker multihost example

This is an example using the docker multi host neworking,
similar to the official experimental guide: https://github.com/docker/docker/blob/master/experimental/compose_swarm_networking.md

You can also check http://blog.docker.com/2015/06/networking-receives-an-upgrade/

This vagrant configuration has been inspired from the old socketplane project (https://github.com/socketplane/socketplane)

By defaul this spins up 3 nodes with the following configuration
* Hostname node-#{n}
* Ip: 10.100.0.#{n + 10}

For example:
* node-1: 10.100.0.10
* node-2: 10.100.0.11
* node-3: 10.100.0.12
* ...

You can change the number of nodes with the `MULTIHOST_NODES` environment var.

For example (Don't try it please)
```sh
$ MULTIHOST_NODES=2048 vagrant up
```

And then it setup in every node a key elements

  - Consul.io cluster in every node
  - Docker engine with overlay network configuration
  - Swarm join using consul discovery
  - Swarm managers in replication mode, using spread strategy


## ----------- Warning this configuration is insecure---------
* The consul configuration is insecure (client on 0.0.0.0)
* The docker engine configuration is insecure (on 0.0.0.0, no TLS)
* Swarm manager configuration is insecure (on 0.0.0.0, using net host, no TLS)

This is just an example of the technology, so it's not focus on security, and it should not seen as a production setup reference by any means.

## Install
```sh
$ vagrant plugin install vagrant-reload
$ vagrant up
```

### Urls

* Consul ui access:
[http://10.100.0.10:8500/ui/](http://10.100.0.10:8500/ui/), [http://10.100.0.11:8500/ui/](http://10.100.0.11:8500/ui/), ....
* Swarm managers
[tcp://10.100.0.10:3375](tcp://10.100.0.10:3375), [tcp://10.100.0.11:3375](tcp://10.100.0.11:3375), ...

### Usage
```sh
$ vagrant ssh node-1 # It can be node-2, or node-3
$ export DOCKER_HOST='tcp://10.100.0.10:3375' # It can be 10.100.0.11 o 10.100.0.12 with no problem
$ docker info
Containers: 6
Images: 3
Role: replica
Primary: 10.100.0.11:3375
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 3
 node-1: 10.100.0.10:2375
  Containers: 2
  Reserved CPUs: 0 / 1
  Reserved Memory: 0 B / 513.6 MiB
  Labels: com.docker.network.driver.overlay.bind_interface=eth1, executiondriver=native-0.2, kernelversion=3.19.0-21-generic, operatingsystem=Ubuntu 15.04, storagedriver=aufs
 node-2: 10.100.0.11:2375
  Containers: 2
  Reserved CPUs: 0 / 1
  Reserved Memory: 0 B / 513.6 MiB
  Labels: com.docker.network.driver.overlay.bind_interface=eth1, com.docker.network.driver.overlay.neighbor_ip=10.100.0.10, executiondriver=native-0.2, kernelversion=3.19.0-21-generic, operatingsystem=Ubuntu 15.04, storagedriver=aufs
 node-3: 10.100.0.12:2375
  Containers: 2
  Reserved CPUs: 0 / 1
  Reserved Memory: 0 B / 513.6 MiB
  Labels: com.docker.network.driver.overlay.bind_interface=eth1, com.docker.network.driver.overlay.neighbor_ip=10.100.0.10, executiondriver=native-0.2, kernelversion=3.19.0-21-generic, operatingsystem=Ubuntu 15.04, storagedriver=aufs
CPUs: 3
Total Memory: 1.505 GiB
```

As you can see we have the three nodes running
### Docker compose example

This is the same example as the original docker guide, but pointing to a direct image build of the counter example (eckz/counter)

If you don't trust this image you can build it yourself from the code bellow composertest/counter

```sh
$ vagrant ssh node-1 # It can be node-2, or node-3
$ export DOCKER_HOST='tcp://10.100.0.10:3375' # It can be 10.100.0.11 o 10.100.0.12 with no problem
$ cd /vagrant/composetest/
# this takes a litle bit since it needs to pull both imagse from both nodes
$ docker-compose up -d
$ docker ps
CONTAINER ID        IMAGE           NAMES
521078ea0ae6        redis           node-1/composetest_redis_1
e055db4590a3        eckz/counter    node-2/composetest_web_1
$ curl http://10.100.0.11/
Hello World! I have been seen 1 times.
$ curl http://10.100.0.11/
Hello World! I have been seen 2 times.
$ docker-compose scale web=2 # This should assign the second web to node-3
$ docker ps
CONTAINER ID        IMAGE               NAMES
5a2164522b0a        eckz/counter        node-3/composetest_web_2
521078ea0ae6        redis               node-1/composetest_redis_1
e055db4590a3        eckz/counter        node-2/composetest_web_1
$ curl http://10.100.0.12/
Hello World! I have been seen 3 times.
$ curl http://10.100.0.11/
Hello World! I have been seen 4 times.
```

That's pretty much everything, as you can see, you are able to create two different web containers in different machines connected to the same redis server, which is in a different machine.

### Version
0.0.-1


### Todo's

Create a balancer that works in multi host mode

License
----

MIT
