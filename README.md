# k8s

## 分发公钥

```bash
ssh-keygen
for host in master1.test.lab master2.test.lab master3.test.lab lb.test.lab etcd1.test.lab etcd2.test.lab etcd3.test.lab infra-node1.test.lab infra-node2.test.lab node1.test.lab node2.test.lab; do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; done
```

## minikube

* 要翻墙：
  * 直接方式`curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install  minikube-linux-amd64 /usr/local/bin/minikube`
  * rpm方式`curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-1.5.1.rpm && sudo rpm -ivh  minikube-1.5.1.rpm`
  * 直接github release里下载`curl -LO https://github.com/kubernetes/minikube/releases/download/v1.5.2/minikube-linux-amd64  &&  sudo install minikube-linux-amd64 /usr/local/bin/minikube`
  * 码云版本，须要自己手动编译（下载不需翻墙，但运行时下载image须翻墙）
  * 本地环境，先翻墙，下载好:
    * minikube/bin/docker-machine-driver-kvm2
    * minikube/cache/\/kubelet和kubeadm
    * minikube/cache/minikube-v1.5.1.iso，这个须要放在--iso-url位置
    * 自建registry，然后把镜像都放在自建registry
    * `minikube start --image-repository nexus.test.lab:8181/google_containers --iso-url=file:///home/edxi/cache/iso/minikube-v1.5.1.iso --registry-mirror=http://nexus.test.lab:8181 --insecure-registry="nexus.test.lab:8181"`
* 不要翻墙：
  * 阿里云版本，只支持--vm-driver是vbox或none\(none也就是docker，须要配置防火墙、docker代理等\)，[https://github.com/AliyunContainerService/minikube/releases](https://github.com/AliyunContainerService/minikube/releases)查看现在支持版本，`curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.2.0  minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/`
  * [https://www.virtualbox.org/wiki/Linux\_Downloads](https://www.virtualbox.org/wiki/Linux_Downloads)找到下载连接，下载后直接yum install VirtualBox-版本，  `yum install  gcc make  perl kernel-header kernel-devel`，    装好后重启运行`/sbin/vboxconfig`
  * `minikube start --image-mirror-country cn --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso  minikube-v1.5.0.iso --registry-mirror=https://dkxtkkl2.mirror.aliyuncs.com # --insecure-registry="nexus.test.lab:8181"  --docker-env HTTP_PROXY=http://192.168.0.227:33080`
  * 本地环境： `minikube start --image-repository nexus.test.lab:8181/google_containers --iso-url=file:///root/cache/iso  minikube-v1.5.0.iso --registry-mirror=http://nexus.test.lab:8181 --insecure-registry="nexus.test.lab:8181"`
* kvm方式和vbox须要`egrep -q 'vmx|svm' /proc/cpuinfo && echo yes || echo no`开启vt
* kvm 安装

  ```bash
  yum update
  yum install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install
  systemctl start libvirtd.service
  systemctl enable libvirtd.service
  useradd edxi
  usermod -a -G libvirt edxi
  ```

* 直接安装命令

  ```bash
  yum install -y kubectl

  minikube start --image-mirror-country cn # --registry-mirror=https://dkxtkkl2.mirror.aliyuncs.com --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers

  minikube status

  // 如果有proxy，加上export no_proxy=$no_proxy,192.168.39.34 #(minikube虚拟机的IP)
  ```

## kubeadm安装群集

### 所有节点

#### 存在本地nexus的yum repo

```bash
[nexus-centos]
name=Nexus CentOS Repository
baseurl=http://nexus.test.lab:8081/repository/yum-centos/$releasever/os/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
proxy=_none_

[nexus-centos-updates]
name=Nexus CentOS Updates Repository
baseurl=http://nexus.test.lab:8081/repository/yum-centos/$releasever/updates/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
proxy=_none_

[nexus-centos-extras]
name=Nexus CentOS Extras Repository
baseurl=http://nexus.test.lab:8081/repository/yum-centos/$releasever/extras/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
proxy=_none_

[nexus-epel]
name = Nexus EPEL Repository
baseurl = http://nexus.test.lab:8081/repository/yum-epel/$releasever/$basearch
gpgcheck = 0
proxy=_none_

[nexus-kubernetes]
name = Nexus Kubernetes Repository
baseurl = http://nexus.test.lab:8081/repository/yum-kubernetes/kubernetes-el$releasever-$basearch/
gpgcheck = 0
proxy=_none_

[nexus-docker-ce-stable]
name = Nexus Docker CE Stable Repository
baseurl = http://nexus.test.lab:8081/repository/yum-docker-ce/$releasever/$basearch/stable
gpgcheck = 0
proxy=_none_

[nexus-ceph-nautilus]
name = Nexus ceph nautilus Repository
baseurl = http://nexus.test.lab:8081/repository/yum-ceph/rpm-nautilus/el7/$basearch
gpgcheck = 0
type = rpm-md
proxy=_none_

[nexus-ceph-nautilus-noarch]
name = Nexus ceph nautilus noarch Repository
baseurl = http://nexus.test.lab:8081/repository/yum-ceph/rpm-nautilus/el7/noarch
gpgcheck = 0
type = rpm-md
proxy=_none_
```

#### [bridge-nf-call-iptables is disabled with overlay storage driver](https://github.com/moby/moby/issues/24809)

```bash
# Enable IP Forwarding
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

#### 网络没法拿到镜像

* 镜像可以先手动下载到本地，然后tag
* 如果可以直接访问网络，可以通过国内代理下载gcr，其他用国内加速镜像。用来下载gcr镜像的国内代理，编辑增加一个docker启动参数 `vi /usr/lib/systemd/system/docker.service`

  ```bash
  Environment="HTTPS_PROXY=http://www.ik8s.io:10080"
  Environment="NO_PROXY=127.0.0.0/8,172.0.0.0/8"
  ```

* 推荐方式——自建registry，然后把镜像都放在自建registry:
  * [docker国内加速列表](https://juejin.im/post/5cd2cf01f265da0374189441)
  * [Azure国内docker镜像](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy)
  * 可以把上述镜像创建到Nexus docker proxy repository里，然后做成groupg给端口8181， anonymous，可以这么访问到：

    ```bash
    docker pull nexus.test.lab:8181/library/busybox
    docker pull nexus.test.lab:8181/google-containers/kube-apiserver:v1.15.2
    docker pull nexus.test.lab:8181/coreos/kube-state-metrics:v1.7.2
    ```

  * 注意把`/etc/docker/daemon.json`加上`"insecure-registries": ["nexus.test.lab:8181"]`

#### Disable swap

```bash
swapoff -a
sed -i 's/^\(.*swap.*\)$/#\1/' /etc/fstab
```

或者可以 `vi /etc/sysconfig/kubelet KUBELET_EXTRA_ARGS="--fail-swap-on=false"`，kubeadm的时候加入参数 `--ignore-preflight-errors=Swap`

#### 配置 IPVS 模块

**注意，ipvs模式后不能用内网穿透访问（跨网段或ip映射都没问题）**

由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

* ip\_vs
* ip\_vs\_rr
* ip\_vs\_wrr
* ip\_vs\_sh
* nf\_conntrack\_ipv4

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF

#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4

EOF
```

执行脚本并查看是否正常加载内核模块：

```bash
#修改脚本权限
chmod 755 /etc/sysconfig/modules/ipvs.modules 

#执行脚本
bash /etc/sysconfig/modules/ipvs.modules 

#查看是否已经正确加载所需的内核模块
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

安装 ipset

```bash
yum install ipset
```

#### 配置资源限制

```bash
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```

### master安装步骤

```bash
yum install kubelet kubeadm kubectl docker-ce #kubectl是客户端，可以不装
systemctl enable kubelet
systemctl enable docker

mkdir /etc/docker/

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["http://nexus.test.lab:8181"],
  "insecure-registries": ["nexus.test.lab:8181"]
}
EOF

# Restart Docker
systemctl daemon-reload
systemctl restart docker
```

#### 防火墙

* master经测试最少需要启用端口

```bash
firewall-cmd --add-port=6443/tcp --add-port=10250/tcp --permanent
firewall-cmd --reload
```

* 参考[Kubernetes on CentOS 7 with Firewalld](https://medium.com/platformer-blog/kubernetes-on-centos-7-with-firewalld-e7b53c1316af)
  * On Master\*\* open the following ports and restart the service

    ```bash
    firewall-cmd --permanent --add-port=6443/tcp
    firewall-cmd --permanent --add-port=2379-2380/tcp
    firewall-cmd --permanent --add-port=10250/tcp
    firewall-cmd --permanent --add-port=10251/tcp
    firewall-cmd --permanent --add-port=10252/tcp
    firewall-cmd --permanent --add-port=10255/tcp
    firewall-cmd --permanent --add-port=8472/udp
    firewall-cmd --add-masquerade --permanent
    # only if you want NodePorts exposed on control plane IP as well
    firewall-cmd --permanent --add-port=30000-32767/tcpsystemctl restart firewalld
    ```

  * On Worker Nodes\*\* open the following ports and restart the service

    ```bash
    firewall-cmd --permanent --add-port=10250/tcp
    firewall-cmd --permanent --add-port=10255/tcp
    firewall-cmd --permanent --add-port=8472/udp
    firewall-cmd --permanent --add-port=30000-32767/tcp
    firewall-cmd --add-masquerade --permanentsystemctl restart firewalld
    ```
* 关闭防火墙

  ```bash
  # 在每台机器上关闭防火墙，清理防火墙规则，设置默认转发策略：
  systemctl stop firewalld
  systemctl disable firewalld
  iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X && iptables -X -t nat
  iptables -P FORWARD ACCEPT
  firewall-cmd --state
  ```

#### 关闭SELinux

关闭SELinux，否则后续K8S挂载目录时可能报错 Permission denied：

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

#### master安装参数

* --kubernetes-version=v1.16.2 不指定会上网拿[https://dl.k8s.io/release/stable-1.txt](https://dl.k8s.io/release/stable-1.txt)
* 使用flannel必须在init时指定--pod-network-cidr=10.244.0.0/16
* 直接参数初始化或者导出yml文件初始化
  * `kubeadm init --ignore-preflight-errors=Swap --image-repository nexus.test.lab:8181/google-containers --kubernetes-version=v1.16.3 --pod-network-cidr=10.244.0.0/16`，这里没有开启ipvs，已经安装好的群集，可以通过如下方法修改到ipvs模式：

    Edit the configmap

    ```bash
    kubectl edit configmap kube-proxy -n kube-system
    ```

    change mode from "" to ipvs

    ```bash
    mode: ipvs
    ```

    Kill any kube-proxy pods

    ```bash
    kubectl get po -n kube-system
    kubectl delete po -n kube-system <pod-name>
    ```

    Verify kube-proxy is started with ipvs proxier

    ```bash
    kubectl logs [kube-proxy pod] -n kube-system | grep "Using ipvs Proxier"
    ```

  * 建议导出yaml这种方式

    ```bash
    # 导出配置文件
    kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
    ```

    ```yaml
    # 修改配置为如下内容，主要需要修改的内容下面都有注释
    apiVersion: kubeadm.k8s.io/v1beta1
    bootstrapTokens:
    - groups:
      - system:bootstrappers:kubeadm:default-node-token
      token: abcdef.0123456789abcdef
      ttl: 24h0m0s
      usages:
      - signing
      - authentication
    kind: InitConfiguration
    localAPIEndpoint:
      # 修改为主节点 IP，可以是0.0.0.0
      advertiseAddress: 192.168.141.130
      bindPort: 6443
    nodeRegistration:
      criSocket: /var/run/dockershim.sock
      name: kubernetes-master
      taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
    ---
    apiServer:
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta1
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: ""
    controllerManager: {}
    dns:
      type: CoreDNS
    etcd:
      local:
        dataDir: /var/lib/etcd
    # 国内不能访问 Google，修改为阿里云，也可以自建registry，就直接改成google_containers
    imageRepository: registry.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    # 修改版本号
    kubernetesVersion: v1.14.1
    networking:
      dnsDomain: cluster.local
      # 配置成 Calico 或 flannel 的默认podSubnet网段
      podSubnet: "10.244.0.0/16"
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
    ---
    # 开启 IPVS 模式
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    featureGates:
      SupportIPVSProxyMode: true
    mode: ipvs
    ```

    ```bash
    # 可选
    # 查看所需镜像列表
    kubeadm config images list --config kubeadm.yml
    # 拉取镜像
    kubeadm config images pull --config kubeadm.yml
    ```

    ```bash
    kubeadm init --config=kubeadm.yml | tee kubeadm-init.log
    ```
* 完成后拿到加入cluster的 kubeadm命令（最好先保存下来，token 24小时后失效，可以重新生成和获取`kubeadm token create; kubeadm token list`，[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)）
  * `kubeadm join 10.184.101.10:6443 --token rv2jcy.fpon2ycg0ajhky6g --discovery-token-ca-cert-hash sha256:44aaa597b8cf24643c5cb0e2d004409bd3610d94a4c70f64e40d93c92dfdcf67`（这一串可以`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`）
* 并为kubectl创建.kube命令目录

  ```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

* 还提示了要部署网络插件 You should now deploy a pod network to the cluster. Run "kubectl apply -f \[podnetwork\].yaml" with one of the options listed at: [https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
* 如果上述步骤有问题，kubeadm reset一下删除跑错的环境，然后重来
* 如果使用proxy，注意把master的ip加入no\_proxy环境变量，kubectl config view看

#### 安装网络插件

* 按照这个配置网络插件[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/\#pod-network](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
* 用calico须要[https://docs.projectcalico.org/v3.11/getting-started/kubernetes/installation/calico](https://docs.projectcalico.org/v3.11/getting-started/kubernetes/installation/calico)：
  * kubeadm时-pod-network-cidr=10.244.0.0/16和--allocate-node-cidrs=true
  * 如需要使用本地image，下载calico.yml后修改image到本地库
* 用flannel须要：
  * kubeadm时-pod-network-cidr=10.244.0.0/16
  * /proc/sys/net/bridge/bridge-nf-call-iptables为1
  * 开启UDP ports 8285 and 8472 ————可能不需要开启，因为是出栈的UDP
  * 安装文档使用如下命令安装:

    ```bash
    # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    #内网环境先下载好yml文件，然后修改image的registry，如下用自建nexus的（也可以自己下载好image到本地后tag）
    curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    sed -i "s/quay.io\/coreos\/flannel/nexus.test.lab:8181\/coreos\/flannel/g" kube-flannel.yml
    kubectl apply -f kube-flannel.yml
    rm -f kube-flannel.yml
    ```

### Node上

#### node防火墙

* node经测试最少需要启用端口

```bash
firewall-cmd --add-port=10250/tcp --permanent
firewall-cmd --reload
```

#### node关闭SELinux

#### 安装配置

`yum install kubelet kubeadm docker-ce`

* docker daemon.json

  ```bash
   mkdir /etc/docker/
   # Setup daemon.
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
     "max-size": "100m"
   },
   "storage-driver": "overlay2",
   "storage-opts": [
    "overlay2.override_kernel_check=true"
   ],
     "registry-mirrors": ["http://nexus.test.lab:8181"],
     "insecure-registries": ["nexus.test.lab:8181"]
   }
   EOF
   # Restart Docker
   systemctl daemon-reload
   systemctl restart docker
  ```

* 如须要`vi /etc/sysconfig/kubelet` `KUBELET_EXTRA_ARGS="--fail-swap-on=false"`
* `systemctl enable kubelet docker`
* `systemctl start docker`
* 设置bridge-nf-call-iptables

  ```bash
  cat <<EOF >  /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  sysctl --system
  ```

* `kubeadm join 10.184.101.10:6443 --token rv2jcy.fpon2ycg0ajhky6g --discovery-token-ca-cert-hash sha256:44aaa597b8cf24643c5cb0e2d004409bd3610d94a4c70f64e40d93c92dfdcf67 #如需要--ignore-preflight-errors=Swap`

### k8s升级

#### 升级k8s

```bash
yum update #升级kubeadm版本
kubeadm upgrade plan <version>
kubeadm upgrade apply <version>
```

#### 维护

* 比如须要升级docker:
  * `kubectl drain node1 --ignore-daemonsets`
  * 在Node上完成升级docker后，可以用`docker system prune`来清理exited containers
  * `kubectl uncordon node1`

### k8s删除

删除节点[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/\#tear-down](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down)

1. 从cluster里删节点

   ```bash
   kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
   kubectl delete node <node name>
   ```

2. 在删除的节点上清理

   ```bash
   kubeadm reset
   iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X && iptables -X -t nat
   ipvsadm -C
   ```

## k8s使用

### deploy

#### 滚动升级

* 可以patch/edit/set image/apply -f等，用rollout来history/pause/resume/status/undo
* deploy升级后，可以describe看现在image用哪个，也可以看到old和new的replicaset用哪个，
* 如果是金丝雀升级，可以用
  1. 用label方式把service指定到新老deploy上，[https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/\#canary-deployments](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)
  2. 用`kubectl pause`方式
* 看现在哪些Pod的image已经升级了kubectl get pod -o=custom-columns=NAME:.metadata.name,IMAGE:.spec.containers\[\*\].image -l app=deploy-demo

### daemonset

更新的时候只能删一个建一个，通过指定updateStrategy更新方式

### pv pvc storageclass

* pv是对应实际物理存储的。
* pvc是申请需要大小的存储量，然后去绑定现有符合大小的pv，一旦绑定，那个pv就不能再给其他pvc用了，所以可能存在申请1g pvc绑定5g pv的情况（但实际可以使用的量不一定，目前测试看下来nfs存储，不是pvc要的1g和pv的5g，而是nfs盘所在硬盘的真实容量～，cephfs也有同样的问题，但rbd没有）
* StorageClass自动创建pv和pvc（动态存储供应）
  * 不在 [Provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)里的需要使用孵化器项目的[external-storage](https://github.com/kubernetes-incubator/external-storage)手动创建
  * pvc创建后自动创建出pv，pv默认reclaim policy是delete，删除pvc时pv自动删除，也删除对应存储上对象（如ceph rbd的存放的objects）
  * 可以直接pvc调用storageclass，也可以sts的volumeClaimTemplates来调用创建pvc

**ceph rbd**

* 部署ceph rbd provisioner

```bash
# 下载external storage的git repo
git clone https://github.com/kubernetes-incubator/external-storage.git

# 一般使用rbac方式部署，按情况可能修改两个yaml
# vi external-storage/ceph/rbd/deploy/rbac/clusterrole.yaml，增加如下行
#  - apiGroups: [""]
#    resources: ["secrets"]
#    verbs: ["get","list"]
# vi external-storage/ceph/rbd/deploy/rbac/deployment.yaml，修改image为本地registry
cd external-storage/ceph/rbd/deploy
NAMESPACE=default # change this if you want to deploy it in another namespace
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/clusterrolebinding.yaml ./rbac/rolebinding.yaml
kubectl -n $NAMESPACE apply -f ./rbac
```

* 在所有k8s node上安装ceph-common`yum install ceph-common -y`
* 在任何一台装有个ceph-common和kubectl的机器上（并配置了ceph和kubectl管理key），创建ceph admin key的secret

```bash
ceph auth get client.admin 2>&1 |grep "key = " |awk '{print  $3'} |xargs echo -n > /tmp/key
kubectl create secret generic ceph-admin-secret --from-file=/tmp/key --namespace=kube-system --type=kubernetes.io/rbd
```

* 创建所需ceph pool和用户secrete

```bash
ceph osd pool create ceph-pool1 8 8
ceph auth add client.kubeuser1 mon 'allow r' osd 'allow rwx pool=kube'
ceph auth get-key client.kubeuser1 > /tmp/key
kubectl create secret generic ceph-secret-kubeuser1 --from-file=/tmp/key --namespace=kube-system --type=kubernetes.io/rbd # 可以在default，namespace在storage class可以里指明
```

* 创建storage class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-pool1
provisioner: ceph.com/rbd
parameters:
  monitors: 10.184.101.245:6789,10.184.101.246:6789,10.184.101.247:6789 # 经测试不能用主机名，只能用ip
  adminId: admin
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: kube-system
  pool: ceph-pool1
  userId: kubeuser1
  userSecretName: ceph-secret-kubeuser1
  userSecretNamespace: kube-system
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

* pvc的spec写上`storageClassName: ceph-pool1`即可创建对应pvc和pv
* 自动创建的pv的reclaim policy是delete，会在pvc删除时删除。但在ceph rbd provisioner在ceph pool quota满的时候会不能正常删除pv，手动删除pv不能对应删除image，在ceph admin node上可以手动删除

```bash
rbd rm kube-pool/kubernetes-dynamic-pvc-9961abb8-4046-11ea-ac16-826e98441635
```

**cephfs**

* 部署cephfs provisioner

```bash
# 下载external storage的git repo
git clone https://github.com/kubernetes-incubator/external-storage.git

# 一般使用rbac方式部署，按情况可能修改yaml
# vi external-storage/ceph/cephfs/deploy/rbac/deployment.yaml，修改image为本地registry
cd external-storage/ceph/cephfs/deploy
NAMESPACE=cephfs # change this if you want to deploy it in another namespace
kubectl create ns $NAMESPACE # 如果不是现有的名称空间的话，需要先创建起来
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/*.yaml
sed -r -i "N;s/(name: PROVISIONER_SECRET_NAMESPACE.*\n[[:space:]]*)value:.*/\1value: $NAMESPACE/" ./rbac/deployment.yaml
kubectl -n $NAMESPACE apply -f ./rbac
```

* 在所有k8s node上安装ceph-common`yum install ceph-common -y`。须注意最新的ceph-common的v14.2.6和v14.2.5使用的mount.ceph限制了用户名长短，会导致k8s provisioner自动创建的ceph user挂载失败，这个ceph [issue](https://github.com/ceph/ceph/commit/8db2a44c9438749be98d41fb309f10d5084805df#diff-7d5d3406f4cfc7ade26efbcf40725fba) 已经修复，估计后续版本可以正常挂载。目前可以指定安装v14.2.4版本。
* 在任何一台装有个ceph-common和kubectl的机器上（并配置了ceph和kubectl管理key），创建ceph admin key的secret

```bash
ceph auth get-key client.admin > /tmp/secret
kubectl create secret generic ceph-secret-admin --from-file=/tmp/secret --namespace=cephfs
```

* 创建storage class

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mycephfs
provisioner: ceph.com/cephfs
parameters:
    monitors: 10.184.101.245:6789,10.184.101.246:6789,10.184.101.247:6789 # 经测试不能用主机名，只能用ip
    adminId: admin
    adminSecretName: ceph-secret-admin
    adminSecretNamespace: "cephfs"
    claimRoot: /pvc-volumes #这个可以任意目录（也可以直接用/），通过这个storage class创建出来的pv会自动在创建类似于/pvc-volumes/kubernetes/kubernetes-dynamic-pvc-85e12989-41dd-11ea-bed7-1a25aaeb420a这样的目录，并自动创建一个ceph用户和相应的k8s secret对这个目录有权限。
```

* pvc的spec写上`storageClassName: mycephfs`即可创建对应pvc和pv

