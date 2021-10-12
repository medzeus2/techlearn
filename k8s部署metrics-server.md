# k8s部署metrics-server

metrics-server可以方便的对k8s进行度量，配合hpa进行自动扩容。

## 安装

### 方法1

通过helm安装

首先增加bitnami的repo

```
```

安装metrics-server

```bash
helm install metrics-server  bitnami/metrics-server
```

### 方法2

直接通过yml安装

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```



## 修改运行



默认的可能跑不起来，需要进行以下修改。

* 增加beta的命名空间

```
helm upgrade --namespace default metrics-server bitnami/metrics-server --set apiService.create=true
```

* 修改运行参数

```bash
# 在deployment增加下面2行
Command:
      metrics-server
      --kubelet-preferred-address-types=InternalIP
      --kubelet-insecure-tls

```

