# kubernetes使用命名空间划分租户



## 创建租户使用的角色

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: xxyu
  name: xxyu-admin
rules:
- apiGroups: ["","apps","extension"] # "" 标明 core API 组
  resources: ["*"]
  verbs: ["*"]
```



## 增加用户

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: xxuser
  namespace: xxyu
```



## 角色绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pods
kind: RoleBinding
metadata:
  name: xxyu-bind
  namespace: xxyu
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: xxuser # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role # 此字段必须是 Role 或 ClusterRole
  name: xxyu-admin     # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io
```

## 导出租户配置文件
```bash
export NAMESPACE="xxyu"
export K8S_USER="xxuser"

token=kubectl -n ${NAMESPACE} describe secret $(kubectl -n ${NAMESPACE} get secret | (grep ${K8S_USER} || echo "$_") | awk '{print $1}') | grep token: | awk '{print $2}'\n
cert=kubectl  -n ${NAMESPACE} get secret `kubectl -n ${NAMESPACE} get secret | (grep ${K8S_USER} || echo "$_") | awk '{print $1}'` -o "jsonpath={.data['ca\.crt']}"
```
生成最终配置文件

```yaml
piVersion: v1
clusters:
- cluster:
    certificate-authority-data: $clusert_cert
    server: https://k3s-master01:6443
  name: mycluster

contexts:
- context:
    cluster: mycluster
    namespace: xxyu
    user: xxuser
  name: xxyu

current-context: xxyu
kind: Config
preferences: {}
users:
- name: demo-user
  user:
    token: $token
    client-key-data: $cert
```



