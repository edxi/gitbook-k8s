# docker笔记

- docker使用代理 <https://docs.docker.com/config/daemon/systemd/#httphttps-proxy>

- docker commit/save/export区别
  - commit用于自定义镜像，建议使用dockerfile方式替代
  - save用于迁移镜像，建议使用docker hub或registry方式替代
  - export用于导出导入镜像快照，临时调试使用
  - commit/export是修改好的容器，做成镜像/导出文件，save是直接把镜像导出文件
  - commit/save保留历史层，可以用docker history看到，可以回滚到一层，export不保留

-v和--mount都是挂volume/目录/文件，后者可以制定mount类型和RO/RW

- 访问容器网络：
  - -P是随机分配端口
  - -p指定IP：port:containerPort，可以用多次以指定多个Port，省略port即随机分配Port

- docker-compose
  - 最好不要通过apt安装，会带上下面那么多的包，其中有包和docker client冲突，会导致docker login失败

  ```txt
  cgroupfs-mount golang-docker-credential-helpers libpython-stdlib libpython2.7-minimal libpython2.7-stdlib libsecret-1-0  libsecret-common python python-asn1crypto python-backports.ssl-match-hostname python-cached-property python-certifi  python-cffi-backend python-chardet python-cryptography python-docker python-dockerpty python-dockerpycreds python-docopt  python-enum34 python-funcsigs python-functools32 python-idna python-ipaddress python-jsonschema python-minimal python-mock  python-openssl python-pbr python-pkg-resources python-requests python-six python-texttable python-urllib3 python-websocket  python-yaml python2.7 python2.7-minimal
  ```

  - 推荐通过docker方式运行（也可以pip或github上官方curl方式下载对应release使用），docker方式也是curl
    - `curl -L https://github.com/docker/compose/releases/download/版本号/run.sh > /usr/local/bin/docker-compose`
    - `chmod +x /usr/local/bin/docker-compose`

- docker-machine
  - 安装——使用github release里的curl命令下载安装，如果出现github的aws s3 connection refuse,手动加dns，指定香港s3服务器219.76.4.4 github-cloud.s3.amazonaws.com
  - vm重启后IP改变会报错Unable to query docker version:... certificate is valid for 192.168.99.xxx, not 192.168.99.xxx，有docker-machine regenerate-certs vmname重新生成证书就行
  - 使用vmwarevsphere， docker-machine create -d vmwarevsphere来创建，参见 <https://docs.docker.com/machine/drivers/vsphere/#options>，  create时还可以指定--engine-insecure-registry（使用自己的registry）--engine-registry-mirror，--engine-registry-cert，--swarm ，--engine-env（比如`--engine-env HTTP_PROXY=http://example.com:8080`）等参数
  
  ```bash
  export VSPHERE_VCENTER='hpshvcenter02.vpcsh.com.cn'# vCenter IP/FQDN
  export VSPHERE_USERNAME='administrator@vsphere.local'# vCenter user
  export VSPHERE_PASSWORD='0!MoneyGomyHome'# vCenter user password
  export VSPHERE_NETWORK='107'# PortGroup 这个网段要有DHCP，否则没法连上管理这个DOCKER MACHINE，这个有bug，必须要在docker-machine命里使用数
  export VSPHERE_DATASTORE='SS8400/8400-1t' # Datastore
  export VSPHERE_DATACENTER='SHVPClab' # Datacenter name
  export VSPHERE_FOLDER='LabVM/labdockervm'
  #could be ommited if DRS, export VSPHERE_HOSTSYSTEM='SHVPClab/*' # cluster name (inside the datacenter)
  ```

- swarm
  - 如果你的 Docker 主机有多个网卡，拥有多个 IP，必须使用 --advertise-addr 指定 IP
  - 使用 docker service create 一次只能部署一个服务，使用 docker-compose.yml 我们可以一次启动多个关联的服务,Swarm 集群中使用 docker-compose.yml 我们用 docker stack 命令(也就是多个service是一个stack)
