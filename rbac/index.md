# Kubernets-Rbac


# 基于角色的权限控制：RBAC

在kubernetes项目中 负责完成授权的（Authorization）工作的记住， 就是RBAC， 基于角色的访问控制（Role-based Access  Control）

三个基本概念

1. **Role**： 角色，它其实是一组规则， 定义了一组对Kubernetes API对象的操作权限
2. **Subject**： 被作用者，即可以是“人”， 也可以是“机器”
3. **RoleBinding**： 定义了“被作用者”和“角色”的绑定关系

### Role

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

## Subject是如何指定的

通过RoleBinding来实现的

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

## User是从哪里来的？

kubernetes 的User ， 只是一个授权系统的系统里的逻辑概念。

1. 通过外部认证服务，比如 keystone
2. 直接给APIServer指定一个用户名、密码文件

RoleBinding对象就可以直接通过名字， 来引用前面定义的Role对象， 从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系

## ClusterRole 和 ClusterRolebind

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

赋予用户所有权限

```yaml
verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

针对某一具体对象进行权限设置

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

这个由 Kubernetes 负责管理的“内置用户”，正是我们前面曾经提到过的：ServiceAccount。

## 定义一个ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

编写RoleBinding的Yaml 文件， 为这个ServiceAccount分配权限

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

创建这个对象

```yaml
$ kubectl create -f svc-account.yaml
$ kubectl create -f role-binding.yaml
$ kubectl create -f role.yaml
```

查看详情

```yaml
$ kubectl get sa -n mynamespace -o yaml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: "409327"
    ...
  secrets:
  - name: example-sa-token-vmfg6
```

