# What

This will provision a Rancher/PX cluster on a local VirtualBox. An etcd container will also be started on the master node for Portworx KVDB.

# How

1. Install [Vagrant](https://www.vagrantup.com/downloads.html) and [VirtualBox](https://www.virtualbox.org/wiki/Downloads).
2. Clone this repo and cd to it.
3. Edit top section of `Vagrantfile` as necessary.
4. Generate SSH keys:
```
$ ssh-keygen -t rsa -b 2048 -f id_rsa
```
This will allow SSH as root between the various nodes.

5. Start the cluster:
```
vagrant up
```

6. Check the status of the Portworx cluster:
```
$ vagrant ssh node1
[vagrant@node1 ~]$ sudo /opt/pwx/bin/pxctl status
Status: PX is operational
License: Trial (expires in 31 days)
Node ID: 74baa3e6-41c4-4a76-a5c8-36ec5e8f0a41
	IP: 192.168.99.101
 	Local Storage Pool: 1 pool
	POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE	REGION
	0	LOW		raid0		20 GiB	3.0 GiB	Online	default	default
	Local Storage Devices: 1 device
	Device	Path		Media Type		Size		Last-Scan
	0:1	/dev/sdb	STORAGE_MEDIUM_MAGNETIC	20 GiB		21 May 19 08:23 UTC
	total			-			20 GiB
	Cache Devices:
	No cache devices
Cluster Summary
	Cluster ID: px-test-cluster
	Cluster UUID: 81b20e78-6389-458f-a712-14eedd6066fa
	Scheduler: rancher
	Nodes: 3 node(s) with storage (3 online)
	IP		ID					SchedulerNodeName	StorageNode	Used	Capacity	Status	StorageStatus	Version		Kernel				OS
	192.168.99.102	c3e8f539-8c6d-4929-adb5-3d66b3ac06d0	N/A			Yes		3.0 GiB	20 GiB		Online	Up		2.1.0.0-6495007	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
	192.168.99.103	c1dd972d-30ab-49ca-9335-1a4eeb6384d1	N/A			Yes		3.0 GiB	20 GiB		Online	Up		2.1.0.0-6495007	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
	192.168.99.101	74baa3e6-41c4-4a76-a5c8-36ec5e8f0a41	N/A			Yes		3.0 GiB	20 GiB		Online	Up (This node)	2.1.0.0-6495007	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
	Warnings:
		 WARNING: Persistent journald logging is not enabled on this node.
Global Storage Pool
	Total Used    	:  9.1 GiB
	Total Capacity	:  60 GiB
```

# Notes

The object is to provision everything on top of a base CentOS installation to make it easier to test different software versions. This is inherently slower than baking everything into a prebuilt image.

After each VM is provisioned, the bootstrap process runs in the background (so Vagrant can continue provisioning the subsequent nodes in parallel).

The process logs to `/var/log/vagrant.boostrap` on each node. When the process is completed, the line "End" is logged.
