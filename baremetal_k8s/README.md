## Introduction

Hello! My goal is to outline the detailed steps I followed in bringing up a kubernetes cluster on "bare-metal" VMs running Ubuntu 18.04 LTS. This was inspired by the exellent tutorials in [kubernetes_the_hard_way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and [Michael Champagne's blog](https://blog.csnet.me/)

Click [View On Github](https://github.com/papudatta/papudatta.github.io) above to report any issues with this procedure.



### Summary of contents

* Details of the environment - OS, component versions, subnets used, etc
* Prepare TLS certificates
* Generate kubeconfig files
* Data encryption config
* Prepare ETCD cluster
* Prepare the master node
* Prepare the 2 nodes
* Generate kubectl config
* Configure networking
* Configure DNS
* Verify DNS
* Setup nginx ingress
* Setup rook for persistent storage requirements

### Environment

**Machine details**

| Hostname | IP Address | Memory | Disk space |
| --- | --- | --- | --- |
| **master.home** | 192.168.1.111 | 3GiB | 25GiB |
| **node1.home** | 192.168.1.112 | 4GiB | 15GiB |
| **node2.home** | 192.168.1.113 | 4GiB | 15GiB |

**Please change the hostname to the FQDN (like node1.home) in** `/etc/hostname` **and reboot**


**Component versions**

| Component | Version |
| --- | --- |
| **kubectl** | 1.13.2 |
| **kubelet** | 1.13.2 |
| **cfssl** | 1.2 |
| **etcd** | 3.3.9 |
| **docker** | 18.06.3 |
| **calico** | 3.4 |
| **coredns** | latest |

**Subnets used**

| Component | Subnet |
| --- | --- |
| **pod/cluster cidr** | 10.150.0.0/16 |
| **service cidr** | 10.32.0.0/24 |

I added a static route on the host iMac for service subnet 10.32.0.0/24

```$ sudo route add -net 10.32.0.0/24 192.168.1.112```

**master.home can log into both worker nodes using public key authentication**
```bash
$ ssh-keygen
$ ssh-copy-id node1.home
$ ssh-copy-id node2.home
```


### Prepare TLS certificates

Start by downloading prebuilt `cfssl` packages
```bash
$ curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssl cfssljson
$ sudo mv cfssl cfssljson /usr/local/bin/
```
**Verify**
```bash
$ cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

**Generate Root CA**
```bash
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
```bash
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

**Create kubelet certificates for worker nodes**
```bash
$ for instance in node1.home node2.home; do
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

**Create kube-controller-manager certificate**
```bash
$ cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BGLR",
      "O": "system:kube-controller-manager",
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
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

**Create kube-proxy certificate**
```bash
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

**Create kube-scheduler certificate**
```bash
$ cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BGLR",
      "O": "system:kube-scheduler",
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
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```


**Create api server certificate**

Please note the SAN field containing master's IP, kubernetes api IP and name
```bash
$ cat > kubernetes-csr.json <<EOF
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

**Create service-account certificate**
```bash
$ cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
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
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```


**Distribute the certificates to worker nodes**
```bash
  for instance in node1.home node2.home; do
    scp ca.pem ${instance}-key.pem ${instance}.pem $instance:~/
  done
```

### Generate kubeconfig files
**We will need kubectl**
```bash
  $ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kubectl
  $ chmod +x kubectl
  $ sudo mv kubectl /usr/local/bin/kubectl
```

These config files are required by kubelet and kube-proxy clients running on worker nodes in order to authenticate to kubernetes api server running on master node. We'll also create the cluster and context.
```bash
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

$ kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes \
    --user=admin \
    --kubeconfig=admin.kubeconfig

$ kubectl config use-context default --kubeconfig=admin.kubeconfig
```

**Distribute the config files**
```bash
$ for instance in node1.home node2.home; do
    scp ${instance}.kubeconfig kube-proxy.kubeconfig $instance:~/
done
```

### Data encryption config

This is to secure the data stored in etcd key/value database. The `--encryption-provider-config` flag in `kube-api` service will use this config file.
```bash
$ ENCRYPTION_KEY=`head -c 32 /dev/urandom | base64`
$ cat > encryption-config.yaml <<EOF
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
    - identity: {}
EOF
```

### Prepare ETCD cluster

**Download etcd, move the executables and copy required certs.**
```bash
$ wget -q --show-progress --https-only --timestamping \
    "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"

$ tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
$ sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
$ sudo mkdir -p /etc/etcd /var/lib/etcd
$ sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

**Create service file for etcd**
```bash
$ ETCD_NAME=$(hostname -f)
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
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable etcd
$ sudo systemctl start etcd
```

**Verify etcd operation**
```bash
$ sudo ETCDCTL_API=3 etcdctl member list \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ca.pem \
--cert=/etc/etcd/kubernetes.pem \
--key=/etc/etcd/kubernetes-key.pem
3f3a73a30a4872c4, started, master.home, https://192.168.1.111:2380, https://192.168.1.111:2379
```

### Prepare the master node

**Install kube-apiserver, kube-controller-manager and kube-scheduler binaries**
```bash
$ sudo mkdir -p /etc/kubernetes/config
$ wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kube-scheduler"

$ chmod +x kube-apiserver kube-controller-manager kube-scheduler
$ sudo mv kube-apiserver kube-controller-manager kube-scheduler /usr/local/bin/
$ sudo mkdir -p /var/lib/kubernetes/
$ sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

**Create service files for above components.**
We'll use **Node** and **RBAC** authorization mode.
```bash
$ cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=192.168.1.111 \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.1.111:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

$ cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.150.0.0/16 \\
  --allocate-node-cidrs=true \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

$ cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

$ cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Start the above services and verify if they are running**
```bash
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  $ sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
  $ for service in kube-apiserver kube-controller-manager kube-scheduler; do sudo systemctl status $service; done
```

**Check component statuses**
```bash
  $ kubectl get componentstatuses
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok                  
  scheduler            Healthy   ok                  
  etcd-0               Healthy   {"health":"true"}   
```

**Configure RBAC so that kubelet on worker nodes can authorize the api server**
```bash
$ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
  
$ cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

**Verify api server access**
```bash
$ curl --cacert ca.pem https://192.168.1.111:6443/version
{
  "major": "1",
  "minor": "13",
  "gitVersion": "v1.13.2",
  "gitCommit": "cff46ab41ff0bb44d8584413b598ad8360ec1def",
  "gitTreeState": "clean",
  "buildDate": "2019-01-10T23:28:14Z",
  "goVersion": "go1.11.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

### Prepare the 2 nodes

**Disable swap, enable ip forwarding, disable firewall and install socat, conntrack, ipset and docker on all worker nodes**
```bash
$ sudo apt-get update
$ sudo apt-get -y install socat conntrack ipset
$ sudo -i
# echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
# sysctl -p /etc/sysctl.conf
# swapoff -a ; edit /etc/fstab as well
# ufw disable
# exit

$ lsb_release -cs
$ sudo apt-get -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo apt-key fingerprint 0EBFCD88
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
$ sudo apt-get -y update
$ apt-cache madison docker-ce
$ sudo apt-get install -y docker-ce=18.06.3~ce~3-0~ubuntu
$ sudo usermod -aG docker $USER
$ sudo systemctl enable docker
$ exit  ; and log back 
$ docker run hello-world
```

**Install kubernetes binaries**
```bash
$ wget -q --show-progress --https-only --timestamping \
     https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kube-proxy \
     https://storage.googleapis.com/kubernetes-release/release/v1.13.2/bin/linux/amd64/kubelet

$ chmod +x kube-proxy kubelet
$ sudo  mv kube-proxy kubelet /usr/local/bin/
$ sudo mkdir -p /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
$ sudo cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
$ sudo cp ca.pem /var/lib/kubernetes/
$ sudo cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
$ sudo cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

**Create systemd unit files for above**
```bash
$ cat << EOF | sudo tee /etc/systemd/system/kubelet.service
  [Unit]
  Description=Kubernetes Kubelet
  Documentation=https://github.com/kubernetes/kubernetes
  After=docker.service
  Requires=docker.service
  
  [Service]
  ExecStart=/usr/local/bin/kubelet \\
    --allow-privileged=true \\
    --anonymous-auth=false \\
    --authorization-mode=Webhook \\
    --client-ca-file=/var/lib/kubernetes/ca.pem \\
    --cloud-provider= \\
    --cluster-dns=10.32.0.10 \\
    --cluster-domain=cluster.local \\
    --container-runtime=docker \\
    --docker=unix:///var/run/docker.sock \\
    --image-pull-progress-deadline=5m \\
    --kubeconfig=/var/lib/kubelet/kubeconfig \\
    --network-plugin=cni \\
    --pod-cidr=10.150.0.0/16 \\
    --register-node=true \\
    --runtime-request-timeout=15m \\
    --tls-cert-file=/var/lib/kubelet/$(hostname -f).pem \\
    --tls-private-key-file=/var/lib/kubelet/$(hostname -f)-key.pem \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
EOF
  
$ cat << EOF | sudo tee /etc/systemd/system/kube-proxy.service
  [Unit]
  Description=Kubernetes Kube Proxy
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  ExecStart=/usr/local/bin/kube-proxy \\
    --cluster-cidr=10.150.0.0/16 \\
    --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
    --proxy-mode=iptables \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
EOF
```

**Start and verify the above services**
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable kubelet kube-proxy
$ sudo systemctl start kubelet kube-proxy
$ for service in kubelet kube-proxy; do sudo systemctl status $service; done
```

### Configure networking with Calico
```bash
$ curl -O \
https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

**Wait till the pods are created**
```bash
$ kubectl get pod --namespace=kube-system -l k8s-app=calico-node -o wide
NAME                READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
calico-node-jcfxn   1/1     Running   0          2m32s   192.168.1.113   node2.home   <none>           <none>
calico-node-t2x7d   1/1     Running   0          2m32s   192.168.1.112   node1.home   <none>           <none>

$ kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node1.home   Ready    <none>   6m50s   v1.13.2   192.168.1.112   <none>        Ubuntu 18.04.2 LTS   4.15.0-47-generic   docker://18.6.3
node2.home   Ready    <none>   6m50s   v1.13.2   192.168.1.113   <none>        Ubuntu 18.04.2 LTS   4.15.0-47-generic   docker://18.6.3
```

### Configure DNS

**We'll use coredns and apply the yaml config here as well. First download the yaml and edit the IP**
```bash
$ wget https://raw.githubusercontent.com/mch1307/k8s-thw/master/coredns.yaml
```

Edited the IP, subnets in respective sections, as shown in below snippet:
```bash
   ...
   Corefile: |
       .:53 {
           errors
           health
           kubernetes cluster.local 10.150.0.0/16 10.32.0.0/24 { 
             pods insecure
             upstream
             fallthrough in-addr.arpa ip6.arpa
           }
   ...
   ...
   spec:
     selector:
       k8s-app: coredns
     clusterIP: 10.32.0.10 
   ...
```

**Apply the yaml file**
```bash
$ kubectl apply -f coredns.yaml

$ kubectl get pod --namespace=kube-system -l k8s-app=coredns -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
coredns-7848f666d8-kjxdr   1/1     Running   0          37s   10.150.0.2   node1.home   <none>           <none>
coredns-7848f666d8-stjxp   1/1     Running   0          37s   10.150.1.2   node2.home   <none>           <none>
```

### Verify DNS
```bash
$ kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
    If you don't see a command prompt, try pressing enter.
    dnstools# host kubernetes
    kubernetes.default.svc.cluster.local has address 10.32.0.1
    dnstools# host kube-dns.kube-system
    kube-dns.kube-system.svc.cluster.local has address 10.32.0.10
    dnstools# host 10.32.0.10
    10.0.32.10.in-addr.arpa domain name pointer kube-dns.kube-system.svc.cluster.local.
    dnstools# exit
    pod "dnstools" deleted
```

### Setup nginx ingress
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml

$ kubectl get svc --all-namespaces
NAMESPACE       NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
default         kubernetes      ClusterIP   10.32.0.1     <none>        443/TCP                      56m
ingress-nginx   ingress-nginx   NodePort    10.32.0.129   <none>        80:30989/TCP,443:30686/TCP   115s
kube-system     calico-typha    ClusterIP   10.32.0.225   <none>        5473/TCP                     10m
kube-system     kube-dns        ClusterIP   10.32.0.10    <none>        53/UDP,53/TCP                4m21s

$ kubectl get deployment --all-namespaces
NAMESPACE       NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx   nginx-ingress-controller   1/1     1            1           2m12s
kube-system     calico-typha               0/0     0            0           10m
kube-system     coredns                    2/2     2            2           4m26s

$ kubectl get pods --all-namespaces -o wide
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE    IP              NODE         NOMINATED NODE   READINESS GATES
ingress-nginx   nginx-ingress-controller-689498bc7c-vw9fx   1/1     Running   0          111s   10.150.1.3      node2.home   <none>           <none>
kube-system     calico-node-jcfxn                           1/1     Running   0          10m    192.168.1.113   node2.home   <none>           <none>
kube-system     calico-node-t2x7d                           1/1     Running   0          10m    192.168.1.112   node1.home   <none>           <none>
kube-system     coredns-7848f666d8-kjxdr                    1/1     Running   0          4m5s   10.150.0.2      node1.home   <none>           <none>
kube-system     coredns-7848f666d8-stjxp                    1/1     Running   0          4m5s   10.150.1.2      node2.home   <none>           <none>
```

*In case we should be required to use an ingress version different from the latest*
```bash
$ kubectl set image deployment/nginx-ingress-controller -n \
    ingress-nginx nginx-ingress-controller=quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
```

**To be able to pull images from a different registry like gcr.io**
```bash
$ kubectl create secret docker-registry mydockercfg \
  --docker-server="https://gcr.io" \
  --docker-username=DOCKER_USER \
  --docker-email=DOCKER_EMAIL \
  --docker-password=DOCKER_PASSWORD

$ cat pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  imagePullSecrets:
  - name: mydockercfg  <===
  ...
  ...
```


### Setup rook for persistent volume requirements

```bash
$ git clone https://github.com/rook/rook
$ cd rook/cluster/examples/kubernetes/ceph
$ git checkout remotes/origin/release-0.9
$ git branch -a

$ kubectl create -f operator.yaml

$ kubectl get pods -n rook-ceph-system -o wide
NAME                                  READY     STATUS    RESTARTS   AGE       IP              NODE
rook-ceph-agent-qjhbz                 1/1       Running   0          2m        192.168.1.113   node2.home
rook-ceph-agent-vr8n5                 1/1       Running   0          2m        192.168.1.112   node1.home
rook-ceph-operator-64454f9bf5-rzrrt   1/1       Running   0          5m        10.150.128.2    node2.home
rook-discover-s6r2h                   1/1       Running   0          2m        10.150.128.3    node2.home
rook-discover-x49k6                   1/1       Running   0          2m        10.150.0.4      node1.home

$ kubectl create -f cluster.yaml

$ kubectl get pods -n rook-ceph -o wide
NAME                                     READY     STATUS      RESTARTS   AGE       IP             NODE
rook-ceph-mgr-a-55bb9c6474-rsjhd         1/1       Running     0          42s       10.150.128.5   node2.home
rook-ceph-mon-a-55587db7d5-ttcgt         1/1       Running     0          2m        10.150.0.6     node1.home
rook-ceph-mon-b-c856d479d-5n9t8          1/1       Running     0          1m        10.150.128.4   node2.home
rook-ceph-osd-0-5bb87d785-wwq4c          1/1       Running     0          15s       10.150.0.7     node1.home
rook-ceph-osd-1-d66cc6fd-xlppl           1/1       Running     0          13s       10.150.128.7   node2.home
rook-ceph-osd-prepare-node1.home-2h9h4   0/2       Completed   1          24s       10.150.0.5     node1.home
rook-ceph-osd-prepare-node2.home-d698s   0/2       Completed   1          24s       10.150.128.6   node2.home

$ kubectl create -f storageclass.yaml
```

**We'll make this storage class the default one**
```bash
$ kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

$ kubectl get sc
NAME                        PROVISIONER          AGE
rook-ceph-block (default)   ceph.rook.io/block   1m
```
