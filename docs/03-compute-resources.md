# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute region](https://docs.digitalocean.com/products/platform/availability-matrix/).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region) lab.

## Create a new project

DigitalOcean Projects allow you to organize your DigitalOcean resources (like Droplets, Spaces, load balancers, domains, and floating IPs) into groups that fit the way you work.

```sh
doctl projects create --name kubernetes-the-hard-way \
  --description "Kubernetes tutorial project" \
  --purpose "Class project / Educational purposes" \
  --format "ID, Name, IsDefault, CreatedAt"
```

> output

```
ID                                      Name                       Is Default?    Created At
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    kubernetes-the-hard-way    false          2023-03-15T03:19:16Z
```

Set your new project to the ID in the returned output

```sh
doctl projects update xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --is_default \
  --format "ID, Name, IsDefault, CreatedAt"

```

### Verification

```sh
doctl projects list --format "ID, Name, IsDefault, CreatedAt"
```

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.digitalocean.com/products/networking/vpc/) (VPC) network will be setup to host the Kubernetes cluster.

A [subnet](https://docs.digitalocean.com/products/networking/vpc/) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

> The `10.240.0.0/24` IP address range used below can host up to 254 compute instances.

Create the `kubernetes-the-hard-way` VPC network with subnet:

```sh
doctl vpcs create --name kubernetes-the-hard-way \
  --description "Kubernetes tutorial network" \
  --ip-range 10.240.0.0/24
```

> output

```
ID                                      URN                                            Name                       Description                    IP Range         Region    Created At                                 Default
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    do:vpc:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    kubernetes-the-hard-way    Kubernetes tutorial network    10.240.0.0/24    sfo3      2023-03-15 04:28:44.517360199 +0000 UTC    false
```

Set your new VPC as the default for the region

```sh
doctl vpcs update xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx --default
```

### Firewall Rules

Create a firewall with a rule that allow internal communication across all protocols and a firewall rule that allows external SSH, ICMP, and HTTPS:

##### TODO: pretty this up

```sh
doctl compute firewall create \
  --name kubernetes-the-hard-way \
  --tag-names kubernetes-the-hard-way \
  --outbound-rules "protocol:tcp,ports:0,address:0.0.0.0/0 protocol:udp,ports:0,address:0.0.0.0/0 protocol:icmp,address:0.0.0.0/0" \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0 protocol:tcp,ports:6443,address:0.0.0.0/0 protocol:icmp,address:0.0.0.0/0"
```

> An [external load balancer](https://docs.digitalocean.com/products/networking/load-balancers/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:


```sh
doctl compute firewall list --format "Name, Status, Created, InboundRules, OutboundRules, Tags"
```
> output

```
Name                       Status       Created At              Inbound Rules                                                 Outbound Rules                                                                                                       Tags
kubernetes-the-hard-way    succeeded    2023-03-15T22:16:27Z    protocol:icmp, protocol:tcp,ports:0, protocol:udp,ports:0,    protocol:icmp,address:0.0.0.0/0 protocol:tcp,ports:22,address:0.0.0.0/0 protocol:tcp,ports:6443,address:0.0.0.0/0    kubernetes-the-hard-way
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers using the ID you got while [creating your project earlier](#Create-a-new-project):

```sh
doctl compute reserved-ip create \
  --project-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --format "IP, Region, ProjectID"
```

> output

```
IP                 Region    Project ID
XXX.XXX.XXX.XXX    sfo3      xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Upload a SSH public key

DigitalOcean allows you to add SSH public keys to the interface so that you can embed your public key into a Droplet at the time of creation. Only the public key is required to take advantage of this functionality.

```sh
doctl compute ssh-key create ships-captain \
  --public-key $K8S_HARD_SSH_PUB_KEY
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:


```sh
for i in 0 1 2; do
  doctl compute droplet create controller-${i} \
    --size s-1vcpu-1gb \
    --image ubuntu-22-04-x64 \    --enable-monitoring \
    --enable-private-networking \
    --tag-names kubernetes-the-hard-way,controller \
    --format "ID, Name, Memory, VCPUs, Disk, Region, Image, Status"
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```sh
for i in 0 1 2; do
  doctl compute droplet create worker-${i} \
    --size s-1vcpu-1gb \
    --image ubuntu-22-04-x64 \
    --enable-monitoring \
    --enable-private-networking \
    --tag-names kubernetes-the-hard-way,worker \
    --user-data pod-cidr=10.200.${i}.0/24 \
    --format "ID, Name, Memory, VCPUs, Disk, Region, Image, Status" 
done
```

### Verification

List the compute instances in your default compute zone:

```sh
doctl compute droplet list \
  --format "ID, Name, PublicIPv4, PrivateIPv4, Region, Image, Status, Features"
```

> output

```
ID           Name               Public IPv4        Private IPv4    Region    Image                     Status
xxxxxxxxx    controller-0       xxx.xxx.xxx.xxx    10.240.0.3      sfo3      Ubuntu 20.04 (LTS) x64    active
xxxxxxxxx    worker-0           xxx.xxx.xxx.xxx    10.240.0.2      sfo3      Ubuntu 20.04 (LTS) x64    active
```

## Validate SSH Access

SSH will be used to connect to compute instances. Refer to the [Connect With SSH](https://docs.digitalocean.com/products/droplets/how-to/connect-with-ssh/) documentation.

Test SSH access to the `controller-0` compute instances:

```
doctl compute ssh controller-0
```

You'll be prompted to enter the `root` password that was emailed to you when you created the droplet, and then forced to change it immediately.

After the you've been successfully connected you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-50-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Mar 15 22:42:08 UTC 2023

  System load:  0.0               Users logged in:       0
  Usage of /:   6.5% of 24.05GB   IPv4 address for eth0: xxx.xxx.xxx.xxx
  Memory usage: 22%               IPv4 address for eth0: xxx.xxx.xxx.xxx
  Swap usage:   0%                IPv4 address for eth1: xxx.xxx.xxx.xxx
  Processes:    92

0 updates can be applied immediately.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XX.XX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
