# kubeadm 安装 k8s集群

全文参见

```bash
https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/
```



## 禁用selinux

```shell
apt install selinux-utils
setenforce 0
```



## 修改内核

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```



## 安装docker-ce

```shell
# 删除已有的包
sudo apt-get remove docker docker-engine docker.io containerd runc

# 更新包
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
# 导入证书
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 导入仓库
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
# 安装docker-ce
apt-get install docker-ce

```



## 安装kubeadm

```shell
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 

cat <<EOF | tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial main
EOF


apt install kubelet kubeadm kubectl  -y

systemctl enable --now kubelet
# 控制版本号不变
apt-mark hold kubelet kubeadm kubectl

```



kubeadm join 192.168.254.67:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:7c7ed2c180ff0e25f88746e41933efae18beb7148addfb414673343fbd1fb508





## 初始化集群

```shell
kubeadm config print init-defaults > init-config.yml

生成初始化 配置文件
```

配置文件基本如下

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
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
  advertiseAddress: 192.168.254.67
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  imagePullPolicy: IfNotPresent
  name: master
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.22.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: "172.32.0.0/16"
scheduler: {}

```

修改 imageRepository，及podSubnet 用于安装集群

其他方法

```shell
for i in `kubeadm config images list`; do 
  imageName=${i#k8s.gcr.io/}
  docker pull registry.aliyuncs.com/google_containers/$imageName
  docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi registry.aliyuncs.com/google_containers/$imageName
done;
```



## 初始化启动不了的情况

方案1：修改kubelet为cgroupfs

```shell
vi /etc/sysconfig/kubelet
vi /etc/default/kubelet

改为如下参数
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs
```

方案2：修改docker为systemd

```bash
vim /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
systemctl restart docker.service

```



## 初始化集群

```shell
kubeadm init  --config=init-config.yml

# 完成之后有如下结果
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.159.210:6443 --token ct4248.2egr8dv9k4avqul7 \
    --discovery-token-ca-cert-hash sha256:4ca4f6835e9cd70b43be16b81d8340876dca0e064c6168342c140140d17f449b 
    
    最后的命令需要在node节点中执行，从而加入的k8s集群
```





## 将master用于调度

默认情况下，出于安全原因，你的集群不会在控制平面节点上调度 Pod。 如果你希望能够在控制平面节点上调度 Pod， 例如用于开发的单机 Kubernetes 集群，请运行：

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 增加节点



```shell
kubeadm config print join-defaults > join-confog.yaml

kubeadm join --config=join-defaults.yaml
```



默认情况下，令牌会在24小时后过期。如果要在当前令牌过期后将节点加入集群， 则可以通过在控制平面节点上运行以下命令来创建新令牌：

```bash
kubeadm token create
```

输出类似于以下内容：

```console
5didvk.d09sbcov8ph2amjw
```

如果你没有 `--discovery-token-ca-cert-hash` 的值，则可以通过在控制平面节点上执行以下命令链来获取它：

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

```bash
kubeadm join 192.168.159.210:6443 --token 5didvk.d09sbcov8ph2amjw \
    --discovery-token-ca-cert-hash sha256:4ca4f6835e9cd70b43be16b81d8340876dca0e064c6168342c140140d17f449b 
```





## kubectl智能提示

```shell
kubectl completion bash >/etc/bash_completion.d/kubectl

# 如果 kubectl 有关联的别名，你可以扩展 shell 补全来适配此别名：

echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## 删除集群

使用适当的凭证与控制平面节点通信，运行：

```bash
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
```

在删除节点之前，请重置 `kubeadm` 安装的状态：

```bash
kubeadm reset
```

重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

如果要重置 IPVS 表，则必须运行以下命令：

```bash
ipvsadm -C
```

现在删除节点：

```bash
kubectl delete node <node name>
```

如果你想重新开始，只需运行 `kubeadm init` 或 `kubeadm join` 并加上适当的参数。



## 安装网络插件

### 使用calico插件

具体参见

```url
https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
```



```bash
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml
```

If you are using pod CIDR `192.168.0.0/16`, skip to the next step. If you are using a different pod CIDR with kubeadm, no changes are required - Calico will automatically detect the CIDR based on the running configuration. 

Modify the replica count to the desired number in the `Deployment` named, `calico-typha`

We recommend at least one replica for every 200 nodes, and no more than 20 replicas. In production, we recommend a minimum of three replicas to reduce the impact of rolling upgrades and failures. 

### 部署flannel插件

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

部署前注意修改下 网络跟之前配置一致，backend 可选vxlan 或者 host-gw

```json
net-conf.json: |
    {
      "Network": "172.32.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }

```



## 安装ingress

