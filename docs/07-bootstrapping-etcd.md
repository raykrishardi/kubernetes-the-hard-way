# Bootstrapping (i.e. install, configure, and start on system boot up) the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd) which means etcd is stateful. In this lab you will bootstrap a two node etcd cluster and configure it for high availability and secure remote access. `etcd` uses 2 ports (2379 and 2380), 2379 is used for etcdctl (client)/kube-apiserver to configure etcd server while 2380 is used for etcd peer-to-peer communication using RAFT protocol for leader election process.

## Prerequisites

The commands in this lab must be run on each controller instance: `master-1`, and `master-2`. Login to each of these using an SSH terminal.

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```
{
  # /etc/etcd contains the configuration files and certificates (CA.crt, etcd server crt and key files)
  # /var/lib/etcd contains all the kubernetes data/objects (e.g. secrets, configmaps, pod configs, etc)
  sudo mkdir -p /etc/etcd /var/lib/etcd
  
  sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address of the master(etcd) nodes:

```
INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

```
ETCD_NAME=$(hostname -s)

# Result
master-1 OR master-2
```

Create the `etcd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/etcd-server.crt \\
  --key-file=/etc/etcd/etcd-server.key \\
  --peer-cert-file=/etc/etcd/etcd-server.crt \\
  --peer-key-file=/etc/etcd/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-1=https://192.168.5.11:2380,master-2=https://192.168.5.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
NOTE:
*--name ${ETCD_NAME} -> etcd member name in the etcd cluster (MUST be unique, in this case set to master-1 OR master-2 (hostname of the system))

peer-to-peer comms (no 127.0.0.1 because this is used between peers)
--initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
--listen-peer-urls https://${INTERNAL_IP}:2380 \\
--initial-cluster master-1=https://192.168.5.11:2380,master-2=https://192.168.5.12:2380 \\
  
client-to-server comms (there's 127.0.0.1 so that you could use etcdctl from within the host itself, API server will use https://${INTERNAL_IP}:2379)
The only place that uses port 2379 (others are using 2380)
--listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
--advertise-client-urls https://${INTERNAL_IP}:2379 \\

Change the etcd cluster ID/token when restoring from etcd snapshot
--initial-cluster-token etcd-cluster-0 \\

Where etcd stores all kubernetes data/object
--data-dir=/var/lib/etcd
```

### Start the etcd Server

```
{
  # reload all systemd unit files (i.e. *.service files), when you make changes to your unit file (in this case, adding new systemd unit, there should be NO downtime)
  sudo systemctl daemon-reload 
  
  # Ensure that etcd starts on system boot and start etcd service
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> Remember to run the above commands on each controller node: `master-1`, and `master-2`.

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

> Remember to specify endpoints, ca crt, crt, and key everytime you make a call to the etcd server

```
# Ensure that etcd API v3 is used for the remaining session of the shell
export ETCDCTL_API=3

You could check whether etcd API 2 or 3 by:
-Command available in API 2 (etcdctl -v)
-The above command is NOT available in API 3 (instead it's available from etcdctl version)
```

> output

```
45bf9ccad8d8900a, started, master-2, https://192.168.5.12:2380, https://192.168.5.12:2379
54a5796a6803f252, started, master-1, https://192.168.5.11:2380, https://192.168.5.11:2379
```

## Check etcd logs (using journalctl)
```
journalctl -u etcd.service
```

Reference: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#starting-etcd-clusters

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
