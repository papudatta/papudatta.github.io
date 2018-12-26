## Introduction

Hello! My goal is to outline the detailed steps i followed in bringing up a kubernetes cluster on "bare-metal" Ubuntu VMs. All this was inspired by the exellent tutorials in [kubernetes_the_hard_way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and [Michael Champagne's blog](https://blog.csnet.me/)

## Summary of contents

* Details of the environment i used, including version of various components
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

## Environment

**Machine details**

| Hostname | IP Address | Memory | Disk space |
| --- | --- | --- | --- |
| **master.home** | 192.168.1.111 | 4GiB | 15GiB |
| **node1.home** | 192.168.1.112 | 3GiB | 20GiB |
| **node2.home** | 192.168.1.113 | 3GiB | 20GiB |

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


### Markdown1

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes1

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/papudatta/papudatta.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact1

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
https://github.com/kelseyhightower/kubernetes-the-hard-way