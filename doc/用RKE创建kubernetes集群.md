# 用RKE搭建kubernetes集群

## RKE简介  

众所周知，原生搭建一个kubernetes集群是一个难度较大，门槛较高，周期较长的一件事情。为了降低难度，[Rancher](https://rancher.com/) 公司推出了一系列相关产品，今天介绍的[RKE](https://rancher.com/products/rke/)就是其中的一个比较有特色的产品之一。  
RKE有点像kubeadm，基本上一条命令就可以搭建一个完整的kubernetes集群。它的特点之一是：用容器化的方式运行所有K8S组件(apiserver,scheduler,controller等)，包括ETCD。所以，运行RKE的前置条件就是必须有docker([环境准备](#preqequisit))。另外一个前置条件是安装节点必须能免密登录集群所有节点。  
它的另外一个优点是使用非常简单。比如，集群的所有配置都被集中定义在一个yml文件(默认是cluster.yml)中，这样用户可以非常方便，直观的定义和了解集群的配置信息。相比于那些把配置信息分散到系统各个位置的管理方式，我觉得RKE的配置集中管理方式更人性化。类似的，安装后的集群状态信息也会被集中记录到一个json文件(默认是cluster.rkestate)中，有了这个文件，集群的扩容、缩容、升级甚至灾备都能做到定位精准，有的放矢。又比如，RKE提供的命令行工具本身，使用起来也是超级简单，简单列举如下：    
 - 建立集群: `rke up`
 - 销毁集群: `rke remove`
 - 生成证书CSR：`rke cert`
 - 生成集群配置： `rke config`
 - ...  
虽然，RKE极大的降低了搭建K8S集群的难度，但这并不意味着RKE仅仅是一种“玩具”。它的稳定性和灵活性还是值得信赖的，因此官方也是对它不吝赞美之词：  
  - CNCF(Cloud Native Computing Foundation)认证：CNCF意味着RKE建立的集群和原生安装的kubernetes集群API是兼容的，这就保证了RKE集群和K8S原生集群之间的可移植性；  
  - 一键操作：扩容/缩容/安装/升级/灾备/恢复，基本上一条命令搞定一个个大事件，确实比较简单；  
  - 包容并济：REK可以兼容各种OS，各种K8S平台甚至各种工具等  
  - 7x24小时企业级支持  
  - 安全的、原子性的升级  
下面我们具体介绍一下怎么用RKE一步一步搭建k8s集群。  

## 快速入门  
<span id="preqequisit"></span>
### 环境准备  
1. 准备一台机器(虚拟机或裸金属服务器)，假设:  
- IP: 192.168.1.1  
- OS: CentOS  
2. 创建非root用户(这里假设用户名为rke)，并把它进入到docker用户组中：  
     ```
     #create "docker" group and create a new user named "rke", add it into "docker" group
     groupadd docker
     useradd rke
     usermod -aG docker rke
     passwd rke
     ```
3. 将rke加入sudoer中
4. 配置ssh server端，允许TCP转发: 编辑 /etc/ssh/sshd_config，将AllowTcpForwardin设置为"yes"
5. 设置ssh免密登录：
```
$ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:M3jHo+f+jiy9HxSL6yrPbfowY/LeSx7hj71hgq0zGq8 root@SGDLITVM0351.hpeswlab.net
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|            .    |
|       . . . o   |
|      . S * o    |
|       . O =     |
|      o B.O +    |
|      .B+#oO o   |
|      E*XX#*B.   |
+----[SHA256]-----+

$ssh-copy-id -i ~/.ssh/id_rsa.pub rke@192.168.1.1 
```
6. 安装docker，两种方式，建议使用第二章方式，因为这些版本的docker是rke验证通过的：
  - docker的官方教程，[下载docker](https://docs.docker.com/engine/install/centos/)  
  - RKE提供的脚本，比如：
    - 18.09.2 : curl https://releases.rancher.com/install-docker/18.09.2.sh | sh
    - 18.06.2 : curl https://releases.rancher.com/install-docker/18.06.2.sh | sh
    - 17.03.2 : curl https://releases.rancher.com/install-docker/17.03.2.sh | sh  
7. 关闭防火墙:  
`systemctl stop firewalld`
8. 手动编辑cluster.yml文件  
```
nodes:
    - address: 192.168.1.1  
      user: rke
      role:
        - controlplane
        - etcd
        - worker
```
9. 创建集群：  
```
rke up

INFO[0000] Building Kubernetes cluster
INFO[0000] [dialer] Setup tunnel for host [10.0.0.1]
INFO[0000] [network] Deploying port listener containers
INFO[0000] [network] Pulling image [alpine:latest] on host [10.0.0.1]
...
INFO[0101] Finished building Kubernetes cluster successfully
```
一口气做下来，如果看到了"Finished building Kubernetes cluster successfully"，那就说明集群创建成功了。这时，可以看到，在cluster.yml同级目录下，rke顺带生成了2个文件:  
- kube_config_cluster.yml  
- cluster.rkestate  
其中，cluster.rkestate是集群状态信息文件，集群的后续操作都会从这个文件中提取必要信息，这里暂且不表。而kube_config_cluster.yml是rke为kubectl生成的配置文件，为了后续操作方便，建议将kube_config_cluster.yml覆盖kubectl的默认配置文件~/.kube/config： 
```
cp kube_config_cluster.yml ~/.kube/config
```
有了配置文件后，就可以通过kubectl获得集群的各种信息了:  
```
$ kubectl get nodes
192.168.1.1    Ready    controlplane,etcd,worker   97m   v1.19.5
$ kubectl get pod -A
$ kubectl get pod -A
NAMESPACE       NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx   default-http-backend-65dd5949d9-wn5pm      1/1     Running     0          84m
ingress-nginx   nginx-ingress-controller-4lx7f             1/1     Running     0          89m
kube-system     calico-kube-controllers-7fbff695b4-bdtcp   1/1     Running     0          97m
kube-system     canal-4g7mn                                2/2     Running     0          89m
kube-system     coredns-6f85d5fb88-dxrbm                   1/1     Running     0          97m
kube-system     metrics-server-8449844bf-fmxjx             1/1     Running     0          97m
kube-system     rke-coredns-addon-deploy-job-bx7h5         0/1     Completed   0          97m
kube-system     rke-ingress-controller-deploy-job-4xf7m    0/1     Completed   0          97m
kube-system     rke-metrics-addon-deploy-job-xz8wq         0/1     Completed   0          97m
kube-system     rke-network-plugin-deploy-job-rxw9s        0/1     Completed   0          97m
```
之前我们说过rke的特色之一就是用容器化的方式运行所有K8S组件，上面命名输出也印证了这一点： K8S的组件没有被纳入到K8S集群中统一管理。而这一点我个人认为美中不足的地方。我们跑一下"docker ps" 可以看到K8S组件和etcd都是由docker管理:  
```
$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS               NAMES
ed0ed05e8aa0        rancher/rke-tools:v0.1.67              "/docker-entrypoint.…"   2 hours ago         Up 2 hours                              etcd-rolling-snapshots
d019fc51dbd2        1f0ca6d99110                           "/usr/bin/dumb-init …"   2 hours ago         Up 2 hours                              k8s_nginx-ingress-controller_nginx-ingress-controller-r578w_ingress-nginx_de784445-7e62-47bf-85b2-ccd559d93591_0
892fb5245147        rancher/pause:3.2                      "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_nginx-ingress-controller-r578w_ingress-nginx_de784445-7e62-47bf-85b2-ccd559d93591_0
c84d2b1fa5cd        rancher/hyperkube:v1.19.5-rancher1     "/opt/rke-tools/entr…"   2 hours ago         Up 2 hours                              kubelet
.......
```
万事大吉，到此为止，我们的入门教程也差不多该告一段落了。结束之前，让我们来稍微回顾总结一下：  
1. rke给node定义了3种角色(role)：  
   - controlplane: 用于安装K8S组件  
   - etcd: 用于安装etcd  
   - worker: 负载节点，负责运行除K8S和etcd之外的所有组件  
  理论上，每个节点都可以是这3个角色的任意组合，但通常，我们会把集群分为master节点和worker节点以便于管理。master节点通常是controlplane和etcd的组合，而worker节点的角色就是worker。
2. 虽然这里选择CentOS作为节点的底层操作系统，但事实上RKE可以支持很多Linux操作系统，比如，它的很多测试都是在Ubuntu上进行的。  
3. 入门教程图省事把firewalld直接停了，这种粗暴的做法会带来很大的安全隐患，实际生产环境上一般会用iptalbe/firewall对端口流量进行精细化的控制，关于端口设定请参考[端口配置]()  
4. 创建非root用户不是rke的要求，而是SSH的要求(Red Hat Enterprise Linux/Oracle Linux/CentOS 用户不可以使用root作为ssh 用户，见[Bugzilla 1527565](https://bugzilla.redhat.com/show_bug.cgi?id=1527565))，但ssh是rke的要求。  
5. 集群配置可以简单到只配置一个节点信息，也可以复制到让你感到头昏眼花，更多集群配置信息参考:[集群配置](#cluster_config)。  
6. 如果通过"--config"显式指定，集群配置文件可以不叫"cluster.yml"  
通过本节读者应该已经大致能理解rke的用途和工作原理了，但到目前为止我们也就只能创建了一个最最简陋的一个集群，这离真正生产可用的集群差得远了，如果想了解rke更多的用法，创建更复杂的集群，请阅读[RKE进阶](#rke_advance)和[RKE高级教程]()  

<span id="rke_advance"></span>
## RKE进阶

### 集群伸缩  
#### 扩容/缩容  
增加和删除节点对rke来说都是小菜一碟:  
- 增加节点: 编辑集群配置文件(cluster.yml)，在node段中新增节点信息，然后运行`rke up`  
- 删除节点: 编辑集群配置文件(cluster.yml)，在node段中删除节点信息，然后运行`rke up`  
- 节点角色变更: 编辑集群配置文件(cluster.yml)，在node段中修改节点role信息，然后运行`rke up`  
就这么简单：编辑cluster.yml文件，运行`rke up`。一招鲜，吃遍天。  
另外，rke为增加/删除worker节点，单独定义了一个参数："--update-only"。使用这个参数后，cluster.yml中和worker节点不相干的配置都会被忽略。  

#### 销毁集群  
运行"rke remove"来销毁整个集群，注意，一旦运行该命令，集群将被彻底销毁，并且etcd在本地和AWS S3(如果有的话)上的数据备份都将被销毁。"rke remove" 将删除cluster.yml中定义的所有节点的以下组件：
- etcd  
- kube-apiserver  
- kube-controller-manager  
- kubelet  
- kube-proxy  
- nginx-proxy  
同时它还将清除节点的以下目录:
- /etc/kubernetes/ssl  
- /var/lib/etcd  
- /etc/cni  
- /opt/cni  
- /var/run/calico  

**特别注意：**  
如果希望节点能重用，需要清理目录/var/kubelet/ 下的历史残留文件，否则有可能导致以下莫名其妙的问题。而该目录下的文件有可能又被残留的容器(正在运行或已经退出的)应用，所以正确的清除/var/kubelet的步骤为：  
1. 删除所有容器  
2. 重启机器  
3. 删除/var/kubelet/下所有文件和目录  

### 证书管理  
#### 证书更新
Kubernetes要求组件之间通过证书加密通讯。默认情况下，rke自动生成所有组件需要的证书。但是，为了防止证书过期，或者证书泄露，我们还是有必要在适当的时候更新证书。证书更新后，集群的相关组件将自动重启，这些组件包括：  
- etcd  
- kubelet  
- kube-apiserver  
- kube-proxy  
- kube-scheduler  
- kube-controller-manager  
另外，rke提供以下3种证书更新方式：  
- 在原CA基础上更新所有K8S组件证书  
- 在原CA基础上更新特定K8S组件证书  
- 更新CA及所有K8S组件证书 
注意：更新证书需要从cluster.yml中提取集群信息，所以，如果有必要记得使用"--config"选项。  
##### 在原CA基础上更新所有K8S组件证书  
`rke cert rotate`命令将在保留原有CA的基础上更新所有相关组件的证书，更新完成后，组件自动重启：  
```
$ rke cert rotate --config cluster-1x1.yml
WARN[0000] This is not an officially supported version (v1.2.4-rc6) of RKE. Please download the latest official release at https://github.com/rancher/rke/releases
INFO[0000] Running RKE version: v1.2.4-rc6
INFO[0000] Initiating Kubernetes cluster
INFO[0000] Rotating Kubernetes cluster certificates
INFO[0000] [certificates] GenerateServingCertificate is disabled, checking if there are unused kubelet certificates
INFO[0000] [certificates] Generating Kubernetes API server certificates
INFO[0000] [certificates] Generating Kube Controller certificates
INFO[0001] [certificates] Generating Kube Scheduler certificates
INFO[0002] [certificates] Generating Kube Proxy certificates
INFO[0002] [certificates] Generating Node certificate
INFO[0002] [certificates] Generating admin certificates and kubeconfig
INFO[0003] [certificates] Generating Kubernetes API server proxy client certificates
INFO[0003] [certificates] Generating kube-etcd-16-187-190-95 certificate and key
INFO[0004] Successfully Deployed state file at [./cluster-1x1.rkestate]
INFO[0004] Rebuilding Kubernetes cluster with rotated certificates
INFO[0004] [dialer] Setup tunnel for host [192.168.1.1]
......
INFO[0021] [worker] Successfully restarted Worker Plane..
```
##### 在原CA基础上更新特定K8S组件证书  
`--service`选项可以指定待更新的组件。证书更新完成后，该组件将自动重启，比如下面代码将指定更新kubelet证书：  
```
$ rke cert rotate --service kubelet --config cluster-1x1.yml
WARN[0000] This is not an officially supported version (v1.2.4-rc6) of RKE. Please download the latest official release at https://github.com/rancher/rke/releases
INFO[0000] Running RKE version: v1.2.4-rc6
INFO[0000] Initiating Kubernetes cluster
INFO[0000] Rotating Kubernetes cluster certificates
INFO[0000] [certificates] Generating Node certificate
INFO[0000] Successfully Deployed state file at [./cluster-1x1.rkestate]
INFO[0000] Rebuilding Kubernetes cluster with rotated certificates
INFO[0000] [dialer] Setup tunnel for host [192.168.1.1]
......
INFO[0014] [restart/kube-proxy] Successfully restarted container on host [192.168.1.1]
INFO[0014] [worker] Successfully restarted Worker Plane..
```
##### 更新CA及所有K8S组件证书  
`--rotate-ca`选项强制更新CA证书。CA证书更新后，所有用到该CA的系统组件(包括K8S组件、网络组件等)的证书都将失效，这些证书也会被逐一更新，更新之后，组件将自动重启：  
```
$ rke cert rotate --rotate-ca --config cluster-1x1.yml
WARN[0000] This is not an officially supported version (v1.2.4-rc6) of RKE. Please download the latest official release at https://github.com/rancher/rke/releases
INFO[0000] Running RKE version: v1.2.4-rc6
INFO[0000] Initiating Kubernetes cluster
INFO[0000] Rotating Kubernetes cluster certificates
INFO[0000] [certificates] Generating CA kubernetes certificates
INFO[0000] [certificates] Generating Kubernetes API server aggregation layer requestheader client CA certificates
INFO[0001] [certificates] GenerateServingCertificate is disabled, checking if there are unused kubelet certificates
INFO[0001] [certificates] Generating Kubernetes API server certificates
INFO[0001] [certificates] Generating Kube Controller certificates
INFO[0002] [certificates] Generating Kube Scheduler certificates
INFO[0002] [certificates] Generating Kube Proxy certificates
INFO[0002] [certificates] Generating Node certificate
INFO[0003] [certificates] Generating admin certificates and kubeconfig
INFO[0003] [certificates] Generating Kubernetes API server proxy client certificates
INFO[0004] [certificates] Generating kube-etcd-16-187-190-95 certificate and key
INFO[0004] Successfully Deployed state file at [./cluster-1x1.rkestate]
INFO[0004] Rebuilding Kubernetes cluster with rotated certificates
INFO[0004] [dialer] Setup tunnel for host [192.168.1.1]
......
INFO[0052] Restarting container [kube-proxy] on host [192.168.1.1], try #1
INFO[0052] [restart/kube-proxy] Successfully restarted container on host [192.168.1.1]
INFO[0052] [worker] Successfully restarted Worker Plane..
INFO[0052] Restarting network, ingress, and metrics pods
```

### 自定义证书安装  
#### 生成CSR  
rke非常贴心的提供了一键生成CSR的命令：`rke cert generate-csr`。 用真正的CA去给这些CSR文件签名后，就可以得到自定义的证书了。默认CSR文件生成在./cluster_certs目录下，安装的时候，可以通过--cert-dir指定自定义证书目录。  
```
$ rke cert generate-csr     
INFO[0000] Generating Kubernetes cluster CSR certificates
INFO[0000] [certificates] Generating Kubernetes API server csr
INFO[0000] [certificates] Generating Kube Controller csr
INFO[0000] [certificates] Generating Kube Scheduler csr
INFO[0000] [certificates] Generating Kube Proxy csr     
INFO[0001] [certificates] Generating Node csr and key   
INFO[0001] [certificates] Generating admin csr and kubeconfig
INFO[0001] [certificates] Generating Kubernetes API server proxy client csr
INFO[0001] [certificates] Generating etcd-x.x.x.x csr and key
INFO[0001] Successfully Deployed certificates at [./cluster_certs]
```
除了生成CSR，还要生成kube-service-account-token-key.pem，可以通过以下命令得到：  
`$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./cluster_certs/kube-service-account-token-key.pem -out ./cluster_certs/kube-service-account-token.pem`

#### 证书签名  
1. 如果有现成的CA证书那么可以跳过这一步。如果没有，我们可以生成一个自签名CA证书：  
```
$cd cluster-certs
$cn="CA on RKE-cluster"
$openssl genrsa -out kube-ca.key 2048
$openssl req -x509 -new -nodes -key ca.key -subj "/CN=${cn}" -days 3660 -out kube-ca.pem
```
2. 生成一个extfile文件，文件包含证书的SAN信息：  
`echo "subjectAltName=IP:192.168.1.1,IP:127.0.0.1,IP:10.43.0.1" > extfile.cnf`
注意：这里的SAN应该包含安装节点的IP,安装节点的loopback IP，集群DNS的IP
3. 签名生成证书:  
```
names=("kube-admin" "kube-apiserver" "kube-apiserver-proxy-client" "kube-controller-manager" "kube-etcd-192-168-1-1" "kube-node" "kube-proxy" "kube-scheduler")
for name in ${names[@]}; do
openssl x509 -req -sha256 -in ${name}-csr.pem -CA kube-ca.pem -CAkey ca.key -CAcreateserial  -extfile extfile.cnf -out ${name}.pem -days 365
```
#### 自定义证书安装：  
`rke up --config cluster.yml --custom-certs --cert-dir ./cluster-certs`

## RKE高级

 








## 集群配置
集群配置最小集:
```
nodes:
    - address: 1.2.3.4
      user: ubuntu
      role:
        - controlplane
        - etcd
        - worker
```
集群配置全集：
```
nodes:
    - address: 1.1.1.1
      user: ubuntu
      role:
        - controlplane
        - etcd
      ssh_key_path: /home/user/.ssh/id_rsa
      port: 2222
    - address: 2.2.2.2
      user: ubuntu
      role:
        - worker
      ssh_key: |-
        -----BEGIN RSA PRIVATE KEY-----

        -----END RSA PRIVATE KEY-----
    - address: example.com
      user: ubuntu
      role:
        - worker
      hostname_override: node3
      internal_address: 192.168.1.6
      labels:
        app: ingress

# If set to true, RKE will not fail when unsupported Docker version
# are found
ignore_docker_version: false

# Cluster level SSH private key
# Used if no ssh information is set for the node
ssh_key_path: ~/.ssh/test

# Enable use of SSH agent to use SSH private keys with passphrase
# This requires the environment `SSH_AUTH_SOCK` configured pointing
#to your SSH agent which has the private key added
ssh_agent_auth: true

# List of registry credentials
# If you are using a Docker Hub registry, you can omit the `url`
# or set it to `docker.io`
# is_default set to `true` will override the system default
# registry set in the global settings
private_registries:
     - url: registry.com
       user: Username
       password: password
       is_default: true

# Bastion/Jump host configuration
bastion_host:
    address: x.x.x.x
    user: ubuntu
    port: 22
    ssh_key_path: /home/user/.ssh/bastion_rsa
# or
#   ssh_key: |-
#     -----BEGIN RSA PRIVATE KEY-----
#
#     -----END RSA PRIVATE KEY-----

# Set the name of the Kubernetes cluster  
cluster_name: mycluster


# The Kubernetes version used. The default versions of Kubernetes
# are tied to specific versions of the system images.
#
# For RKE v0.2.x and below, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/types/blob/release/v2.2/apis/management.cattle.io/v3/k8s_defaults.go
#
# For RKE v0.3.0 and above, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/kontainer-driver-metadata/blob/master/rke/k8s_rke_system_images.go
#
# In case the kubernetes_version and kubernetes image in
# system_images are defined, the system_images configuration
# will take precedence over kubernetes_version.
kubernetes_version: v1.10.3-rancher2

# System Images are defaulted to a tag that is mapped to a specific
# Kubernetes Version and not required in a cluster.yml. 
# Each individual system image can be specified if you want to use a different tag.
#
# For RKE v0.2.x and below, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/types/blob/release/v2.2/apis/management.cattle.io/v3/k8s_defaults.go
#
# For RKE v0.3.0 and above, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/kontainer-driver-metadata/blob/master/rke/k8s_rke_system_images.go
#
system_images:
    kubernetes: rancher/hyperkube:v1.10.3-rancher2
    etcd: rancher/coreos-etcd:v3.1.12
    alpine: rancher/rke-tools:v0.1.9
    nginx_proxy: rancher/rke-tools:v0.1.9
    cert_downloader: rancher/rke-tools:v0.1.9
    kubernetes_services_sidecar: rancher/rke-tools:v0.1.9
    kubedns: rancher/k8s-dns-kube-dns-amd64:1.14.8
    dnsmasq: rancher/k8s-dns-dnsmasq-nanny-amd64:1.14.8
    kubedns_sidecar: rancher/k8s-dns-sidecar-amd64:1.14.8
    kubedns_autoscaler: rancher/cluster-proportional-autoscaler-amd64:1.0.0
    pod_infra_container: rancher/pause-amd64:3.1

services:
    etcd:
      # if external etcd is used
      # path: /etcdcluster
      # external_urls:
      #   - https://etcd-example.com:2379
      # ca_cert: |-
      #   -----BEGIN CERTIFICATE-----
      #   xxxxxxxxxx
      #   -----END CERTIFICATE-----
      # cert: |-
      #   -----BEGIN CERTIFICATE-----
      #   xxxxxxxxxx
      #   -----END CERTIFICATE-----
      # key: |-
      #   -----BEGIN PRIVATE KEY-----
      #   xxxxxxxxxx
      #   -----END PRIVATE KEY-----
    # Note for Rancher v2.0.5 and v2.0.6 users: If you are configuring
    # Cluster Options using a Config File when creating Rancher Launched
    # Kubernetes, the names of services should contain underscores
    # only: `kube_api`.
    kube-api:
      # IP range for any services created on Kubernetes
      # This must match the service_cluster_ip_range in kube-controller
      service_cluster_ip_range: 10.43.0.0/16
      # Expose a different port range for NodePort services
      service_node_port_range: 30000-32767    
      pod_security_policy: false
      # Add additional arguments to the kubernetes API server
      # This WILL OVERRIDE any existing defaults
      extra_args:
        # Enable audit log to stdout
        audit-log-path: "-"
        # Increase number of delete workers
        delete-collection-workers: 3
        # Set the level of log output to debug-level
        v: 4
    # Note for Rancher 2 users: If you are configuring Cluster Options
    # using a Config File when creating Rancher Launched Kubernetes,
    # the names of services should contain underscores only:
    # `kube_controller`. This only applies to Rancher v2.0.5 and v2.0.6.
    kube-controller:
      # CIDR pool used to assign IP addresses to pods in the cluster
      cluster_cidr: 10.42.0.0/16
      # IP range for any services created on Kubernetes
      # This must match the service_cluster_ip_range in kube-api
      service_cluster_ip_range: 10.43.0.0/16
    kubelet:
      # Base domain for the cluster
      cluster_domain: cluster.local
      # IP address for the DNS service endpoint
      cluster_dns_server: 10.43.0.10
      # Fail if swap is on
      fail_swap_on: false
      # Set max pods to 250 instead of default 110
      extra_args:
        max-pods: 250
      # Optionally define additional volume binds to a service
      extra_binds:
        - "/usr/libexec/kubernetes/kubelet-plugins:/usr/libexec/kubernetes/kubelet-plugins"

# Currently, only authentication strategy supported is x509.
# You can optionally create additional SANs (hostnames or IPs) to
# add to the API server PKI certificate.
# This is useful if you want to use a load balancer for the
# control plane servers.
authentication:
    strategy: x509
    sans:
      - "10.18.160.10"
      - "my-loadbalancer-1234567890.us-west-2.elb.amazonaws.com"

# Kubernetes Authorization mode
# Use `mode: rbac` to enable RBAC
# Use `mode: none` to disable authorization
authorization:
    mode: rbac

# If you want to set a Kubernetes cloud provider, you specify
# the name and configuration
cloud_provider:
    name: aws

# Add-ons are deployed using kubernetes jobs. RKE will give
# up on trying to get the job status after this timeout in seconds..
addon_job_timeout: 30

# Specify network plugin-in (canal, calico, flannel, weave, or none)
network:
    plugin: canal

# Specify DNS provider (coredns or kube-dns)
dns:
    provider: coredns

# Currently only nginx ingress provider is supported.
# To disable ingress controller, set `provider: none`
# `node_selector` controls ingress placement and is optional
ingress:
    provider: nginx
    node_selector:
      app: ingress
      
# All add-on manifests MUST specify a namespace
addons: |-
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-nginx
      namespace: default
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80

addons_include:
    - https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/rook-operator.yaml
    - https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/rook-cluster.yaml
    - /path/to/manifest
```
可以看到，集群的配置项还是很多的，以下挑一些常用的解释一下：  
TO DO:  



## 常见问题
1. rke ssh连接不到其他节点，这种情况一般是ssh免密登录没配置好，请参考[ssh免密登录](#ssh_config)
2. rke 从 dockhub下载image失败，这种情况，你可能需要配置[docker proxy](https://docs.docker.com/config/daemon/systemd/#http-proxy)
3. 启动多节点集群，发现某一个(多个)节点启动失败，RKE终端日志如下：  
WARN[0018] Can't start Docker container [rke-worker-port-listener] on host [16.187.191.114]: Error response from daemon: driver failed programming external connectivity on endpoint rke-worker-port-listener (b3cfec0c0e0143ffbf8fcbfd4ebd6d5b68458215584479d94cfb7d7f7df8dab2):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 10250 -j DNAT --to-destination 172.17.0.2:1337 ! -i docker0: iptables: No chain/target/match by that name.
 (exit status 1))  
 原因: 登录到失败节点，运行"docker ps -a", 会发现有些container 状态为 Created，如下所示：  
 ```
 #docker ps -a
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS                     PORTS               NAMES
9cbb50f4866d        rancher/rke-tools:v0.1.67   "/docker-entrypoint.…"   2 minutes ago       Created                                        rke-worker-port-listener
c5017a694b54        rancher/rke-tools:v0.1.67   "/docker-entrypoint.…"   3 minutes ago       Exited (0) 2 minutes ago                       cluster-state-deployer
 ```
说明container被创建了，但是没来得及运行，这种问题一般是docker出现了异常，重启docker就可以了：  
`sudo systemctl resart docker`

4. 启动集群的时候，某个节点kubelet起不来，RKE终端日志如下：  
ERRO[0110] Failed to upgrade hosts: 16.187.191.58 with error [Failed to verify healthcheck: Failed to check http://localhost:10248/healthz for service [kubelet] on host [16.187.191.58]: Get "http://localhost:10248/healthz": Unable to access the service on localhost:10248. The service might be still starting up. Error: ssh: rejected: connect failed (Connection refused), log: + umount /var/lib/docker/devicemapper/mnt/68ff7c2f1246d1b0cce5ceb85696eab7b00f768a54c4f6e9696c2a5c1c6b65fb/rootfs/var/lib/kubelet/pods/09bf295d-0517-43b7-af04-8a578cdcc657/volume-subpaths/itsma-bmant-smartanalytics-volume/smarta-sawmeta-dih/0]  
FATA[0110] [workerPlane] Failed to upgrade Worker Plane: [Failed to verify healthcheck: Failed to check http://localhost:10248/healthz for service [kubelet] on host [16.187.191.58]: Get "http://localhost:10248/healthz": Unable to access the service on localhost:10248. The service might be still starting up. Error: ssh: rejected: connect failed (Connection refused), log: + umount /var/lib/docker/devicemapper/mnt/68ff7c2f1246d1b0cce5ceb85696eab7b00f768a54c4f6e9696c2a5c1c6b65fb/rootfs/var/lib/kubelet/pods/09bf295d-0517-43b7-af04-8a578cdcc657/volume-subpaths/itsma-bmant-smartanalytics-volume/smarta-sawmeta-dih/0]  
登录到失败的机器，运行"docker ps | grep kubelet"，如下：  
```
#docker ps | grep kubelet
1f697eb6a724        rancher/hyperkube:v1.19.5-rancher1     "/opt/rke-tools/entr…"   12 minutes ago      Up 12 minutes                           kubelet
```
然后，查看容器日志，运行: "docker logs -f 1f697eb6a724"  
日志很长一串，其中错误日志中可能有误导性，比如，kubelet报cni 网络插件找不到之类的(TO DO： add the log)  
但其实原因可能是因为docker运行时还运行着很多container(包括已经退出的container)，以及/var/lib/kubelet目录下被container引用的文件引起的冲突。解决的办法就是把这些所有container清干净，然后重启系统。


