# arista-ceos-topo
Arista cEOS topology builder

# Installation

````bash
pip install git+https://github.com/networkop/arista-ceos-topo.git
````

# Usage

```bash
# ceos-topo -h
usage: ceos-topo [-h] [-d] [--create | --destroy] topology

Tool to create cEOS topologies

positional arguments:
  topology     Topology file

optional arguments:
  -h, --help   show this help message and exit
  -d, --debug  Enable Debug
  --create     Create topology
  --destroy    Destroy topology
```

Topology file is a YAML file describing how cEOS are to be interconnected.
This information is stored in the `links` dictionary which 
contains a list of links, each described by a unique set of connected interfaces.
For example, the following will result in two cEOS containers
connected together with their first interfaces:
```yaml
links:
 - ["Device-1:Interface1", "Device-2:Interface1"]
```

Links can also be connected to host interfaces using the Macvlan kernel driver.
This works by appending a section to the links between the devices.
```yaml
links:
 - ["Device-1:Interface1", "Device-2:Interface1", "Macvlan:Eth1"]
```

There should be three topology examples in the `./arista-ceos-files/examples` directory

# (Optional) Override default variables
There are several global variables used by the script:

* PREFIX - unique label assigned to all docker containers and 
links within a single topology. Used as a filter during the 
cleanup process.
* CONF_DIR - path to the config directory if/when configuration 
files are supplied
* CEOS_IMAGE - Name of the cEOS image to use
* PUBLISH_BASE - A port number offset by a device index to which to publish
the internal https port.

These variables can be defined in either a topology file (first choice) or 
as environment variables of the shell calling the script (second choice).
If any of the variables is not explicitly defined, the following defaults 
will be used:

* CONF_DIR = './config'
* PREFIX = 'CEOS-LAB'
* CEOS_IMAGE = 'ceos:latest'
* PUBLISH_BASE = 8000

For the case of 3 containers, the above PUBLISH_BASE setting will result in 
the following mapping:
* cEOS-1 port 443 - Docker host port 8000
* cEOS-2 port 443 - Docker host port 8001
* cEOS-3 port 443 - Docker host port 8002

# Example 1 - Creating a 2-node topology (without config)

```text
+------+             +------+
|      |et1+-----+et2|      |
|cEOS 1|             |cEOS 2|
|      |et2+-----+et1|      |
+------+             +------+
```

```bash
ceos-topo --create arista-ceos-files/examples/2-node.yml
```

# Example 2 - Creating a 3-node topology (with config)
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
echo "hostname cEOS-1" > ./config/cEOS-1
echo "hostname cEOS-2" > ./config/cEOS-2
echo "hostname cEOS-3" > ./config/cEOS-3
ceos-topo --create arista-ceos-files/examples/3-node.yml
```

# List and connect to devices

```bash
# docker ps -a 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
3d61fb999756        veos:latest         "/sbin/init"        2 minutes ago       Up 2 minutes                            Device-B
4b5743e64e5e        veos:latest         "/sbin/init"        2 minutes ago       Up 2 minutes                            Device-A

# docker exec -it Device-A Cli
localhost>
```

# Destroy a topology

```bash
ceos-topo --destroy arista-ceos-files/examples/2-node.yml
```
