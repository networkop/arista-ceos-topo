# docker-topo
Docker network topology builder

[![Build Status](https://travis-ci.org/networkop/arista-ceos-topo.svg)](https://travis-ci.org/networkop/arista-ceos-topo)

# TODO

* Write more tests
* Multi-host topologies with VXLAN links - not sure if veth + linux bridges are needed or if its possible to get away with macvlan (bridge mode) over a vxlan interface???
* Add a configurable subnet option for docker bridge networks

# Installation

With Python virtualenv (recommended)

```bash
python3 -m pip install virtualenv
python3 -m virtualenv testdir; cd testdir
source bin/activate 
pip install git+https://github.com/networkop/arista-ceos-topo.git
```

Without virtualenv

````bash
python3 -m pip install git+https://github.com/networkop/arista-ceos-topo.git
````

> Note: Python 2.x is not supported

# Usage

```bash
# docker-topo -h
usage: docker-topo [-h] [-d] [--create | --destroy] topology

Tool to create cEOS topologies

positional arguments:
  topology     Topology file

optional arguments:
  -h, --help   show this help message and exit
  -d, --debug  Enable Debug
  --create     Create topology
  --destroy    Destroy topology
```

# Topology file

Topology file is a YAML file describing how docker containers are to be interconnected.
This information is stored in the `links` variable which 
contains a list of links. Each link is described by a unique set of connected interfaces. 
There are several versions of topology file formats. 

## Topology file v1
This version is considered legacy and is documented [here](https://github.com/networkop/arista-ceos-topo/blob/master/v1.md). 

## Topology file v2
Each link in a `links` array is a dictionary with the following format:
```yaml
VERSION: 2
links:
  - endpoints:
      - "Device-A:Interface-2" 
      - "Device-B:Interface-1"
  - driver: macvlan
    driver_opts: 
      parent: wlp58s0
    endpoints: ["Device-A:Interface-1", "Device-B:Interface-2"]
```
Each link dictionary supports the following elements:

* **endpoints** - the only mandatory element, contains a list of endpoints to be connected to a link. 
* **driver** - defines the link driver to be used. Currently supported drivers are **veth, bridge, macvlan**. When driver is not specified, default **bridge** driver is used. The following limitations apply:
  * **macvlan** driver will require a mandatory **driver_opts** object described below
  * **veth** driver is talking directly to netlink (no libnetwork involved) and making changes to namespaces
* **driver_opts** - optional object containing driver options as required by Docker's libnetwork. Currently only used for macvlan's parent interface definition

Each link endpoint is encoded as "DeviceName:InterfaceName:IPPrefix" with the following contraints:

* **DeviceName** determines which docker image is going to be used by (case-insensitive) matching of the following strings:
  * **host** - [alpine-host][alpine-host] image is going to be used
  * **cvp** - [cvp](https://github.com/networkop/arista-ceos-topo/tree/master/topo-extra-files/cvp) image is going to be used
  * For anything else Arista cEOS image will be used 
* **InterfaceName** must match the exact name of the interface you expect to see inside a container. For example if you expect to connect a link to DeviceA interface eth0, endpoint definition should be "DeviceA:eth0"
* **IPPrefix** - Optional parameter that works **ONLY** for [alpine-host][alpine-host] devices and will attempt to configure a provided IP prefix inside a container.

## Bridge vs veth driver caveats

Both bridge and veth driver have their own set of caveats. Keep them in mind when choosing a driver:

| Features | bridge | veth |
|--------|------|------|
| multipoint links | supported | not supported |
| sudo privileges | not required | required | 
| docker modifications | requires [patched][1] docker deamon | uses standard docker daemon |
| L2 multicast | only LLDP | supported | 
 
You can mix both **bridge** and **veth** drivers in the same topology, however make sure that **bridge** driver links always come first, followed by the **veth** links. For example:

```
VERSION: 2
driver: veth

links:
  - endpoints: ["Leaf1:eth1", "Leaf2:eth1"]
    driver: 'bridge'
  - endpoints: ["Leaf1:eth2", "Leaf2:eth2"]
  - endpoints: ["Leaf1:eth3", "Leaf2:eth3"]
```


## (Optional) Global variables
Along with the mandatory `link` array, there are a number of options that can be specified to override some of the default settings. Below are the list of options with their default values:

```yaml
VERSION: 1  # Topology file version. Accepts [1|2]
CEOS_IMAGE: ceos:latest # cEOS docker image name
CONF_DIR: './config' # Config directory to store cEOS startup configuration files
PUBLISH_BASE: 8000 # Publish cEOS ports starting from this number
OOB_PREFIX: '192.168.100.0/24' # Only used when link contains CVP or for Macvlan addressing. This prefix is assinged to the containers first ethernet interface.
OOB_IPPREFIX: 192.168.100.128/25' # Used to steer which addresses within the prefix are being assigned to the containers using Macvlan addressing.
OOB_GATEWAY: '192.168.100.1' # The default gateway when using Macvlan addressing.
PREFIX: 'CEOS-LAB' # This will default to a topology filename (without .yml extension)
driver: None
```

All of the capitalised global variables can also be provided as environment variables with the following priority:

1. Global variables defined in a topology file
2. Global variables from environment variables
3. Defaults

The final **driver** variable can be used to specify the default link driver for **ALL** links at once. This can be used to create all links with non-default **veth** type drivers:

```yaml
VERSION: 2
driver: veth
links:
  - endpoints: ["host1:eth1", "host2:eth1"]
  - endpoints: ["host1:eth2", "host3:eth1"]
```


There should be several examples in the `./topo-extra-files/examples` directory


# Example 1 - Creating a 2-node topology interconnected directly with veth links (without config)

```text
+------+             +------+
|      |et1+-----+et2|      |
|cEOS 1|             |cEOS 2|
|      |et2+-----+et1|      |
+------+             +------+
```

```bash
sudo docker-topo --create topo-extra-files/examples/v2/2-node.yml
```

# Example 2 - Creating a 3-node topology using the default docker bridge driver (with config)
```text
+------+             +------+
|cEOS 1|et1+-----+et2|cEOS 2|
+------+             +------+
   et2                  et1
    +                    +
    |      +------+      |
    +--+et1|cEOS 3|et2+--+
           +------+

```

```bash
mkdir config
echo "hostname cEOS-1" > ./config/3-node_cEOS-1
echo "hostname cEOS-2" > ./config/3-node_cEOS-2
echo "hostname cEOS-3" > ./config/3-node_cEOS-3
docker-topo --create topo-extra-files/examples/v1-legacy/3-node.yml
```

# List and connect to devices

```bash
# docker ps -a 
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                     PORTS                   NAMES
2315373f8741        ceosimage:latest    "/sbin/init"             About a minute ago   Up About a minute          0.0.0.0:9002->443/tcp   3-node_cEOS-3
e427def01f3a        ceosimage:latest    "/sbin/init"             About a minute ago   Up About a minute          0.0.0.0:9001->443/tcp   3-node_cEOS-2
f1a2ac8a904f        ceosimage:latest    "/sbin/init"             About a minute ago   Up About a minute          0.0.0.0:9000->443/tcp   3-node_cEOS-1


# docker exec -it 3-node_cEOS-1 Cli
cEOS-1>
```

# Destroy a topology

```bash
docker-topo --destroy topo-extra-files/examples/3-node.yml
```

# Troubleshooting

* If you get the following error, try renaming the topology filename to a string [shorter than 15 characters](https://github.com/svinota/pyroute2/issues/452)

  ```
  pyroute2.netlink.exceptions.NetlinkError: (34, 'Numerical result out of range')
  ```

* CVP can't connect to cEOS devices - make sure that CVP is attached with at least two interfaces. The first one is always for external access and the second one if always for device management

[1]: https://networkop.co.uk/post/2018-03-03-docker-multinet/
[alpine-host]: https://github.com/networkop/arista-ceos-topo/tree/master/topo-extra-files/host
