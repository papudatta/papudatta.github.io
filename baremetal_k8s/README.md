## Introduction

Hello! My goal is to outline the detailed steps i followed in bringing up a kubernetes cluster on "bare-metal" Ubuntu VMs. All this was inspired by the exellent tutorials in [kubernetes_the_hard_way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and [Michael Champagne's blog](https://blog.csnet.me/)

Click [View On Github](https://github.com/papudatta/papudatta.github.io) above to report any issues with this procedure.

### Summary of contents

* Details of the environment followed by component verions
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

### - Data encryption config

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

### - Prepare the master node

**Install kube-apiserver, kube-controller-manager and kube-scheduler binaries**
```text
  $ wget -q --show-progress --https-only --timestamping \
    "https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/amd64/kube-apiserver" \
    "https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/  amd64/kube-controller-manager" \
    "https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/amd64/kube-scheduler"

  $ chmod +x kube-apiserver kube-controller-manager kube-scheduler
  $ sudo mv kube-apiserver kube-controller-manager kube-scheduler /usr/local/bin/
  $ sudo mkdir -p /var/lib/kubernetes/
  $ sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem encryption-config.yaml /var/lib/kubernetes/
```

**Create service files for above components**
```text
  $ cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
  [Unit]
  Description=Kubernetes API Server
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  ExecStart=/usr/local/bin/kube-apiserver \\
    --admission-control=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,Defau  ltStorageClass,ResourceQuota \\
    --advertise-address=192.168.1.111 \\
    --allow-privileged=true \\
    --apiserver-count=2 \\
    --audit-log-maxage=30 \\
    --audit-log-maxbackup=2 \\
    --audit-log-maxsize=100 \\
    --audit-log-path=/var/log/audit.log \\
    --authorization-mode=Node,RBAC \\
    --bind-address=0.0.0.0 \\
    --client-ca-file=/var/lib/kubernetes/ca.pem \\
    --enable-swagger-ui=true \\
    --etcd-cafile=/var/lib/kubernetes/ca.pem \\
    --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
    --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
    --etcd-servers=https://192.168.1.111:2379 \\
    --event-ttl=1h \\
    --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
    --insecure-bind-address=127.0.0.1 \\
    --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
    --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
    --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
    --kubelet-https=true \\
    --runtime-config=api/all \\
    --service-account-key-file=/var/lib/kubernetes/ca-key.pem \\
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

  $ cat << EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  ExecStart=/usr/local/bin/kube-controller-manager \\
    --address=0.0.0.0 \\
    --cluster-cidr=10.150.0.0/16 \\
    --cluster-name=kubernetes \\
    --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
    --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
    --leader-elect=true \\
    --master=http://127.0.0.1:8080 \\
    --root-ca-file=/var/lib/kubernetes/ca.pem \\
    --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \\
    --service-cluster-ip-range=10.32.0.0/24 \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  $ cat << EOF | sudo tee /etc/systemd/system/kube-scheduler.service
  [Unit]
  Description=Kubernetes Scheduler
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  ExecStart=/usr/local/bin/kube-scheduler \\
    --leader-elect=true \\
    --master=http://127.0.0.1:8080 \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
  EOF
```

**Start the above services and verify if they were started**
```text
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  $ sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
  $ for service in kube-apiserver kube-controller-manager kube-scheduler; do sudo systemctl status $service; done
```

**Check component statuses**
```text
  $ kubectl get componentstatuses
  NAME                 STATUS    MESSAGE             ERROR
  controller-manager   Healthy   ok                  
  scheduler            Healthy   ok                  
  etcd-0               Healthy   {"health":"true"}   
```

**Configure RBAC so that kubelet can authorize the api server**
```text
  $ cat <<EOF | kubectl apply -f -
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
  
  
  $ cat <<EOF | kubectl apply -f -
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
```text
  $ curl -k https://192.168.1.111:6443/version
  {
    "major": "1",
    "minor": "11",
    "gitVersion": "v1.11.1",
    "gitCommit": "b1b29978270dc22fecc592ac55d903350454310a",
    "gitTreeState": "clean",
    "buildDate": "2018-07-17T18:43:26Z",
    "goVersion": "go1.10.3",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
```

### - Prepare the 2 nodes

**Disable swap, enable ip forwarding, disable firewall, install, install socat and conntrack on all nodes**
```text
  $ sudo apt install socat conntrack
  =====> Add net.ipv4.ip_forward=1  to  /etc/sysctl.conf
  $ sudo sysctl -p /etc/sysctl.conf
  $ sudo swapoff -a  (/etc/fstab as well)
  $ sudo ufw disable
```

**Install CNI plugins, containerd and runc**
```text
  $ wget -q --show-progress --https-only --timestamping \
      "https://github.com/containernetworking/plugins/releases/download/  v0.7.1/cni-plugins-amd64-v0.7.1.tgz" \
      "https://github.com/containerd/containerd/releases/download/  v1.1.5/containerd-1.1.5.linux-amd64.tar.gz" \
      "https://github.com/opencontainers/runc/releases/download/v1.0.0-rc6/runc.amd64"

  $ sudo tar xvf cni-plugins-amd64-v0.7.1.tgz -C /opt/cni/bin
  $ sudo tar xvf containerd-1.1.5.linux-amd64.tar.gz -C /usr/local/
  $ mv runc.amd64 runc
  $ chmod +x runc
  $ sudo mv runc /usr/local/bin/
```

**Create systemd unit file for containerd**
```text
cat << EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

**Download and configure crictl for interacting with container runtime**
```text
  $ wget -q --show-progress --https-only --timestamping \
    https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.11.1/crictl-v1.11.1-linux-amd64.tar.gz
  $ tar xvf crictl-v1.11.1-linux-amd64.tar.gz
  $ chmod +x crictl
  $ sudo mv crictl /usr/local/bin
  $ cat << EOF | sudo tee /etc/crictl.yaml
  runtime-endpoint: unix:///var/run/containerd/containerd.sock
  timeout: 10
  EOF
```

**Install kubernetes binaries**
```text
  $ wget -q --show-progress --https-only --timestamping \
     https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/amd64/kube-proxy \
     https://storage.googleapis.com/kubernetes-release/release/v1.11.1/bin/linux/amd64/kubelet

  $ chmod +x kube-proxy kubelet
  $ sudo  mv kube-proxy kubelet /usr/local/bin/
  $ sudo mkdir -p /var/lib/kubelet \
    /var/lib/kube-proxy \
    /var/lib/kubernetes \
    /var/run/kubernetes
  $ sudo mv $(hostname -f)-key.pem $(hostname -f).pem /var/lib/kubelet/
  $ sudo mv ca.pem /var/lib/kubernetes/
  $ sudo mv $(hostname -f).kubeconfig /var/lib/kubelet/kubeconfig
  $ sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

**Create systemd unit files for above**
```text
  $ cat << EOF | sudo tee /etc/systemd/system/kubelet.service
  [Unit]
  Description=Kubernetes Kubelet
  Documentation=https://github.com/kubernetes/kubernetes
  After=containerd.service
  Requires=containerd.service
  
  [Service]
  ExecStart=/usr/local/bin/kubelet \\
    --allow-privileged=true \\
    --anonymous-auth=false \\
    --authorization-mode=Webhook \\
    --client-ca-file=/var/lib/kubernetes/ca.pem \\
    --cloud-provider= \\
    --cluster-dns=10.32.0.10 \\
    --cluster-domain=cluster.local \\
    --container-runtime=remote \\
    --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
    --image-pull-progress-deadline=2m \\
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
```text
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable containerd kubelet kube-proxy
  $ sudo systemctl start containerd kubelet kube-proxy
  $ for service in containerd kubelet kube-proxy; do sudo systemctl status $service; done
```

**I had to change containerd component ownerships. Gathering more details on this meanwhile**
```text
  $ sudo chown $(whoami):$(whoami) /run/containerd/containerd.sock
  $ sudo chown -R $(whoami):$(whoami) /var/run/containerd/io.containerd.runtime.v1.linux
  $ sudo chown -R $(whoami):$(whoami) /var/run/containerd/runc
```

### - Generate kubectl config

**we are back in master.node**
```text
  $ kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.1.111:6443
  
  $ kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem
  
  $ kubectl config set-context kubernetes \
    --cluster=kubernetes \
    --user=admin
  
  $ kubectl config use-context kubernetes
```

### - Configure networking

**We'll use weave net and apply the yaml config**
```text
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.150.0.0/16"
```

**Wait till the pods are created**
```text
  $ kubectl get pod --namespace=kube-system -l name=weave-net -o wide
  NAME              READY     STATUS    RESTARTS   AGE       IP              NODE
  weave-net-7kswf   2/2       Running   0          8m        192.168.1.112   node1.home
  weave-net-jx68d   2/2       Running   0          8m        192.168.1.113   node2.home
```

### - Configure DNS

**We'll use coredns and apply the yaml config here as well. First download the yaml and edit the IP**
```text
  $ wget https://raw.githubusercontent.com/mch1307/k8s-thw/master/coredns.yaml
```

Edited the IP in respective sections, as shown in below snippet:
```text
   ...
   Corefile: |
       .:53 {
           errors
           health
           kubernetes cluster.local 10.32.0.0/24 { 
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
```text
$ kubectl apply -f coredns.yaml

$ kubectl get pod -n kube-system -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP              NODE
coredns-5f7d467445-5xj24   1/1       Running   0          56s       10.150.128.1    node2.home
coredns-5f7d467445-gnmfd   1/1       Running   0          56s       10.150.0.2      node1.home
weave-net-7kswf            2/2       Running   0          10m       192.168.1.112   node1.home
weave-net-jx68d            2/2       Running   0          10m       192.168.1.113   node2.home
```

### - Verify DNS
```text
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

### - Setup nginx ingress
```text
  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/  service-nodeport.yaml

  $ kubectl get svc --all-namespaces 
  NAMESPACE       NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
  default         kubernetes      ClusterIP   10.32.0.1    <none>        443/TCP                      54m
  ingress-nginx   ingress-nginx   NodePort    10.32.0.34   <none>        80:32106/TCP,443:30158/TCP   1m
  kube-system     kube-dns        ClusterIP   10.32.0.10   <none>        53/UDP,53/TCP                5m

  $ kubectl get deployment --all-namespaces 
  NAMESPACE       NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  ingress-nginx   nginx-ingress-controller   1         1         1            1           2m
  kube-system     coredns                    2         2         2            2           6m
  
  $ kubectl get pods --all-namespaces -o wide
  NAMESPACE       NAME                                       READY     STATUS    RESTARTS   AGE       IP              NODE
  ingress-nginx   nginx-ingress-controller-87554c57b-tpfd2   1/1       Running   0          2m        10.150.0.3      node1.home
  kube-system     coredns-5f7d467445-5xj24                   1/1       Running   0          6m        10.150.128.1    node2.home
  kube-system     coredns-5f7d467445-gnmfd                   1/1       Running   0          6m        10.150.0.2      node1.home
  kube-system     weave-net-7kswf                            2/2       Running   0          16m       192.168.1.112   node1.home
  kube-system     weave-net-jx68d                            2/2       Running   0          16m       192.168.1.113   node2.home
```