# Install ceph manually

Add ceph repository to apt with commands:
`wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -` 

`echo deb https://download.ceph.com/debian-quincy/ **$(**lsb_release -sc**)** main | sudo tee /etc/apt/sources.list.d/ceph.list`

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled.png)

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%201.png)

Run `apt update` and `apt install ceph ceph-common` 

**Add backport for some additional** 

`echo deb [http://archive.debian.org/debian](http://deb.debian.org/debian) buster-backports main >> /etc/apt/sources.list`

Then `apt update`

If encounter this error, 

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%202.png)

then copy 2 key for this command, then update again

```bash
apt-key adv --keyserver keyserver.ubuntu.com --recv 0E98404D386FA1D9 6ED0E7B82643E131
```

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%203.png)

Then install `smartmontools` 

```bash
apt install smartmontools/buster-backports
```

Create configuration file `/etc/ceph/ceph.conf` with the following content

```bash
[global]
fsid = 3d522417-9b07-4bb1-8a43-c8c867c642e4
mon initial members = sld-ceph-1
mon host = 192.168.0.97,192.168.0.124,192.168.0.216
public network = 192.168.0.0/24
cluster network = 192.168.0.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
```

**Then create keyring**

- Mon key
    
    ```bash
    ceph-authtool --create-keyring /tmp/monkey --gen-key -n mon. --cap mon 'allow *'
    ```
    
- client admin key
    
    ```bash
    root@sld-ceph-node-1:~# ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
    creating /etc/ceph/ceph.client.admin.keyring
    ```
    
- Client client.bootstrap-osd key
    
    ```bash
    root@sld-ceph-node-1:~# ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
    creating /var/lib/ceph/bootstrap-osd/ceph.keyring
    ```
    

Then import the key into a file

```bash
sudo ceph-authtool /tmp/monkey --import-keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-authtool /tmp/monkey --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
```

After importing, the `/tmp/monkey` file will have the following information

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%204.png)

Tự đọc tự hiểu nhé 🙂

Change user and group owner into `ceph:ceph` 

```bash
sudo chown ceph:ceph /tmp/monkey
```

Then create monitor map using `monmaptool` 

```bash
monmaptool --create --add {node1-id} {node1-ip} --fsid {cluster uuid} /tmp/monmap
monmaptool --add {node2-id} {node2-ip} --fsid {cluster uuid} /tmp/monmap
monmaptool --add {node3-id} {node3-ip} --fsid {cluster uuid} /tmp/monmap
```

Output

```bash
root@sld-ceph-node-1:~# monmaptool --create --add sld-ceph-node-1 192.168.0.34 --fsid 3176ee78-3524-4381-a672-cf44b1f6931f /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 3176ee78-3524-4381-a672-cf44b1f6931f
monmaptool: writing epoch 0 to /tmp/monmap (1 monitors)
root@sld-ceph-node-1:~# monmaptool --add sld-ceph-node-2 192.168.0.127 --fsid 3176ee78-3524-4381-a672-cf44b1f6931f /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 3176ee78-3524-4381-a672-cf44b1f6931f
monmaptool: writing epoch 0 to /tmp/monmap (2 monitors)
root@sld-ceph-node-1:~# monmaptool --add sld-ceph-node-3 192.168.0.142 --fsid 3176ee78-3524-4381-a672-cf44b1f6931f /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 3176ee78-3524-4381-a672-cf44b1f6931f
monmaptool: writing epoch 0 to /tmp/monmap (3 monitors)
```

Starting a new monitor is as easy as creating a new directory, creating the filesystem for and starting the service.

```
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-{node1-id}
sudo -u ceph ceph-mon --mkfs -i {node1-id} --monmap /tmp/monmap --keyring /tmp/monkey
sudo systemctl start ceph-mon@{node1-id}
```

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%205.png)

And to remove an Mon daemon, use command

```bash
service ceph -a stop mon.{moceph -sn-id}
ceph mon remove {mon-id}
```

Then remove it from the ceph.conf file

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%206.png)

Run `ceph -s` to check the status of the new mon daemon.

## Set up more nodes

Get all config file from the first node of the cluster

```bash
sudo scp {user}@{server}:/etc/ceph/ceph.conf /etc/ceph/ceph.conf
sudo scp {user}@{server}:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
sudo scp {user}@{server}:/var/lib/ceph/bootstrap-osd/ceph.keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
sudo scp {user}@{server}:/tmp/monmap /tmp/monmap
sudo scp {user}@{server}:/tmp/monkey /tmp/monkey
```

Change user and group owner into `ceph:ceph` 

```bash
sudo chown ceph:ceph /tmp/monkey
```

Then set up the monitor node

```bash
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-{node2-id}
sudo -u ceph ceph-mon --mkfs -i {node2-id} --monmap /tmp/monmap --keyring /tmp/monkey
sudo systemctl start ceph-mon@{node2-id}
sudo ceph -s
sudo ceph mon enable-msgr2
```

The result is now we have 3 monitor nodes

![Untitled](Install%20ceph%20manually%206ab2562249674430961d7c3eab4a3326/Untitled%207.png)

# Setup Manager Node

```bash
sudo ceph auth get-or-create mgr.{node1-id} mon 'allow profile mgr' osd 'allow *' mds 'allow *'
sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-{node1-id}
sudo -u ceph vi /var/lib/ceph/mgr/ceph-{node1-id}/keyring
sudo systemctl start ceph-mgr@{node1-id}
sudo ceph mgr module enable dashboard
sudo ceph dashboard create-self-signed-cert
sudo ceph dashboard ac-user-create admin -i passwd administrator
```

## Add storage device

Just 2 command

```bash
sudo ceph-volume lvm prepare --data /dev/sdb
sudo ceph-volume lvm activate {osd-number} {osd-uuid}
```

## Removing storage device

```bash
ceph osd out {osd-num}
```