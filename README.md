## Smartphone Test Farm Setup Examples

This repository contains setup examples of [openstf/stf](https://github.com/openstf/stf) on [various hosts](#supported-hosts) using [Vagrant](https://www.vagrantup.com/) with [VirtualBox](https://www.virtualbox.org/) provider.

## Supported Hosts
- [CoreOS](#run-stf-on-coreos-cluster)

## Requirements
- [VirtualBox](https://www.virtualbox.org/) >= 5.0.0
- [Vagrant](https://www.vagrantup.com/) >= 1.7.3

## Setup

```sh
git clone https://github.com/openstf/setup-examples.git stf-setup-examples
cd stf-setup-examples
```

## Run Rethinkdb Cluster

```sh
cd ./db
vagrant up
```
Above command will do
- Download **ubuntu/trusty64** image if image is not present (*this may take time depending on internet speed*)
- Launch Ubuntu VM and set its IP to `$rethinkdb_host_ip` configured in [Configuration File](config.rb) (**Default: 198.162.50.11**)
- Install and run rethinkdb server

You can confirm if rethinkdb is up by visiting rethinkdb admin console (http://198.162.50.11:8080 or http://RETHINKDB_HOST_IP:8080)

## Run STF on CoreOS Cluster

### Requirements
- Install [fleetctl client](https://coreos.com/fleet/docs/latest/launching-containers-fleet.html)
  - On OS X, you can install it using `brew install fleetctl`.
- Make sure that rethinkdb is running by following [these instructions](#run-rethinkdb-cluster)

### Configuration
You can configure global variable present in [CoreOS Configuration File](coreos/coreos_config.rb) according to your Requirements.

### Run CoreOS Cluster

```sh
cd coreos
vagrant up
```
Above command will do
- Download **CoreOS** image if image is not present (*this may take time depending on internet speed*)
- Launch `$num_instances` configured in [CoreOS Configuration File](coreos/coreos_config.rb) instances of CoreOS VM
- You can check if all the instances are running using `vagrant global-status` command

### Configuring fleetctl
Next, we will be launching services inside CoreOS cluster using fleetctl. First we need to export a global variable
FLEETCTL_ENDPOINT. This endpoint will tell fleetctl running on host os (OS X in my case) to talk with fleet daemon running inside one of the guest OS.

```sh
export FLEETCTL_ENDPOINT=http://172.17.8.101:2379
```
Using core-01 guest OS.

Now, run `fleetctl list-machines` command, You will see something like below

```sh
MACHINE     IP           METADATA
6e3e7dc2... 172.17.8.101 -
9821f508... 172.17.8.102 -
ab5747a8... 172.17.8.103 -
```

Now, your CoreOS cluster is ready to deploy STF Components.

### Deploy STF Components
#### adbd.service
You need to have a single `adbd.service` unit running on each host where devices are connected.`Global=true` X-Fleet parameter in [adbd.service](coreos/unit_files/adbd.service) will make it run on each cluster host.

Deploy using below command

```sh
fleetctl submit ./unit_files/adbd.service
fleetctl start adbd
```
It will take `adbd.service` some time to start on each host. Once it is done, you can check by running `fleetctl list-units` command. You will see

```sh
UNIT		MACHINE				ACTIVE	SUB
adbd.service	6e3e7dc2.../172.17.8.101	active	running
adbd.service	9821f508.../172.17.8.102	active	running
adbd.service	ab5747a8.../172.17.8.103	active	running
```

#### rethinkdb-proxy-28015.service
You need a single instance of the rethinkdb-proxy-28015.service unit on each host where you have another unit that needs to access the database. In this example we will run this unit on all host by setting `Global=true` parameter in [rethinkdb-proxy-28015.service](coreos/unit_files/rethinkdb-proxy-28015.service) unit file.

Start service using

```sh
fleetctl submit ./unit_files/rethinkdb-proxy-28015.service
fleetctl start rethinkdb-proxy-28015
```

#### stf-migrate.service
This unit creates database and tables necessary for STF. This is a oneshot unit, meaning that it shuts down after it's done.

Start service using

```sh
fleetctl submit ./unit_files/stf-migrate.service
fleetctl start stf-migrate
```
Again, this unit will take time to start since, it will be downloading stf docker image for the first time.

Once the service is done, go to the rethinkdb admin console (http://198.162.50.11:8080/) and check if you can see [devices, logs, users, vncauth] tables in "Tables" tab.
 
