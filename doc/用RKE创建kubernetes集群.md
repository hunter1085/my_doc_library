# 用RKE搭建kubernetes集群

## 简介  
RKE
众所周知，原生搭建一个kubernetes集群是一个难度较大，门槛较高，周期较长的一件事情。为了降低难度，[Rancher](https://rancher.com/) 公司推出了一系列相关产品，今天介绍的[RKE](https://rancher.com/products/rke/)就是其中的一个比较有特色的产品之一。  
RKE有点像kubeadm，基本上一条命令就可以搭建一个完整的kubernetes集群。它的特点之一是：用容器化的方式运行所有k8s组件(apiserver,scheduler,controller等)，包括ETCD。所以，运行RKE的前置条件就是必须有docker([环境准备](#preqequisit))。另外一个前置条件是按照节点必须能免密登录集群所有节点，所以  
它的另外一个优点是使用非常简单，比如，集群的所有配置都被集中定义在一个yml文件(默认是cluster.yml)中，这样用户可以非常方便，直观的定义和了解集群的配置信息。相比于那些把配置信息分散到系统各个位置的管理方式，我觉得RKE的配置集中管理方式更人性化。类似的，安装后，集群的状态信息也会被集中记录到一个yml文件(默认是cluster.rkestate)中，有了这个文件，集群的扩容、缩容、升级甚至灾备都能做到有的放矢。又比如，RKE提供的命令行工具本身，使用起来也是超级简单，比如：    
 - 建立集群: rke up
 - 销毁集群: rke remove
 - 生成证书CSR：rke cert
 - 生成集群配置： rke config
 - ...
虽然，RKE极大的降低了搭建k8s集群的难度，但这并不意味着RKE仅仅是一种“玩具”。它的稳定性和灵活性还是值得信赖的，因此官方也是对它不吝赞美之词：  
  - CNCF(Cloud Native Computing Foundation)认证：意味着和原生的kubernetes API兼容，这就保证了RKE集群和k8s原生集群之间的可移植性；  
  - 一键安装：安装确实变得很容易；  
  - 包容并济：REK可以兼容各种OS，各种k8s平台甚至各种工具等  
  - 7x24小时企业级支持  
  - 安全的、原子性的升级  
下面我们具体介绍一下怎么用RKE一步一步搭建k8s集群。  

<span id="preqequisit"></span>
## 环境准备  
<span id="cluster_design"></span>
### 集群规划  
- Node-1:  
  - IP: 16.187.190.1  
  - OS: CentOS  
  - Role: controlplane,etcd  
  - User Account: rke-user  
- Node-2:  
  - IP: 16.187.190.2  
  - OS: CentOS  
  - Role: worker  
  - User Account: rke-user  
这里只规划了一个比较简单的集群：2个节点，其中一个节点是master节点，一个是work节点。RKE定义了3种Role：  
  * controlplane  
  * etcd  
  * worker  
controleplane这个角色对应的就是master节点，etcd可以放在master节点也可以和master节点隔离，worker就是负载节点，但其实这3个角色可以任意组合。这里为了清晰，也为了方便起见，将controlplane和etcd整合在一起，和worker隔离开。  
另外，需要注意的是，虽然这里选择CentOS作为节点的底层操作系统，但事实上RKE可以支持很多Linux操作系统，比如，它的很多测试都是在Ubuntu上进行的。

言归正传，上面我们规划的集群，可以用下面的yaml配置来描述，并保存为cluster.yml：  
```
nodes:
    - address: 16.187.190.1
      user: rke-user
      role:
        - controlplane
        - etcd
      ssh_key_path: ~/.ssh/id_rsa
    - address: 16.187.190.2
      user: rke-user
      role:
        - worker
      ssh_key_path: ~/.ssh/id_rsa
```
关于集群配置文件，这里多说两句。当我们运行"rke up"来建立集群的时候，rke会在当前工作目录下搜索名为cluster.yml的文件作为集群的配置文件，如果找不到就会报错，当然用户还可以通过"--config" 来指定集群配置文件。另外，用户可以用手动编辑的方式生成集群配置文件，也可以运行RKE config，通过和RKE交互，最终生成集群配置文件。RKE定义了很多集群的配置项，具体可以参考:[集群配置](#cluster_config)  
### 集群节点准备  
  1. 创建非root用户  
     这个并不是RKE的要求，而是SSH的要求: Red Hat Enterprise Linux/Oracle Linux/CentOS 用户不可以使用root作为ssh 用户，见[Bugzilla 1527565](https://bugzilla.redhat.com/show_bug.cgi?id=1527565)。根据[集群规划](#cluster_design)节中的设计，我们需要:  
     - 新建rke-user用户，并把它加入到"docker"用户组：  
     ```
     #create "docker" group and create a new user named "rke-user", add "rke-user" into "docker" group
     groupadd docker
     useradd rke-user
     usermod -aG docker rke
     passwd rke
     ```
     - 将rke-user加入到sudoers中：在/etc/sudoers文件中，增加"rke   ALL=(ALL)       ALL"  
     ```
     cat /etc/sudoers
     ......
     ## Allow root to run any commands anywhere
      root    ALL=(ALL)       ALL
      admin   ALL=(ALL)       ALL
      rke   ALL=(ALL)       ALL
     ......
     ```
     - 用rke-user重新登录Node-1和Node-2  
  2. ssh server端配置:  
    - 打开/etc/ssh/sshd_config, 将AllowTcpForwarding的值设置成 yes  
    ```
    #AllowAgentForwarding yes
    AllowTcpForwarding yes
    #GatewayPorts no
    X11Forwarding yes
    ```
  3. 防火墙配置:  
     - 有很多端口需要配置，简单起见可以直接把firewalld关闭:  
     `sudo systemctl stop firewalld`  
     - 也可以根据下标一个一个设置iptable规则:  
       [端口定义](https://rancher.com/docs/rke/latest/en/os/)  

<span id="ssh_config"></span>
### ssh免密登录设置  
RKE要求能通过ssh免密登录到其他节点，因此ssh免密登录也是前置条件之一。假设Node-1是RKE工作节点，我们需要：  
1. 在每台节点上创建ssh公私钥对:  
`ssh-keygen -t rsa`
2. 把每台节点的公钥复制到Node-1：  
`ssh-copy-id -i ~/.ssh/id_rsa.pub rke-user@16.187.190.1`
### 安装docker  
首先，RKE并不是支持所有的docker/kubernetes版本，具体支持哪些版本，请参考[release note](https://rancher.com/support-maintenance-terms)  
有2种方式安装docker:
- docker的官方教程，[下载docker](https://docs.docker.com/engine/install/centos/)  
- RKE提供的脚本，比如：
  - 18.09.2 : curl https://releases.rancher.com/install-docker/18.09.2.sh | sh
  - 18.06.2 : curl https://releases.rancher.com/install-docker/18.06.2.sh | sh
  - 17.03.2 : curl https://releases.rancher.com/install-docker/17.03.2.sh | sh

## 创建集群
万事俱备只欠东风，我们现在只需要下载[RKE可执行文件](https://github.com/rancher/rke/releases)，将可执行文件重命名为rke，并放到PATH可以搜索到的任何目录下，比如，/usr/bin目录。然后运行：rke up，然后就可以看到一行行日志打印出来: 
```
rke up

INFO[0000] Building Kubernetes cluster
INFO[0000] [dialer] Setup tunnel for host [10.0.0.1]
INFO[0000] [network] Deploying port listener containers
INFO[0000] [network] Pulling image [alpine:latest] on host [10.0.0.1]
...
INFO[0101] Finished building Kubernetes cluster successfully
```
如果你能看到"Finished building Kubernetes cluster successfully"这句日志，那么，恭喜，k8s集群创建成功了。  
常见的错误是：
1. rke ssh连接不到其他节点，这种情况一般是ssh免密登录没配置好，请参考[ssh免密登录](#ssh_config)
2. rke 从 dockhub下载image失败，这种情况，你可能需要配置[docker proxy](https://docs.docker.com/config/daemon/systemd/#http-proxy)
<span id="cluster_config"></span>
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

## 集群伸缩  
### 扩容/缩容
使用rke来增加和删除节点是再简单不过的事了：用户只需要重新编辑集群配置文件(cluster.yml)，如果要扩容，那么就新node段新增节点信息；如果要缩容，那么只要删除节点信息；甚至，集群伸缩的过程中，节点的角色都能修改，比如原来Node-1的角色是controlplane+etcd，扩容后，希望Node-1同时也是worker，那么只要把Node-1增加worker角色；当然，最后还得运行"rke up"，这样整个集群的规模就自动伸缩成和你定义的一样了。  
另外，rke为增加/删除worker节点，单独定义了一个参数——"--update-only"。使用这个参数后，cluster.yml中和worker节点不相干的配置都会被忽略。  

### 销毁集群
运行"rke remove"来销毁整个集群，注意，一旦运行该命令，集群将被彻底销毁，并且etcd在本地和AWS(S3)上的数据备份都将被销毁。"rke remove" 还将删除cluster.yml中定义的所有节点的以下组件：
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

## 更新证书

## 常见问题

1. 启动多节点集群，发现某一个(多个)节点启动失败，RKE终端日志如下：  
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

2. 启动集群的时候，某个节点kubelet起不来，RKE终端日志如下：  
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


