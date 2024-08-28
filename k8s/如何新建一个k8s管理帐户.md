1. secrets
2. 创建.kube/config
3. 创建ServiceAccount
4. 通过secrets关联user和ServiceAccount
5. 创建ClusterRole来指定权限
6. 创建ClusterRoleBinding，将ClusterRole指定的权限分配给ServiceAccount和linux user使用

如何通过RABC认证：
- 在pod内部
    - 1. 使用ServicesAccount账号（yaml中指定）
    - 2. 通过copy config文件到container内部，实现user账号使用。
- 在host上
    使用user账号

## linux用户K8s配置文件： .kube/config

.kube目录中的config文件是Kubernetes配置文件，用于存储用户的Kubernetes集群连接信息和管理多个集群的上下文（contexts）。这个文件对开发者和运维人员非常重要，因为它允许用户方便地在多个Kubernetes集群之间切换、配置认证信息、定义命名空间以及设置集群默认参数等。

config文件的关键部分
- clusters:
存储集群信息，包括API服务器的URL、证书和TLS相关设置。每个集群的定义都有一个唯一的名称，用户通过这个名称引用不同的集群。
- contexts:
定义用户、集群和命名空间的组合。一个上下文代表了“在某个集群中使用某个用户访问某个命名空间”的环境。通过上下文，用户可以快速切换工作环境，而不需要手动输入一堆命令。
- users:
存储用户的认证信息，比如证书、token或用户名密码。不同的集群可能需要不同的用户信息，因此这个部分支持多个用户配置。
- current-context:
指定当前正在使用的上下文。当用户执行Kubernetes命令时，kubectl 会使用这个上下文来确定要连接的集群、用户和命名空间。

典型的config文件示例：
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1C......RCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://api.hzdc-ecp-10-110-12-102.ocp.hz.nsn-rdnet.net:6443
  name: hzdc-ecp-10-110-12-102
contexts:
- context:
    cluster: hzdc-ecp-10-110-12-102
    namespace: cran1
    user: cranuser1
  name: cranuser1
current-context: cranuser1
kind: Config
preferences: {}
users:
- name: cranuser1
  user:
    token: eyJhbGciOiJSUzI1NiI......BRsY7_9LfTRvKo7IZcV0RIPGOe4IrX59oIvpY04
```
认证（Authentication）: 通过user字段定义的各种方式（如证书、token、OAuth等）验证用户身份，确保用户能够合法访问Kubernetes API服务器。
授权（Authorization）: 一旦用户身份被验证，RBAC机制决定该用户是否有权限执行某些操作。RBAC通过Role、ClusterRole、RoleBinding和ClusterRoleBinding来管理权限。

server字段是k8s cluser中提供api服务的地址，这个地址不是service IP，而是K8s master node的主接口的真实IP

## ServiceAccount
在Kubernetes中，ServiceAccount与User是两个不同的身份概念，用于不同的目的。ServiceAccount主要用于在集群内的Pod中运行的应用程序或服务，而User通常用于集群外部的管理员或开发者。但是在某些场景下，我们确实需要将ServiceAccount和用户关联起来，以便使集群外部的用户能够以特定的ServiceAccount身份操作Kubernetes集群。

ServiceAccount是Kubernetes中的一种特殊账户，主要用于在Pod中运行的应用程序访问Kubernetes API。每个ServiceAccount都有一个与之关联的Secret，该Secret包含访问API所需的凭据（如token）。

每个命名空间都自带一个默认的ServiceAccount，称为default。
当Pod创建时，如果没有指定ServiceAccount，Kubernetes会自动将其绑定到默认的ServiceAccount。

```
apiVersion: v1
imagePullSecrets:
- name: cranuser1-dockercfg-8xnhn
kind: ServiceAccount
metadata:
  creationTimestamp: "2024-05-14T01:25:45Z"
  name: cranuser1
  namespace: cran1
  resourceVersion: "255183"
  uid: c5ea779f-e3bb-4f4c-a400-cf6b920e7288
secrets:
- name: cranuser1-dockercfg-8xnhn
```
ServiceAccount字段中的secrets和.kube/config文件中的user.token是同一个东西，k8s就是通过secrets将ServiceAccount和user绑定。
通过将ServiceAccount的token导出并配置在用户的kubeconfig中，您可以有效地将ServiceAccount与用户关联起来。

## RBAC的关键概念
- Role: 用于定义某个命名空间内的权限。例如，可以创建一个Role，允许用户在特定命名空间中读取Pod信息。
- ClusterRole: 定义集群范围内的权限。与Role不同，ClusterRole可以跨命名空间使用。
- RoleBinding: 将Role绑定到一个或多个用户、组或服务账户上，限制权限只在特定命名空间中生效。
- ClusterRoleBinding: 将ClusterRole绑定到一个或多个用户、组或服务账户上，使其权限在整个集群范围内生效。


## ClusterRoleBinding

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"name":"cran1-node-crb"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"cran-node-clusterrole"},"subjects":[{"kind":"ServiceAccount","name":"cranuser1","namespace":"cran1"}]}
  creationTimestamp: "2024-05-14T01:25:50Z"
  name: cran1-node-crb
  resourceVersion: "255206"
  uid: dfbc70ff-a787-448a-9914-8b8ec8466a48
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cran-node-clusterrole
subjects:
- kind: ServiceAccount
  name: cranuser1
  namespace: cran1
```
上面的示例ClusterRoleBinding将ServiceAccount cranuser1（也就是同这个ServiceAccount绑定的user）和ClusterRole cran-node-clusterrole关联起来了。

## ClusterRole
上面几步完成后，就可以创建ClusterRole来分配ServiceAccount和user的权限

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2024-05-14T01:25:30Z"
  name: cran-node-clusterrole
  resourceVersion: "255092"
  uid: 727cb1a0-43ad-4330-91a9-0f7b591ea240
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - get
  - watch
  - list
  - create
  - delete
  - update
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
  - get
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - list
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  - events
  verbs:
  - get
  - list
- apiGroups:
  - ""
  resources:
  - pods/portforward
  verbs:
  - create
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  verbs:
  - list
  - get
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  verbs:
  - list
  - get
- apiGroups:
  - metrics.k8s.io
  resources:
  - nodes
  verbs:
  - list
  - get
- apiGroups:
  - performance.openshift.io
  resources:
  - performanceprofiles
  verbs:
  - list
  - get
- apiGroups:
  - node.k8s.io
  resources:
  - runtimeclasses
  verbs:
  - list
  - get
- apiGroups:
  - security.openshift.io
  resources:
  - securitycontextconstraints
  verbs:
  - list
  - get
- apiGroups:
  - config.openshift.io
  resources:
  - clusterversions
  verbs:
  - list
- apiGroups:
  - machineconfiguration.openshift.io
  resources:
  - kubeletconfigs
  verbs:
  - list
  - get
- apiGroups:
  - route.openshift.io
  resources:
  - routes
  verbs:
  - get
- apiGroups:
  - scheduling.k8s.io
  resources:
  - priorityclasses
  verbs:
  - list
  - get
- apiGroups:
  - machineconfiguration.openshift.io
  resources:
  - machineconfigs
  verbs:
  - get
  - list
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - get
  - list
  - create
- apiGroups:
  - apiextensions.k8s.io
  resourceNames:
  - ptpmonitorconfigs.monitoring.nokia.com
  resources:
  - customresourcedefinitions
  verbs:
  - update
  - delete
- apiGroups:
  - monitoring.nokia.com
  resources:
  - ptpmonitorconfigs
  - ptpmonitorconfigs/status
  - ptpmonitorconfigs/finalizers
  verbs:
  - get
  - watch
  - list
  - update
  - patch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  verbs:
  - get
  - list
  - create
- apiGroups:
  - rbac.authorization.k8s.io
  resourceNames:
  - ptp-monitor-custom-resources-operator
  - ptp-monitor-data-source-viewer
  resources:
  - clusterrolebindings
  verbs:
  - update
  - delete
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterroles
  verbs:
  - get
  - list
  - create
- apiGroups:
  - rbac.authorization.k8s.io
  resourceNames:
  - ptp-monitor-custom-resources-operator
  resources:
  - clusterroles
  verbs:
  - update
  - delete
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
- apiGroups:
  - rannic.nokia.com
  resources:
  - rannicnodeconfigs
  verbs:
  - list
  - get
- apiGroups:
  - ""
  resources:
  - pods/log
  verbs:
  - get
```
