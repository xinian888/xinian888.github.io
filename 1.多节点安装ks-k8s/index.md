# 1.多节点安装ks+k8s


<!--more-->

# 概念

多节点集群由至少一个主节点和一个工作节点组成。您可以使用任何节点作为任务机来执行安装任务，也可以在安装之前或之后根据需要新增节点（例如，为了实现高可用性）。

- Control plane node：主节点，通常托管控制平面，控制和管理整个系统。
- Worker node：工作节点，运行部署在工作节点上的实际应用程序。

# 步骤 1：准备 Linux 主机

**硬件推荐配置**

| 操作系统                                           | 最低配置                                                  |
| -------------------------------------------------- | --------------------------------------------------------- |
| Ubuntu 16.04, 18.04                                | 2 核 CPU，4 GB 内存，40 GB 磁盘空间                       |
| Debian Buster, Stretch                             | 2 核 CPU，4 GB 内存，40 GB 磁盘空间                       |
| CentOS 7.x                                         | 2 核 CPU，4 GB 内存，40 GB 磁盘空间                       |
| Red Hat Enterprise Linux 7                         | 2 核 CPU，4 GB 内存，40 GB 磁盘空间Debian Buster, Stretch |
| SUSE Linux Enterprise Server 15/openSUSE Leap 15.2 | 2 核 CPU，4 GB 内存，40 GB 磁盘空间Debian Buster, Stretch |

**依赖项要求**

KubeKey 可以将 Kubernetes 和 KubeSphere 一同安装。针对不同的 Kubernetes 版本，需要安装的依赖项可能有所不同。您可以参考以下列表，查看是否需要提前在节点上安装相关的依赖项。


| 依赖项    | Kubernetes 版本 ≥ 1.18 | Kubernetes 版本 < 1.18 |
| --------- | ---------------------- | ---------------------- |
| socat     | 必须                   | 可选但建议             |
| conntrack | 必须                   | 可选但建议             |
| ebtables  | 可选但建议             | 可选但建议             |
| ipset     | 可选但建议             | 可选但建议             |

**项目准备**

| 主机 IP    | 主机名        | 角色                |
| ---------- | ------------- | ------------------- |
| 10.0.0.200 | control plane | control plane, etcd |
| 10.0.0.201 | node01        | worker              |
| 10.0.0.202 | node02        | worker+harbor       |

**节点要求**

- 所有节点必须都能通过 SSH 访问。
- 所有节点时间同步。
- 所有节点都应使用 sudo/curl/openssl/tar。

配置所有节点ssh互相访问：

```bash
1.三台机器分别执行以下命令
ssh-keygen

2.查看公钥
cat id_rsa.pub

3.把公钥加入到authorized_keys文件
[root@master ~/.ssh]# cat authorized_keys 
ssh-rsa xxx root@master
ssh-rsa xxx root@node01
ssh-rsa xxx root@node02

4.验证
ssh 10.0.0.201 hostname
ssh 10.0.0.202 hostname
```


# 步骤 2：下载 KubeKey

提前准备工作：

```
#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭 selinux
sentenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config

#关闭 swap
swapoff -a  #临时
sed -ri 's/.*swap.*/#&/' /etc/fstab  #永久
```

下载 KubeKey：

```bash
#创建kk目录
mkdir /data && cd /data

#如果您能正常访问 GitHub/Googleapis
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.1 sh -

#如果您访问 GitHub/Googleapis 受限
export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.1 sh -

#为 kk 添加可执行权限：
chmod +x kk
```

# 步骤 3：创建集群

对于多节点安装，您需要通过指定配置文件来创建集群。

## 1. 创建示例配置文件

命令如下：

```bash
./kk create config --with-kubernetes v1.22.10 --with-kubesphere v3.3.0
```

## 2. 编辑配置文件

如果您不更改名称，那么将创建默认文件 config-sample.yaml。编辑文件，以下是多节点集群（具有一个主节点）配置文件的示例。

```bash
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 10.0.0.200, internalAddress: 10.0.0.200, privateKeyPath: "~/.ssh/id_rsa"}
  - {name: node01, address: 10.0.0.201, internalAddress: 10.0.0.201, privateKeyPath: "~/.ssh/id_rsa"}
  - {name: node02, address: 10.0.0.202, internalAddress: 10.0.0.202, privateKeyPath: "~/.ssh/id_rsa"}
  roleGroups:
    etcd:
    - master
    control-plane:
    - master
    worker:
    - node01
    - node02
    registry:
    - node02

network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    type: harbor
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
  
  alerting:
    enabled: true    #告警系统
  auditing:
    enabled: true    #审计日志
  devops:
    enabled: true    #devops
  openpitrix:
    store:
      enabled: true  #应用商店
```

> 备注：
> **roleGroups**
>
> - etcd：etcd 节点名称
> - control-plane：主节点名称
> - worker：工作节点名称   



**使用 SSH 密钥的无密码登录示例：**

```bash
hosts:
  - {name: master, address: 10.0.0.200, internalAddress: 10.0.0.200, privateKeyPath: "~/.ssh/id_rsa"}
```



## 3. 使用配置文件创建集群

**备注：每个节点安装依赖项**

```bash
yum install -y  socat conntrack ebtables ipset
```

**安装集群**

```bash
./kk create cluster -f config-sample.yaml
```

## 4. 验证安装

执行以下命令查看集群状态：

```bash
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

> 报错：提示 -bash: kubectl: command not found ，这里配置环境变量使之生效

```bash
[root@master ~]# whereis is kubectl
[root@master ~]# vim /etc/profile              最后一行加入
export PATH="/usr/local/bin/:$PATH"
[root@master ~]# source /etc/profile
```

安装完成后，您会看到如下内容：

```bash
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://10.0.0.200:30880
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

![image](https://kubesphere.com.cn/images/docs/v3.3/zh-cn/upgrade/air-gapped-upgrade-with-ks-installer/kubesphere-login.PNG)



# 步骤 4：启用 kubectl 自动补全

KubeKey 不会启用 kubectl 自动补全功能，请参见以下内容并将其打开：

>备注：
>
>请确保已安装 bash-autocompletion 并可以正常工作。

```bash
# Install bash-completion
apt-get install bash-completion

# Source the completion script in your ~/.bashrc file
echo 'source <(kubectl completion bash)' >>~/.bashrc

# Add the completion script to the /etc/bash_completion.d directory
kubectl completion bash >/etc/bash_completion.d/kubectl
```

# 步骤 5：安装内网穿透工具 ZeroTier

## 1.注册账号

访问：https://www.zerotier.com/ 注册账号；

## 2.创建虚拟网络

1. 点击“Create A Network”创建一个虚拟网络：
2. 记住Network ID，后边会用到；
   Name随便写；
   Access Control选择：PRIVATE；

![](http://cdn.linuxwf.com/img/20221020115538.png)

## 3.配置网段

1. IPV4 Auto-Assign 选择“Easy”，然后从下发的网段选择一个即可；
2. 这里选择的是10.242.*.*，上方的Managed Routes会自动修改为：10.242.0.0/16 (LAN);
3. 表示：10.242.*.*的虚拟网段，由ZeroTier进行路由转发；

![](http://cdn.linuxwf.com/img/20221020112550.png)

4. 剩下部分保留默认配置即可；接下来配置客户端。

## 4.客户端配置

1. 下载并安装ZeroTier客户端：https://www.zerotier.com/download/

2. 按照官网提示，安装ZeroTier客户端即可；

   ```bash
   curl -s https://install.zerotier.com | sudo bash
   ```

   若需要，可以通过以下命令获取设备的address地址：

   ```bash
   cat /var/lib/zerotier-one/identity.public |cut -d : -f 1
   ```

3. 配置客户端，加入虚拟网络：

   ```bash
   zerotier-cli join <Network ID>
   ```

   MacOS和Windows系统，打开客户端，粘贴Network ID，然后点击“Join Network”：

4. 回到管理UI，找到刚刚申请加入的设备，点击左侧的“Auth?”复选框，给新加入的设备授权：

   >“Name”随便写；
   >“Managed IPs”，输入“10.242.0.0”至“10.242.255.255”之间的任意IP，网段对应上一步创建的网段即可；
   >点击“Managed IPs”左侧的“+”，生成虚拟IP；

![](http://cdn.linuxwf.com/img/20221020115412.png)

5. 客户端配置完成；可以回到客户端，查看是否正常：
6. 将多个设备加入虚拟网络；如果是云服务器，记得开放相应的端口；
7. 多个设备之间，可以使用虚拟IP互联；

## 5.moon节点配置

官方的节点，速度比较慢；可以通过自建中继节点（moon）的方式，提高连接速度；
自建的moon节点服务器必须有外网IP，才有意义；

1. 在Linux的服务器上，配置moon服务器：

```bash
cd /var/lib/zerotier-one
#生成moon
zerotier-idtool initmoon identity.public >>moon.json
```

2. 修改 moon.json文件的id地址、stableEndpoints值

```bash
"id": "12345678",
"stableEndpoints": ["1.2.3.4/9993"]
```

3. 启动moon服务器：

```bash
#生成签名文件
zerotier-idtool genmoon moon.json                #会生成类似：000000xxxxxxxx.moon的文件

#将 moon 节点加入网络
mkdir moons.d
cp 000000xxxxxxxx.moon /moons.d/
systemctl restart zerotier-one.service
```

4. 其他客户端连接moon服务器：

```bash
#Linux
#连接moon节点：
#注意这里的12345678为上边moon.json里的id
#orbit后边跟两个id
zerotier-cli orbit 12345678 12345678
```

5. 如果是Windows系统：

```bash
C:\Program Files (x86)\ZeroTier\One> .\zerotier-cli.bat orbit 12345678 12345678
```

6. 查看节点连接信息：

```bash
zerotier-cli listpeers
#结果中可以看到新加入的moon节点；
#如果没有，可以稍等一会；
```

7. 卸载Zerotier-one

```bash
1.通过dpkg删除zerotier-one服务
dpkg -P zerotier-one

2.删除zerotier-one文件夹，该文件夹存储了address地址，删除后再次安装会获得新的address地址
rm -rf /var/lib/zerotier-one/
```


