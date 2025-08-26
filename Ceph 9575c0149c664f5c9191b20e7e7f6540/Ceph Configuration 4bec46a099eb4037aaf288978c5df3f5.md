# Ceph Configuration

# 1. Configuration

## 2.1 Configuring ceph

A ceph cluster run at least 3 types of daemon:

- Ceph Monitor (`ceph-mon`)
- Ceph Manager (`ceph-mgr`)
- Ceph OSD Daemon (`ceph-osd`)

If ceph cluster provide **File System service**, then it need Ceph metadata server  (ceph -mds) in addition. and if cluster provides Object Storage, it also run Ceph RADOS gateway daemons (radosgw)

## 2.2 Option name

Option name can be specified in config filem in command line, environment variable,…

In command line, example, option `—mon_host` is equivalent to `—mon-host`

In configuration file, we can replace dash (-) and underscore (_) to space.

## 2.3 Config source

The configuration source comes from one of the following, and if one of the latter source is available, then it will override the exsting one.

- the compiled-in default value
- the monitor cluster’s centralized configuration database
- a configuration file stored on the local host
- environment variables
- command-line arguments
- runtime overrides that are set by an administrator

## 2.4 Bootstrap Options

mon_host and mon_host_override

used to specify the monitor node

**Example**

```bash
[global]
mon_host = 192.168.1.100:6789
mon_host = 192.168.1.101:6789
mon_host = 192.168.1.102:6789
```

## 2.5 Configuration Sections

- Global : affect all daemon and client in the Ceph Cluster
- mon : affect al ceph-mon daemon in the cluster, override setting inside global
- mgr : affects all ceph-mgr daemons in the cluster, override the same setting in global
- osd : affect all ceph-osd daemons in the cluster, verride the same setting in global
- mds : affects al ceph-mds daemon
- client affect all ceph client

We can specify an individual daemon or client name. For example: **[mon.sld]** affect the **sld** monitor daemon only 

## 2.6 Metavariables

Like a variable in the configuration file, that store metadata of the cluster

Some common metavariable

- $cluster : cluster name
    
    Example : `/etc/ceph/$cluster.keyring` . The value of ***$cluster*** is the cluster’s name.
    
- $type : daemon process type (`mds`, `osd`, `mon`)
- $host : host name where the process is running
- $name : =  $type.$id
- $pid : daemon id

## 2.7 Configuration file

On startup, Ceph seach for configuration file in one of the following

1. `$CEPH_CONF` (that is, the path following the `$CEPH_CONF` environment variable)
2. `c path/path` (that is, the `c` command line argument)
3. `/etc/ceph/$cluster.conf`
4. `~/.ceph/$cluster.conf`
5. `./$cluster.conf` (that is, in the current working directory)
6. On FreeBSD systems only, `/usr/local/etc/ceph/$cluster.conf`

## 2.8 Monitor configuration database

Monitor cluster manage a database of configuration option for ease of management and transparency.

## 2.9 Command

Command to configure the cluster

- `ceph config dump` dumps the entire monitor configuration database for the cluster.
- `ceph config get <who>` dumps the configuration options stored in the monitor configuration database for a specific daemon or client (for example, `mds.a`).
- `ceph config get <who> <option>` shows either a configuration value stored in the monitor configuration database for a specific daemon or client (for example, `mds.a`), or, if that value is not present in the monitor configuration database, the compiled-in default value.
- `ceph config set <who> <option> <value>` specifies a configuration option in the monitor configuration database. It change the configuration at ***run time.***
- `ceph config show <who>` shows the configuration for a running daemon. These settings might differ from those stored by the monitors if there are also local configuration files in use or if options have been overridden on the command line or at run time. The source of the values of the options is displayed in the output.
- `ceph config assimilate-conf -i <input file> -o <output file>` : Help to move local configuration to a centralized database.