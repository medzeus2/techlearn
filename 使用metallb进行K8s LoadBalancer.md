# 使用metallb进行K8s LoadBalancer

## 背景

使用k8s经常需要暴露服务，通过使用nginx ingress，但是默认apiserver 只能使用30000以上端口。

而在云环境一般可以使用外部负载均衡，例如阿里的SLB，那么在自建k8s集群怎么用软件的方法进行外部负载ip配置。

答案是metallb。

## 创建metallb-system命名空间

```bash
 kubectl create namespace metallb-system
```

或者

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.3/manifests/namespace.yaml
# 或者
kubectl apply -f https://ghproxy.com/https://raw.githubusercontent.com/metallb/metallb/v0.10.3/manifests/namespace.yaml
```



## 创建ip地址池

使用metal负载必须首先创建地址池

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 203.0.113.10-203.0.113.15
```

## 修改kube-proxy ARP模式

```bash
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

或者

```bash
kubectl edit configmap -n kube-system kube-proxy
```

```bash
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

注意 strictARP: true

## 安装metallb

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.3/manifests/metallb.yaml
```

## 验证

修改svc为LoadBalancer，看看是不是有外部ip了。
