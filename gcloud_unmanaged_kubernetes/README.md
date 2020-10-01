# Here is the procedure for coming up with a similar cluster in Google cloud ...

### Summary of contents

* Setting up the environment - network, subnet, firewall rules, VMs and external IP
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

**Create network, subnet, firewall rules, VMs and external IP**
```bash
gcloud config set compute/region us-east1
gcloud config set compute/zone us-east1-c
gcloud compute networks create pilot --subnet-mode custom
gcloud compute networks subnets create knet --network pilot --range 10.241.0.0/24
gcloud compute firewall-rules create pilot-allow-internal --allow tcp,udp,icmp --network pilot --source-ranges 10.241.0.0/24,10.151.0.0/16
gcloud compute firewall-rules create pilot-allow-external --allow tcp:22,tcp:6443,icmp --network pilot --source-ranges 0.0.0.0/0
gcloud compute firewall-rules list
gcloud compute firewall-rules list --filter="network:pilot"
gcloud config get-value compute/region
gcloud compute addresses create pilot --region us-east1
gcloud compute addresses list --filter="name=('pilot')"


for i in 0 1; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-minimal-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-2 \
    --metadata pod-cidr=10.151.0.0/16 \
    --private-network-ip 10.241.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet knet \
    --tags pilot,controller,worker
done

for i in 0 1; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-minimal-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-2 \
    --metadata pod-cidr=10.151.0.0/16 \
    --private-network-ip 10.241.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet knet \
    --tags pilot,worker
done
```

**Enable OS login, verify instance creation and ssh into them**
```bash
for i in 0 1; do
    gcloud compute instances add-metadata worker-${i} --metadata enable-oslogin=TRUE
  done

for i in 0 1; do
  gcloud compute instances add-metadata controller-${i} --metadata enable-oslogin=TRUE
done

gcloud compute instances list
gcloud compute ssh controller-0
```

**We'll need cfssl, kubectl on one of the machines. I chose controller-0**
```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64   https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl*
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl && sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6

wget https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

kubectl version --client
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:
"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

**We'll now initalize Root CA and subsequently generate:**

1. Certificate for user 'admin'
2. Certificates for kubelet in worker nodes
3. Certificate for kube-controller-manager
4. Certificate for kube-proxy 
5. Certificate for kube-scheduler 
6. Certificate for api server (which in this case are controller-0 and controller-1)
7. Certificate for service-account (essentially key pair)

```bash
cat > ca-config.json <<EOF
  {
    "signing": {
      "default": {
        "expiry": "26280h"
      },
      "profiles": {
        "kubernetes": {
          "usages": ["signing", "key encipherment", "server auth", "client auth"],
          "expiry": "8760h"
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
         "OU": "pilot"
       }
     ]
   }
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca


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
        "OU": "pilot"
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

for instance in worker-0 worker-1; do
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
          "OU": "pilot"
        }
      ]
    }
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} --zone us-east1-c \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} --zone us-east1-c \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done

cat > kube-controller-manager-csr.json <<EOF
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
      "OU": "pilot"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager


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
        "OU": "pilot"
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

cat > kube-scheduler-csr.json <<EOF
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
      "OU": "pilot"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

gcloud config set compute/region us-east1
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe pilot \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local


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
        "OU": "pilot"
      }
    ]
  }
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.241.0.10,10.241.0.11,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

cat > service-account-csr.json <<EOF
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
      "OU": "pilot"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

**Let's distribute the certificates and keys generated above to respective hosts**

```bash
for instance in worker-0 worker-1; do
 gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done

gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem controller-1:~/
```

**We'll need config files (aka kubeconfigs) for:**

1. kubelet
2. kube-proxy
3. kube-controller-manager
4. kube-scheduler
5. User 'admin'

```bash
for instance in worker-0 worker-1; do
  kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=pilot \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done


kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=pilot \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=pilot \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=pilot \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

kubectl config set-context default \
    --cluster=pilot \
    --user=admin \
    --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

**Distribute the above configs to respective hosts**
```bash
for instance in worker-0 worker-1; do
 gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
 done

gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig controller-1:~/
```

**Before bootstraping our etcd cluster; generate an encryption key, create encryption config file and ensure all controllers have this file**
```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-config.yaml <<EOF
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

gcloud compute scp encryption-config.yaml controller-1:~/
```

**The following must be run on all controllers. NOTE: Fault tolerance can be achieved only by cluster size of 3 or more**
```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz
tar -xvf etcd-v3.4.0-linux-amd64.tar.gz
sudo mv etcd-v3.4.0-linux-amd64/etcd* /usr/local/bin/
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/

INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)

ETCD_NAME=$(hostname -s)
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.241.0.10:2380,controller-1=https://10.241.0.11:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd

verify:

sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

**We'll now bootstrap the kubernetes control plane**
```bash
sudo mkdir -p /etc/kubernetes/config
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl"

chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/

sudo mkdir -p /var/lib/kubernetes/

sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.241.0.10:2379,https://10.241.0.11:2379 \\
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

sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.151.0.0/16 \\
  --allocate-node-cidrs=true \\
  --cluster-name=pilot \\
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

sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
sudo mkdir -p /etc/kubernetes/config/

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
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

sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
for service in kube-apiserver kube-controller-manager kube-scheduler; do sudo systemctl status $service; done
```

**Install and configure nginx on all controllers to handle HTTP health checks**
```bash
sudo apt-get install -y nginx
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF

sudo mv kubernetes.default.svc.cluster.local /etc/nginx/sites-available/kubernetes.default.svc.cluster.local
sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/

sudo systemctl restart nginx
sudo systemctl enable nginx

Verify:

kubectl get componentstatuses --kubeconfig admin.kubeconfig
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

**Configure RBAC for kubelet authorization on any one controller**
```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
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

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
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

**Provision front end load balancer**
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe pilot \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

gcloud compute firewall-rules create pilot-allow-health-check \
    --network pilot \
    --source-ranges 0.0.0.0/0 \
    --allow tcp

gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

gcloud compute target-pools add-instances kubernetes-target-pool \
   --instances controller-0,controller-1

gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${KUBERNETES_PUBLIC_ADDRESS} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool

Verify:

curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
{
  "major": "1",
  "minor": "15",
  "gitVersion": "v1.15.3",
  "gitCommit": "2d3c76f9091b6bec110a5e63777c332469e0cba2",
  "gitTreeState": "clean",
  "buildDate": "2019-08-19T11:05:50Z",
  "goVersion": "go1.12.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

###On all workers:

**Disable swap, enable ip forwarding, disable firewall and install socat, conntrack on all worker nodes**
```bash
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
sudo -i
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
swapoff -a ; edit /etc/fstab as well
ufw disable ; if running
```

**Install docker on all worker nodes**
```bash
lsb_release -cs
sudo apt-get -y update
sudo apt-get -y install \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
sudo apt-get -y update
apt-cache madison docker-ce
sudo apt-get install -y docker-ce=18.06.3~ce~3-0~ubuntu
sudo usermod -aG docker $USER
sudo systemctl enable docker

>>> logout, log back in and verify docker installation
docker run hello-world
```

**Install kubernetes binaries**
```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet

chmod +x kube-proxy kubelet
sudo  mv kube-proxy kubelet /usr/local/bin/
sudo mkdir -p /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
sudo cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo cp ca.pem /var/lib/kubernetes/
sudo cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

**Configure kubelet and kube-proxy services**
```bash
cat << EOF | sudo tee /etc/systemd/system/kubelet.service
  [Unit]
  Description=Kubernetes Kubelet
  Documentation=https://github.com/kubernetes/kubernetes
  After=docker.service
  Requires=docker.service
  
  [Service]
  ExecStart=/usr/local/bin/kubelet \\
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
    --pod-cidr=10.151.0.0/16 \\
    --register-node=true \\
    --runtime-request-timeout=15m \\
    --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.pem \\
    --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
EOF

cat << EOF | sudo tee /etc/systemd/system/kube-proxy.service
  [Unit]
  Description=Kubernetes Kube Proxy
  Documentation=https://github.com/kubernetes/kubernetes
  
  [Service]
  ExecStart=/usr/local/bin/kube-proxy \\
    --cluster-cidr=10.151.0.0/16 \\
    --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
    --proxy-mode=iptables \\
    --v=2
  Restart=on-failure
  RestartSec=5
  
  [Install]
  WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable kubelet kube-proxy
sudo systemctl start kubelet kube-proxy
for service in kubelet kube-proxy; do sudo systemctl status $service; done
```

**Generate kubeconfig file for authenticating user 'admin' for remote access**
```bash
$ KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe pilot \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

$ kubectl config set-cluster pilot \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

$ kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

$ kubectl config set-context pilot \
    --cluster=pilot \
    --user=admin

$ kubectl config use-context pilot
```

```bash
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

**Configure networking with Calico**
```bash
curl -O \
https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

source <(kubectl completion bash)

POD_CIDR="10.151.0.0/16"
sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml
kubectl apply -f calico.yaml

Verify:

kubectl get nodes -o wide
NAME       STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
worker-0   Ready    <none>   45h   v1.15.3   10.241.0.20   <none>        Ubuntu 18.04.4 LTS   5.0.0-1033-gcp   docker://18.6.3
worker-1   Ready    <none>   45h   v1.15.3   10.241.0.21   <none>        Ubuntu 18.04.4 LTS   5.0.0-1033-gcp   docker://18.6.3
```

**Had to put these routes on workers as well as subsequent gcp routes to get inter-pod comms working**
```bash
sudo route add -net 10.151.0.0/24 gw 10.241.0.1 dev ens4
sudo route add -net 10.151.1.0/24 gw 10.241.0.1 dev ens4

gcloud compute routes create kubernetes-route-10-151-0-0-24 --network pilot --next-hop-address 10.241.0.20 --destination-range 10.151.0.0/24
gcloud compute routes create kubernetes-route-10-151-1-0-24 --network pilot --next-hop-address 10.241.0.21 --destination-range 10.151.1.0/24
gcloud compute routes list --filter "network: pilot"
```

**Configure DNS**
```bash
curl -O https://raw.githubusercontent.com/mch1307/k8s-thw/master/coredns.yaml

>>> edit the following sections in coredns.yaml
   
Corefile: 
    .:53 {
        errors
        health
        kubernetes cluster.local 10.151.0.0/16 10.32.0.0/24 { 
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }

spec:
  selector:
    k8s-app: coredns
  clusterIP: 10.32.0.10 


kubectl apply -f coredns.yaml

Verify:

kubectl get pod -n kube-system -o wide
```

**Verify DNS**
```bash
kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

  dnstools# host kubernetes
  kubernetes.default.svc.cluster.local has address 10.32.0.1
  dnstools# host kube-dns.kube-system
  kube-dns.kube-system.svc.cluster.local has address 10.32.0.10
  dnstools# host 10.32.0.10
  10.0.32.10.in-addr.arpa domain name pointer kube-dns.kube-system.svc.cluster.local.
  dnstools# exit
```

**Setup nginx ingress**

I used this ingress: https://kubernetes.github.io/ingress-nginx/deploy/
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
kubectl set image deployment/nginx-ingress-controller -n \
    ingress-nginx nginx-ingress-controller=quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
```

**Create image pull secret**
```bash
kubectl create secret docker-registry mydockercfg \
  --docker-server "https://gcr.io" \
  --docker-username myusername \
  --docker-email myemail \
  --docker-password=mypassword
```

**Usage:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  imagePullSecrets:
  - name: mydockercfg  <===
  ...
  ...
```

**Persistent storage using rook/ceph**

Referenced from: https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md
```bash
git clone https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f common.yaml
kubectl create -f operator.yaml

Verify all pods are running before proceeding:
kubectl -n rook-ceph get pod

kubectl create -f cluster-test.yaml

Verify:
kubectl -n rook-ceph get pod
```

**Verify rook cluster health using toolbox:**

Reference: https://github.com/rook/rook/blob/master/Documentation/ceph-toolbox.md
```bash
kubectl create -f toolbox.yaml
kubectl exec -n rook-ceph -it rook-ceph-tools-5bd44d9bf8-kptq7 /bin/sh
  sh-4.2# ceph status
```

**Create storage class for block storage**

Reference: https://github.com/rook/rook/blob/master/Documentation/ceph-block.md
```bash
kubectl create -f storageclass.yaml
kubectl describe sc
```

[Click here for similar procedure on on-premise baremetal](https://papudatta.github.io/baremetal_k8s/)
