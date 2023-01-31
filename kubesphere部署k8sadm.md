## [原文链接](https://kubesphere.io/zh/docs/v3.3/installing-on-linux/introduction/multioverview/)
## 概念
多节点集群由至少一个主节点和一个工作节点组成。您可以使用任何节点作为**任务机**来执行安装任务，也可以在安装之前或之后根据需要新增节点（例如，为了实现高可用性）。

- **Control plane node**：主节点，通常托管控制平面，控制和管理整个系统。
- **Worker node**：工作节点，运行部署在工作节点上的实际应用程序。
## 步骤 1：准备 Linux 主机
请参见下表列出的硬件和操作系统要求。在本教程所演示多节点安装示例中，您需要按照下列要求准备至少三台主机。如果您节点的资源充足，也可以将 KubeSphere 容器平台安装在两个节点上。

| **CentOS** _7_.x | CPU：2 核，内存：4 G，硬盘：40 G |
| --- | --- |

### 节点要求

- 所有节点必须都能通过 SSH 访问。
- 所有节点时间同步。
- 所有节点都应使用 sudo/curl/openssl/tar。
### 容器运行时
您的集群必须有一个可用的容器运行时。如果您使用 KubeKey 搭建集群，KubeKey 会默认安装最新版本的 Docker。或者，您也可以在创建集群前手动安装 Docker 或其他容器运行时。
### 依赖项要求
KubeKey 可以一同安装 Kubernetes 和 KubeSphere。根据要安装的 Kubernetes 版本，需要安装的依赖项可能会不同。您可以参考下表，查看是否需要提前在节点上安装相关依赖项
### 网络和 DNS 要求

- 请确保 /etc/resolv.conf 中的 DNS 地址可用，否则，可能会导致集群中的 DNS 出现问题。
- 如果您的网络配置使用防火墙规则或安全组，请务必确保基础设施组件可以通过特定端口相互通信。建议您关闭防火墙。有关更多信息，请参见端口要求。
- 支持的 CNI 插件：Calico 和 Flannel。其他插件也适用（例如 Cilium 和 Kube-OVN 等），但请注意它们未经充分测试。
- 建议您使用干净的操作系统（即不安装任何其他软件）。否则，可能会产生冲突。
- 如果您从 dockerhub.io 下载镜像时遇到问题，建议提前准备仓库的镜像地址（即加速器）。请参见[为安装配置加速器](https://kubesphere.io/zh/docs/v3.3/faq/installation/configure-booster/)或[为 Docker Daemon 配置仓库镜像](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon)。
- 建议把下面的机器给的配置更高点
| **主机 IP** | **主机名** | **角色** |
| --- | --- | --- |
| 192.168.88.4 | master | master, etcd |
| 192.168.88.5 | node1 | node |
| 192.168.88.6 | node2 | node |

```shell
systemctl stop firewalld 
systemctl disable firewalld
setenforce 0
vi /etc/sysconfig/selinux  
将文件中的SELINUX=enforcing改为disabled
yum -y install curl socat conntrack vim
#时间同步,保存
yum install ntp
ntpdate cn.pool.ntp.org
crontab -e
00 12 * * * /sbin/ntpdate cn.pool.ntp.org

```
```shell
vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.88.5 k8s-node1
192.168.88.6 k8s-node2

```
## 步骤 2：下载 KubeKey
```shell
export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.2 sh -
chmod +x kk
```
## 步骤 3：创建集群
对于多节点安装，您需要通过指定配置文件来创建集群。
### 1. 创建示例配置文件
命令如下：
```shell
./kk create config [--with-kubernetes version] [--with-kubesphere version] [(-f | --file) path]
安装 KubeSphere 3.3 的建议 Kubernetes 版本：v1.19.x、v1.20.x、v1.21.x、* v1.22.x、* v1.23.x 和 * v1.24.x。带星号的版本可能出现边缘节点部分功能不可用的情况。因此，如需使用边缘节点，推荐安装 v1.21.x 及之前的版本。如果不指定 Kubernetes 版本，KubeKey 将默认安装 Kubernetes v1.23.10。有关受支持的 Kubernetes 版本的更多信息，请参见支持矩阵。

如果您在此步骤的命令中不添加标志 --with-kubesphere，则不会部署 KubeSphere，只能使用配置文件中的 addons 字段安装，或者在您后续使用 ./kk create cluster 命令时再次添加这个标志。

如果您添加标志 --with-kubesphere 时不指定 KubeSphere 版本，则会安装最新版本的 KubeSphere。
```
```shell
./kk create config --with-kubesphere v3.3.1
```
### 2. 编辑配置文件
如果您不更改名称，那么将创建默认文件 config-sample.yaml。编辑文件，以下是多节点集群（具有一个主节点）配置文件的示例。
```shell
#把文件中的主机IP和账号登录用户和密码修改，以及etcd数据库和控制节点工作节点分布，其他不用修改
vim config-sample.yaml

apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 192.168.88.4, internalAddress: 192.168.88.4, user: root, password: "1"}
  - {name: node1, address: 192.168.88.5, internalAddress: 192.168.88.5, user: root, password: "1"}
  - {name: node2, address: 192.168.88.6, internalAddress: 192.168.88.6, user: root, password: "1"}
  roleGroups:
    etcd:
    - master
    master:
    - master
    worker:
    - node1
    - node2


```
#### 主机
请参照上方示例在 hosts 下列出您的所有机器并添加详细信息。
name：实例的主机名。
address：任务机和其他实例通过 SSH 相互连接所使用的 IP 地址。根据您的环境，可以是公有 IP 地址或私有 IP 地址。例如，一些云平台为每个实例提供一个公有 IP 地址，用于通过 SSH 访问。在这种情况下，您可以在该字段填入这个公有 IP 地址。
internalAddress：实例的私有 IP 地址。
提示：

- 在安装 KubeSphere 之前，您可以使用 hosts 下提供的信息（例如 IP 地址和密码）通过 SSH 的方式测试任务机和其他实例之间的网络连接。
- 在安装前，请确保端口 6443 没有被其他服务占用，否则在安装时会产生冲突（6443 为 API 服务器的默认端口）。
#### roleGroups

- etcd：etcd 节点名称
- control-plane：主节点名称
- worker：工作节点名称
#### controlPlaneEndpoint（仅适用于高可用安装）
您需要在 controlPlaneEndpoint 部分为高可用集群提供外部负载均衡器信息。当且仅当您安装多个主节点时，才需要准备和配置外部负载均衡器。请注意，config-sample.yaml 中的地址和端口应缩进两个空格，address 应为您的负载均衡器地址。有关详细信息，请参见[高可用配置](https://kubesphere.io/zh/docs/v3.3/installing-on-linux/high-availability-configurations/ha-configuration/)。
#### addons
您可以在 config-sample.yaml 的 addons 字段下指定存储，从而自定义持久化存储插件，例如 NFS 客户端、Ceph RBD、GlusterFS 等。有关更多信息，请参见[持久化存储配置](https://kubesphere.io/zh/docs/v3.3/installing-on-linux/persistent-storage-configurations/understand-persistent-storage/)。
KubeSphere 会默认安装 [OpenEBS](https://openebs.io/)，为开发和测试环境配置 [LocalPV](https://kubernetes.io/docs/concepts/storage/volumes/#local)，方便新用户使用。在本多节点安装示例中，使用了默认存储类型（本地存储卷）。对于生产环境，您可以使用 Ceph/GlusterFS/CSI 或者商业存储产品作为持久化存储解决方案。

- 您可以编辑配置文件，启用多集群功能。有关更多信息，请参见[多集群管理](https://kubesphere.io/zh/docs/v3.3/multicluster-management/)。
- 您也可以选择要安装的组件。有关更多信息，请参见[启用可插拔组件](https://kubesphere.io/zh/docs/v3.3/pluggable-components/)。有关完整的 config-sample.yaml 文件的示例，请参见[此文件](https://github.com/kubesphere/kubekey/blob/release-2.2/docs/config-example.md)。
- 完成编辑后，请保存文件。
### 3. 使用配置文件创建集群
```shell
./kk create cluster -f config-sample.yaml
```
如果使用其他名称，则需要将上面的 config-sample.yaml 更改为您自己的文件。
整个安装过程可能需要 10 到 20 分钟，具体取决于您的计算机和网络环境。
### 4. 验证安装
安装完成后，您会看到如下内容：
```shell
#####################################################

###              Welcome to KubeSphere!           ###

#####################################################



Console: http://192.168.0.2:30880

Account: admin

Password: P@88w0rd



NOTES：

  1. After you log into the console, please check the

     monitoring status of service components in

     the "Cluster Management". If any service is not

     ready, please wait patiently until all components

     are up and running.

  2. Please change the default password after login.



#####################################################

https://kubesphere.io             20xx-xx-xx xx:xx:xx

#####################################################

```
现在，您可以通过 <NodeIP:30880 使用默认帐户和密码 (admin/P@88w0rd) 访问 KubeSphere 的 Web 控制台。
部署完成后需要配置docker加速器，否则不能使用，报错
请参见[为安装配置加速器](https://kubesphere.io/zh/docs/v3.3/faq/installation/configure-booster/)或[为 Docker Daemon 配置仓库镜像](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon)。
```shell
mkdir -p /etc/docker
vim /etc/docker/daemon.json 
{
"registry-mirrors": ["https://o2h0rw82.mirror.aliyuncs.com"]
}

systemctl daemon-reload
systemctl restart docker
```
## 启用 kubectl 自动补全
KubeKey 不会启用 kubectl 自动补全功能，请参见以下内容并将其打开：
```shell
# Install bash-completion

apt-get install bash-completion



# Source the completion script in your ~/.bashrc file

echo 'source <(kubectl completion bash)' >>~/.bashrc



# Add the completion script to the /etc/bash_completion.d directory

kubectl completion bash >/etc/bash_completion.d/kubectl

```
