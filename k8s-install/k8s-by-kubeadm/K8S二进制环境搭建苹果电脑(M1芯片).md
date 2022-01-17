

# 一、环境介绍

- 硬件设备

  2021款 macbook pro M1Pro 芯片

- 软件
  - 1、PD17
    - 请自行安装此软件(收费软件，有试用期方式使用，如何处理请自己脑补)
  - 2、linux 操作系统
    - [ubuntu-20.04.3-live-server-arm64.iso](https://releases.ubuntu.com/20.04/)

  - 3、git
  - 4、make
  - 5、docker
***注意*** 3-5 软件是在虚拟机里面安装的

### 问题直达
- calico-apiserver pullErr 直接看第六步骤[calico-apiserver 拉取镜像失败]

# 二、操作系统安装

- 安装PD17
- Ubuntu系统安装
    - 1、选择其它安装镜像安装
    - 2、添加再一个网卡
      - 第一个网卡使用网络共享模式(其实就是NAT方式)，此网卡主要用来虚拟机与外部网络进行通讯使用
        设置网络转发 源端口[6022] 目标端口[22] (之后你切换了工作环境或者网络,都不会影响你ssh登录)
      - 第二个网卡使用host-only,此网卡是给K8S使用，打开高级设置，取消自动动态分配，
        将网路起始IP设置为192.168.34.1，终止IP为192.168.34.254
    - 3、设置硬件资源
      - 内存 12G
      - CPU 4核
      - 磁盘 30G以上

    - 4、启动虚拟机进行系统安装
      - 除了ssh服务之外，其它软件都不需要安装
      - 全部使用默认配置
      - 用户名建议使用[cadmin]密码相同

## 三、设置系统网路
  - 1、查看本机ip
    - 通过`ip a`命令查询当前网卡名称及IP地址，一般有一个NAT地址为10.开头的，另一个为host-only的网卡
  - 2、修改网络配置文件
    - 使用`sudo su `输入密码切换至root用户
    - `vi /etc/netplan/00-installer-config.yaml` 打开网络配置文件
      ```
      network:
        ethernets:
              enp0s3:
                dhcp4: true
              enp0s8:
                dhcp4: no
              addresses:
                - 192.168.34.2/24
        version: 2
      ```
      ***注意***
      - enp0s3 为NAT网卡名称
      - enp0s8 为host-only 网卡名称
  - 3、使网络配置生效
    - `netplan apply`

  - 4、关闭交换空间
    ```sh
    swapoff -a
    vi /etc/fstab
    ```
    删除 /etc/fstab 文件中包含 swap的行(一般是最后一行)
## 四、安装 docker
- 1、命令安装
  - `install docker.io`
- 2、更改服务启动项
  - 原因：docker 默认的服务启动项是cgroupdriver 和k8s的启动项会产生冲突

  - ***注意:*** daemon.json 这个文件并不存在，需要自己创建并写一下内容
  ```sh
  vi /etc/docker/daemon.json
  {
    "exec-opts": ["native.cgroupdriver=systemd"]
  }
  systemctl daemon-reload
  systemctl restart docker
  ```
## 五、安装K8S
- 1、让iptable监控你的K8S流量
  ```shell
  $ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  br_netfilter
  EOF

  $ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  $ sudo sysctl --system
  ```
- 2、更新apt 包的索引，并安装k8s依赖的包
  ```shell
  $ sudo apt-get update
  $ sudo apt-get install -y apt-transport-https ca-certificates curl
  ```
- 3、安装kubeadm

  ```shell
  $ sudo curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
  $ sudo tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
  deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
  EOF
  $ sudo apt-get update
  $ sudo apt-get install -y kubelet kubeadm kubectl
  $ sudo apt-mark hold kubelet kubeadm kubectl
  ```

- 4、kubeadm初始化
  ```shell
  $ echo "192.168.34.2 cncamp.com" >> /etc/hosts
  ```

  ***注意：***
  - k8s初始化,是要去docker hub 拉取镜像的，所以在初始化之前，应该先设置dockerhub的代理,设置代理会修改[/etc/docker/daemon.json]文件。
      这个文件在修改docker服务启动项时创建过，切记不要将原有配置覆盖，应该在"{}"中以","分割添加新的配置
  - 初始化操作会进行镜像拉取时间比较久，只需等待即可。

  ```shell
  $ kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.22.2 \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=192.168.34.2
  ```

  - 5、拷贝k8s配置

  ```shell
  $ mkdir -p $HOME/.kube
  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

  - 6、给master打污点

  ```shell
  $ kubectl taint nodes --all node-role.kubernetes.io/master-
  ```

  - 7、Install calico cni plugin (安装calico CNI 插件)

    https://docs.projectcalico.org/getting-started/kubernetes/quickstart

  ```shell
  $ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
  $ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
  ```
  ***注意***：
  - 可以通过一下命令查看 calico 安装情况,这个安装也比较慢，请耐性等待
    ```shell
    kubectl get pod -n calico-system
    ```

## 六、calico-apiserver 拉取镜像失败
  这个是M1失败的核心问题，其实查询官方，你会发现calico已经支持了M1(Arm64)架构了，但是docker仓库内部并没有对应的镜像。
所以我们需要自己下载源码进行编译和打包，下面我们来描述怎么自己打包镜像
- 1、安装必要软件
  ```shell
    apt-get install golang-go
    apt-get install make
  ```
- 2、来取calico的源码
  - `git clone https://github.com/projectcalico/calico`
- 3、切换目录并执行编译
  - `cd calico/apiserver && make build`
- 4、查询生成的镜像并改标签
  - `docker images`
  - `docker tag calico/apiserver calico/apiserver:verison`
  - `docker tag apiserver apiserver:verison`
  - ***注意*** verison是calico的版本号，其它镜像是多少你就设置多少

- 5、查看k8s是否启动成功
  - `kubectl get pods -A`
  - 全部为 runing 状态及启动成功
