## Introduction

Hello! My goal is to outline the detailed steps i followed in bringing up a kubernetes cluster on "bare-metal" Ubuntu VMs. All this was inspired by the exellent tutorials in [kubernetes_the_hard_way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and [Michael Champagne's blog](https://blog.csnet.me/)

Click [View On Github](https://github.com/papudatta/papudatta.github.io) above to report any issues with this procedure.

### Summary of contents

* Details of the environment followed by component verions
* Prepare TLS certificates
* Generate kubeconfig files
* Generate data encryption config
* Prepare ETCD cluster
* Prepare the master node
* Prepare the 2 nodes
* Generate kubectl config
* Configure networking
* Configure DNS
* Verify DNS
* Setup nginx ingress

### Environment

**Machine details**

| Hostname | IP Address | Memory | Disk space |
| --- | --- | --- | --- |
| **master.home** | 192.168.1.111 | 4GiB | 15GiB |
| **node1.home** | 192.168.1.112 | 3GiB | 20GiB |
| **node2.home** | 192.168.1.113 | 3GiB | 20GiB |

All the above VMs on an iMac were running Ubuntu 16.04 LTS.

**Component versions**

| Hostname | Version |
| --- | --- |
| **kubectl** | 1.11.1 |
| **kubelet** | 1.11.1 |
| **cfssl** | 1.2 |
| **etcd** | 3.3.9 |
| **CNI plugins** | 0.7.1 |
| **containerd** | 1.1.5 |
| **runc** | 1.0 rc6 |
| **containerd** | 1.1.5 |
| **crictl** | 1.11.1 |
| **weave net** | 2.5 |
| **coredns** | latest |

**Subnets used**

| Component | Subnet |
| --- | --- |
| **pod/cluster cidr** | 10.150.0.0/16 |
| **service cidr** | 10.32.0.0/24 |


### - Prepare TLS certificates

Start by downloading prebuilt `cfssl` packages
```text
  $ curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  $ curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  $ chmod +x cfssl cfssljson
  $ sudo mv cfssl cfssljson /usr/local/bin/
```

**Generate Root CA**


### Jekyll Themes1

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/papudatta/papudatta.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.