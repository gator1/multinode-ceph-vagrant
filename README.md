# Multinode Ceph on Vagrant

This workshop walks users through setting up an 8-node [Ceph](http://ceph.com) cluster and mounting a block device, using a CephFS mount, and storing a blob object.
Jepsen test is conducted for ceph on the three osd nodes, osd-1 .. 3

It follows the following Ceph user guides:

* [Preflight checklist](http://ceph.com/docs/master/start/quick-start-preflight/)
* [Storage cluster quick start](http://ceph.com/docs/master/start/quick-ceph-deploy/)
* [Block device quick start](http://ceph.com/docs/master/start/quick-rbd/)
* [Ceph FS quick start](http://ceph.com/docs/master/start/quick-cephfs/)
* [Install Ceph object gateway](http://ceph.com/docs/master/install/install-ceph-gateway/)
* [Configuring Ceph object gateway](http://ceph.com/docs/master/radosgw/config/)

Note that after many commands, you may see something like:

```
Unhandled exception in thread started by
sys.excepthook is missing
lost sys.stderr
```

I'm not sure what this means, but everything seems to have completed successfully, and the cluster will work.

## Install prerequisites

Install [Vagrant](http://www.vagrantup.com/downloads.html) and a provider such as [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

We'll also need the [vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) and [vagrant-hostmanager](https://github.com/smdahlen/vagrant-hostmanager) plugins:

the VM has trouble with apt-get, user _apt etc. Change the Vagrantfile to change. From  
https://github.com/fgrehm/vagrant-cachier/issues/175  

```console
$ vagrant plugin install vagrant-cachier
$ vagrant plugin install vagrant-hostmanager
```

## Add your Vagrant key to the SSH agent

Since the admin machine will need the Vagrant SSH key to log into the server machines, we need to add it to our local SSH agent:

On Mac:
```console
$ ssh-add -K ~/.vagrant.d/insecure_private_key
```

On \*nix:
```console
$ ssh-add -k ~/.vagrant.d/insecure_private_key
```

no agent for jepsen, gator   

## Start the VMs

This instructs Vagrant to start the VMs and install `ceph-deploy` on the admin machine.

```console
$ vagrant up
```

## Create the cluster

We'll create a simple cluster and make sure it's healthy. Then, we'll expand it.

First, we need to get an interactive shell on the admin machine:

```console
$ vagrant ssh ceph-admin
```

The `ceph-deploy` tool will write configuration files and logs to the current directory. So, let's create a directory for the new cluster:

```console
vagrant@ceph-admin:~$ mkdir test-cluster && cd test-cluster
```

Let's prepare the machines:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy new mon-1 mon-2 mon-3
```

Now, we have to change a default setting. For our initial cluster, we are only going to have three [object storage daemons](http://ceph.com/docs/master/man/8/ceph-osd/). We need to tell Ceph to allow us to achieve an `active + clean` state with just three Ceph OSDs. Add `osd pool default size = 3` to `./ceph.conf`.

Because we're dealing with multiple VMs sharing the same host, we can expect to see more clock skew. We can tell Ceph that we'd like to tolerate slightly more clock skew by adding the following section to `ceph.conf`:
```
mon_clock_drift_allowed = 1
```

**Important**: As of the Jewel release of Ceph, the Ceph team recommends using XFS instead of ext4. Unfortunately, I couldn't find an easy and standard way in the Vagrantfile to attach an extra volume to use for Ceph. For more information, see the [Ceph docs](http://docs.ceph.com/docs/jewel/rados/configuration/filesystem-recommendations/#not-recommended) and [multinode-ceph-vagrant/#15](https://github.com/carmstrong/multinode-ceph-vagrant/issues/15).

In the meantime, there is a workaround if we add the following to `ceph.conf`:

```
osd max object name len = 256
osd max object namespace len = 64
```

After these few changes, the file should look similar to:

```
[global]
fsid = 7acac25d-2bd8-4911-807e-e35377e741bf
mon_initial_members = mon-1, mon-2, mon-3
mon_host = 172.21.12.12,172.21.12.13,172.21.12.14
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd pool default size = 3
mon_clock_drift_allowed = 1
osd max object name len = 256
osd max object namespace len = 64
```

## Install Ceph

We're finally ready to install!

Note here that we specify the Ceph release we'd like to install, which is [kraken](http://docs.ceph.com/docs/master/release-notes/#v11-2-0-kraken).

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy install --release=kraken ceph-admin mon-1 mon-2 mon-3 osd-1 osd-2 osd-3 ceph-client
```

## Configure monitor and OSD services

Next, we add a monitor node:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mon create-initial
```

And our two OSDs. For these, we need to log into the server machines directly:

```console
vagrant@ceph-admin:~/test-cluster$ ssh osd-1 "sudo mkdir /var/local/osd0 && sudo chown ceph:ceph /var/local/osd0"
```

```console
vagrant@ceph-admin:~/test-cluster$ ssh osd-2 "sudo mkdir /var/local/osd1 && sudo chown ceph:ceph /var/local/osd1"
vagrant@ceph-admin:~/test-cluster$ ssh osd-3 "sudo mkdir /var/local/osd2 && sudo chown ceph:ceph /var/local/osd2"
```

Now we can prepare and activate the OSDs:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd prepare osd-1:/var/local/osd0 osd-2:/var/local/osd1 osd-3:/var/local/osd2  
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd activate osd-1:/var/local/osd0 osd-2:/var/local/osd1 osd-3:/var/local/osd2  
```

## Configuration and status

We can copy our config file and admin key to all the nodes, so each one can use the `ceph` CLI.

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy admin ceph-admin mon-1 mon-2 mon-3 osd-1 osd-2 osd-3 ceph-client
```

We also should make sure the keyring is readable:

```console
vagrant@ceph-admin:~/test-cluster$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh mon-1 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh mon-2 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh mon-3 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

Finally, check on the health of the cluster:

```console
vagrant@ceph-admin:~/test-cluster$ ceph health
```

You should see something similar to this once it's healthy:

```console
vagrant@ceph-admin:~/test-cluster$ ceph health
HEALTH_OK
vagrant@ceph-admin:~/test-cluster$ ceph -s
    cluster 18197927-3d77-4064-b9be-bba972b00750
     health HEALTH_OK
     monmap e2: 3 mons at {mon-1=172.21.12.12:6789/0,mon-2=172.21.12.13:6789/0,mon-3=172.21.12.14:6789/0}, election epoch 6, quorum 0,1,2 mon-1,mon-2,mon-3
     osdmap e9: 2 osds: 2 up, 2 in
      pgmap v13: 192 pgs, 3 pools, 0 bytes data, 0 objects
            12485 MB used, 64692 MB / 80568 MB avail
                 192 active+clean
```

Notice that we have three OSDs (`osdmap e9: 3 osds: 3 up, 3 in`) and all of the [placement groups](http://ceph.com/docs/master/rados/operations/pg-states/) (pgs) are reporting as `active+clean`.

Congratulations!

## Expanding the cluster

To more closely model a production cluster, we're going to add one more OSD daemon and a [Ceph Metadata Server](http://ceph.com/docs/master/man/8/ceph-mds/). We'll also add monitors to all hosts instead of just one.

### Add an OSD
```console
vagrant@ceph-admin:~/test-cluster$ ssh osd-3 "sudo mkdir /var/local/osd2 && sudo chown ceph:ceph /var/local/osd2"
```

Now, from the admin node, we prepare and activate the OSD:
```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd prepare osd-3:/var/local/osd2
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd activate osd-3:/var/local/osd2
```

Watch the rebalancing:

```console
vagrant@ceph-admin:~/test-cluster$ ceph -w
```

You should eventually see it return to an `active+clean` state, but this time with 4 OSDs:

```console
vagrant@ceph-admin:~/test-cluster$ ceph -w
    cluster 18197927-3d77-4064-b9be-bba972b00750
     health HEALTH_OK
     monmap e2: 3 mons at {mon-1=172.21.12.12:6789/0,ceph-server-2=172.21.12.13:6789/0,ceph-server-3=172.21.12.14:6789/0}, election epoch 30, quorum 0,1,2 mon-1,mon-2,mon-3
     osdmap e38: 4 osds: 4 up, 4 in
      pgmap v415: 192 pgs, 3 pools, 0 bytes data, 0 objects
            18752 MB used, 97014 MB / 118 GB avail
                 192 active+clean
```

### Add metadata server

Let's add a metadata server to server1:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mds create mon-1
```

## Add more monitors

We add monitors to servers 2 and 3.

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mon create ceph-server-2 ceph-server-3
```

Watch the quorum status, and ensure it's happy:

```console
vagrant@ceph-admin:~/test-cluster$ ceph quorum_status --format json-pretty
```

## Install Ceph Object Gateway

TODO

## Play around!

Now that we have everything set up, let's actually use the cluster. We'll use the ceph-client machine for this.

### Create a block device

```console
$ vagrant ssh ceph-client
vagrant@ceph-client:~$ sudo rbd create foo --size 4096 -m mon-1 --image-feature layering   
vagrant@ceph-client:~$ sudo rbd map foo --pool rbd --name client.admin -m mon-1
vagrant@ceph-client:~$ sudo mkfs.ext4 -m0 /dev/rbd/rbd/foo
vagrant@ceph-client:~$ sudo mkdir /mnt/ceph-block-foo
vagrant@ceph-client:~$ sudo mount /dev/rbd/rbd/foo /mnt/ceph-block-foo
```

### Create a mount with Ceph FS

TODO

### Store a blob object

TODO

## jespen  
After rbd is done, do git clone gator1/jepsen.git and has a jepsen directory under /home/vagrant.  

Jepsen needs root access to the osd nodes. Jepsen control node is ceph-client node.   
This could be done automatically but I did manaully.   

first change the root password (I don't know VM's root password, VMs are ubuntu).  
for ceph-client, osd-1 .. 3 do:  
'sudo passwd root' and use 'root' as password.   
also need to enable root ssh access:  
modify /etc/ssh/sshd_config for PermitRootLogin=yes  
sudo systemctl restart sshd.service  


from ceph-client: su root to login as root.   
'ssh-keygen -t rsa -N ""' to generate for root. The files id_rsa and id_ras.pub are generated in /root/.ssh  
ssh-copy-id root@osd-1  
ssh-copy-id root@osd-2  
ssh-copy-id root@osd-3  

from /root/.ssh  
scp id_rsa root@osd-1:/root/.ssh  
scp id_rsa root@osd-2:/root/.ssh  
scp id_rsa root@osd-3:/root/.ssh  
rm known_hosts   
ssh-keyscan -t rsa osd-1 >> ~/.ssh/known_hosts  
ssh-keyscan -t rsa osd-2 >> ~/.ssh/known_hosts  
ssh-keyscan -t rsa osd-3 >> ~/.ssh/known_hosts  

jepsen seems to hard code for iptables for eth0 but VM uses diff interface names.  
## Cleanup

When you're all done, tell Vagrant to destroy the VMs.

```console
$ vagrant destroy -f
```
