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

```$ sudo route add -net 10.32.0.0/24 192.168.1.112```


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
  $ cat > ca-config.json <<EOF
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
  
  $ cat > ca-csr.json <<EOF
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
  
  $ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

**Create certificate for admin**
```text
  $ cat > admin-csr.json <<EOF
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
  
  $ cfssl gencert \
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
  $ cat > kube-proxy-csr.json <<EOF
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
  
  $ cfssl gencert \
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
  
  
  $ cfssl gencert \
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
**We will need kubelet**
```bash
  $ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/amd64/kubectl
  $ chmod +x kubectl
  $ sudo mv kubectl /usr/local/bin/kubectl
```

These config files are required by kubelet and kube-proxy clients running on worker nodes in order to authenticate to kubernetes api server running on master node. We'll also create the cluster and context.
```text
$ for instance in node1.home node2.home; do
  kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.1.111:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done

$ kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.1.111:6443 \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

**Distribute the config files**
```text
$ for instance in node1.home node2.home; do
    scp ${instance}.kubeconfig kube-proxy.kubeconfig $instance:~/
done
```

### - Generate data encryption config

This is to secure the data stored in etcd key/value database. The `--experimental-encryption-provider-config` flag in `kube-api` service will use this config file.
```text
  $ ENCRYPTION_KEY=`head -c 32 /dev/urandom | base64`
  $ cat > encryption-config.yaml <<EOF
  kind: EncryptionConfig
  apiVersion: v1
  resources:
    - resources:
        - secrets
      providers:
        - aescbc:
            keys:
              - name: key
                secret: ${ENCRYPTION_KEY}
        - identity: {}
  EOF
```

### - Prepare ETCD cluster

**Download etcd, move the executables and copy required certs.**
```text
  $ wget -q --show-progress --https-only --timestamping \
      "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
  
  $ tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  $ sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
  $ sudo mkdir -p /etc/etcd /var/lib/etcd
  $ sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
  $ sudo cp ca.pem /etc/ssl/certs/
```

**Create service file for etcd**
```text
  $ ETCD_NAME=`hostname -f`
  $ cat << EOF | sudo tee /etc/systemd/system/etcd.service
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos
  
  [Service]
  ExecStart=/usr/local/bin/etcd \\
    --name $ETCD_NAME \\
    --cert-file=/etc/etcd/kubernetes.pem \\
    --key-file=/etc/etcd/kubernetes-key.pem \\
    --peer-cert-file=/etc/etcd/kubernetes.pem \\
    --peer-key-file=/etc/etcd/kubernetes-key.pem \\
    --trusted-ca-file=/etc/etcd/ca.pem \\
    --peer-trusted-ca-file=/etc/etcd/ca.pem \\
    --peer-client-cert-auth \\
    --client-cert-auth \\
    --initial-advertise-peer-urls https://192.168.1.111:2380 \\
    --listen-peer-urls https://192.168.1.111:2380 \\
    --listen-client-urls https://192.168.1.111:2379,https://127.0.0.1:2379 \\
    --advertise-client-urls https://192.168.1.111:2379 \\
    --initial-cluster-token etcd-cluster-0 \\
    --initial-cluster master.home=https://192.168.1.111:2380 \\
    --initial-cluster-state new \\
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
  EOF
```

**Start the etc service**
```text
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

**Verify etcd operation**
```text
  $ export ETCDCTL_CACERT=/etc/etcd/ca.pem
  $ export ETCDCTL_CERT=/etc/etcd/kubernetes.pem
  $ export ETCDCTL_KEY=/etc/etcd/kubernetes-key.pem
  $ sudo chown $(whoami):$(whoami) /etc/etcd/*
  $ ETCDCTL_API=3 etcdctl member list \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.pem \
    --cert=/etc/etcd/kubernetes.pem \
    --key=/etc/etcd/kubernetes-key.pem
```

