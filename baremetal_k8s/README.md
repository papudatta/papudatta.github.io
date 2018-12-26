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

**Please change the hostname to the FQDN (like mynode.kube) in** `/etc/hostname` **and reboot**

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

I added a static route on the host iMac for service subnet 10.32.0.0/16
```sudo route add -net 10.32.0.0/24 192.168.1.112```


### - Prepare TLS certificates

Start by downloading prebuilt `cfssl` packages
```text
  $ curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  $ curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  $ chmod +x cfssl cfssljson
  $ sudo mv cfssl cfssljson /usr/local/bin/
```

**Generate Root CA**
```text
  cat > ca-config.json <<EOF
  {
    "signing": {
      "default": {
        "expiry": "26280h"
      },
      "profiles": {
        "kubernetes": {
          "usages": ["signing", "key encipherment", "server auth", "client auth"],
          "expiry": "26280h"
        }
      }
    }
  }
  EOF
  
  cat > ca-csr.json <<EOF
   {
     "CN": "Kubernetes",
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names": [
       {
         "C": "IN",
         "L": "BGLR",
         "O": "Kubernetes",
         "OU": "CKA"
       }
     ]
   }
  EOF
  
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

**Create certificate for admin**
```text
  cat > admin-csr.json <<EOF
  {
    "CN": "admin",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "IN",
        "L": "BGLR",
        "O": "system:masters",
        "OU": "CKA"
      }
    ]
  }
  EOF
  
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -profile=kubernetes \
    admin-csr.json | cfssljson -bare admin
```

**Create kubelet cerificates for worker nodes**
```text
  for instance in node1.home node2.home; do
    cat > ${instance}-csr.json <<EOF
    {
      "CN": "system:node:${instance}",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "IN",
          "L": "BGLR",
          "O": "system:nodes",
          "OU": "CKA"
        }
      ]
    }
    EOF
    
    IP=$(ping -c2 ${instance} | sed -nE 's/^PING[^(]+\(([^)]+)\).*/\1/p')
    
    cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -hostname=${instance},$IP \
      -profile=kubernetes \
      ${instance}-csr.json | cfssljson -bare ${instance}
  done
```

**Create kube-proxy cerificate**
```text
  cat > kube-proxy-csr.json <<EOF
  {
    "CN": "system:kube-proxy",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "IN",
        "L": "BGLR",
        "O": "system:node-proxier",
        "OU": "CKA"
      }
    ]
  }
  EOF
  
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -profile=kubernetes \
    kube-proxy-csr.json | cfssljson -bare kube-proxy
```

**Create api server cerificate**

Please note the SAN field containing master's IP, kubernetes api IP and name
```text
  cat > kubernetes-csr.json <<EOF
  {
    "CN": "kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "IN",
        "L": "BGLR",
        "O": "Kubernetes",
        "OU": "CKA"
      }
    ]
  }
  EOF
  
  
  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=192.168.1.111,10.32.0.1,127.0.0.1,kubernetes.default \
    -profile=kubernetes \
    kubernetes-csr.json | cfssljson -bare kubernetes
```

**Distribute the certificates to worker nodes**
```text
  for instance in node1.home node2.home; do
    scp ca.pem ${instance}-key.pem ${instance}.pem $instance:~/
  done
```

### - Generate kubeconfig files





### Jekyll Themes1

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/papudatta/papudatta.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.