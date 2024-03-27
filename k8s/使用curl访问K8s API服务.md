### Kubernetes API 概念
Kubernetes API 是通过 HTTP 提供的基于资源 (RESTful) 的编程接口。 它支持通过标准 HTTP 动词（POST、PUT、PATCH、DELETE、GET）检索、创建、更新和删除主要资源。

#### Kubernetes RESTful术语
Kubernetes 通常使用常见的 RESTful 术语来描述 API 概念：
- 资源类型（Resource Type） 是 URL 中使用的名称（pods、namespaces、services）
- 所有资源类型都有一个具体的表示（它们的对象模式），称为 类别（Kind）
- 资源类型的实例的列表称为 集合（Collection）
- 资源类型的单个实例称为 资源（Resource），通常也表示一个 对象（Object）
- 对于某些资源类型，API 包含一个或多个 子资源（sub-resources），这些子资源表示为资源下的 URI 路径

#### API 动词
所有对象资源类型都支持标准 HTTP 动词 - GET、POST、PUT、PATCH 和 DELETE。 Kubernetes 也使用自己的动词，这些动词通常写成小写，以区别于 HTTP 动词。
Kubernetes使用术语list来描述返回资源集合，以区别于通常称为 get 的单个资源检索。 如果你发送带有 ?watch 查询参数的 HTTP GET 请求， Kubernetes 将其称为 watch 而不是 get。
对于PUT请求，Kubernetes在内部根据现有对象的状态将它们分类为create 或 update。 update 不同于 patch；patch 的 HTTP 动词是 PATCH。

#### 资源 URI
例如:
- /api/v1/namespaces
- /api/v1/pods
- /api/v1/namespaces/my-namespace/pods
- /apis/apps/v1/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments/my-deployment

你还可以访问资源集合（例如：列出所有 Node）。以下路径用于检索集合和资源：

- 集群作用域的资源：
    - GET /apis/GROUP/VERSION/RESOURCETYPE - 返回指定资源类型的资源的集合
    - GET /apis/GROUP/VERSION/RESOURCETYPE/NAME - 返回指定资源类型下名称为 NAME 的资源
- 名字空间作用域的资源：
    - GET /apis/GROUP/VERSION/RESOURCETYPE - 返回所有名字空间中指定资源类型的全部实例的集合
    - GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE - 返回名字空间 NAMESPACE 内给定资源类型的全部实例的集合
    - GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME - 返回名字空间 NAMESPACE 中给定资源类型的名称为 NAME 的实例
由于名字空间本身是一个集群作用域的资源类型，你可以通过 GET /api/v1/namespaces/ 检视所有名字空间的列表（“集合”），使用 GET /api/v1/namespaces/NAME 查看特定名字空间的详细信息。
集群作用域的子资源：GET /apis/GROUP/VERSION/RESOURCETYPE/NAME/SUBRESOURCE
名字空间作用域的子资源：GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME/SUBRESOURCE

每个子资源所支持的动词有所不同, 参见 K8s API文档以了解更多信息。

### 如何通过 curl 访问 k8s-api-server
下面解释如何通过 curl 访问 k8s-api-server。使用service-account-token来进行 k8s-api-server 的身份验证。
kubectl是读取kubeconfig中的key和cert来通过k8s的认证，从而调用k8s api。
所以我们可以使用curl+token来实现相同的功能。

#### 获得serviceaccounts的token
![](../picture/2.jpg)
PS：一般来说，serviceaccounts的name和你使用的登录名字一样，我的设备上的serviceaccount是cranuser1.

如下命令可以获得cranuser1对应的secrets
```
oc get secrets | grep cranuser1 | grep kubernetes.io/service-account-token
```
![](../picture/3.jpg)

使用 base64 -d 解码 base64 编码的token。
```
oc get secrets cranuser1-token-ftnqm -o jsonpath="{.data.token}" | base64 --decode
```
![](../picture/4.jpg)

这样就获得了cranuser1的token

#### curl发送https请求获取k8s资源
当使用 HTTP 客户端进行承载token身份验证时，K8s API server期望具有值为 Bearer <token> 的 Authorization 标头。
所以，如果想用curl获得configmap列表，命令如下：
```
curl -ks --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ikg0MjQtenZmdkZlYXBsdEUwcGlqUUNKNTEwSUlKbDRvM3ZaMEc4WVdENWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJjcmFuMSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjcmFudXNlcjEtdG9rZW4tZnRucW0iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiY3JhbnVzZXIxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMDBhNTA4OTEtYTgxMC00OWQ2LWFjZTItMDU1OGU3ZDViZTI5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNyYW4xOmNyYW51c2VyMSJ9.DOq4_Ih_6iy8IMooW9nNf-yzncyqzggcrYW4PduffxO_A8_YDGyRkUunLAVeZwKpF3rh6ZYhPA0OlqSL3UFKpLIsp4LKdBiKtsv4v8WTkVmORbRklQKGu_QKDJDMP9XDZe545RWt1Kk8VtlsCt0OP2i356Diyv2i2ZXlubM0pSmeDoRGJjULK69urs4g7kIFt2ud4zGJouusIZAB2XqmtPblU2l_IHj95H5vrMNGARPxeh5bYbwDzkGr-tP_c214J5U9zhTDxtnojBLeKMj52IRX6o0HFN7-s9C5u7XG8NOsuftF8PxUt8d-KswXPPTtHAiHwbjV92PYUgIroToiFMO4CcGqDyUaHWcTMxy4oGqcTpDwbsGreF3ucaVcIZLlQLdF-p7ckdM7O1VmNWdb2e-ZaSX8ne_i_gjZy_CilONE1l87E91tJK-08vsSMkmZYo0ulSkPDbvFBL3Dw01Ns-t5eMsCy7KffmizhGwSuhm7IJhZFfLX7y927BiuL72pC62WR1fQya5PR_TUR3mUSB9fZl3nBBAvf5t_F_busSSj0v9YQvAfGnp-pIn9L2XLTZTw3SOu9MHPrKnUHnsSfXyH91s_z5004rMTatOvhBO30y-8vfnT0urCsDB57slnisbk_l1yc-NGqUvdVCDIacQyPgpWOHBKHMLryW72uMs" "https://api.hztt-ecp-10-70-30-210.ocp.hz.nsn-rdnet.net:6443/api/v1/namespaces/cran1/configmaps?limit=1"
```

如果想获得pod列表，命令如下：
```
curl -ks --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Ikg0MjQtenZmdkZlYXBsdEUwcGlqUUNKNTEwSUlKbDRvM3ZaMEc4WVdENWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJjcmFuMSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjcmFudXNlcjEtdG9rZW4tZnRucW0iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiY3JhbnVzZXIxIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMDBhNTA4OTEtYTgxMC00OWQ2LWFjZTItMDU1OGU3ZDViZTI5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNyYW4xOmNyYW51c2VyMSJ9.DOq4_Ih_6iy8IMooW9nNf-yzncyqzggcrYW4PduffxO_A8_YDGyRkUunLAVeZwKpF3rh6ZYhPA0OlqSL3UFKpLIsp4LKdBiKtsv4v8WTkVmORbRklQKGu_QKDJDMP9XDZe545RWt1Kk8VtlsCt0OP2i356Diyv2i2ZXlubM0pSmeDoRGJjULK69urs4g7kIFt2ud4zGJouusIZAB2XqmtPblU2l_IHj95H5vrMNGARPxeh5bYbwDzkGr-tP_c214J5U9zhTDxtnojBLeKMj52IRX6o0HFN7-s9C5u7XG8NOsuftF8PxUt8d-KswXPPTtHAiHwbjV92PYUgIroToiFMO4CcGqDyUaHWcTMxy4oGqcTpDwbsGreF3ucaVcIZLlQLdF-p7ckdM7O1VmNWdb2e-ZaSX8ne_i_gjZy_CilONE1l87E91tJK-08vsSMkmZYo0ulSkPDbvFBL3Dw01Ns-t5eMsCy7KffmizhGwSuhm7IJhZFfLX7y927BiuL72pC62WR1fQya5PR_TUR3mUSB9fZl3nBBAvf5t_F_busSSj0v9YQvAfGnp-pIn9L2XLTZTw3SOu9MHPrKnUHnsSfXyH91s_z5004rMTatOvhBO30y-8vfnT0urCsDB57slnisbk_l1yc-NGqUvdVCDIacQyPgpWOHBKHMLryW72uMs" "https://10.70.30.210:6443/api/v1/namespaces/cran1/pods?limit=500"
```
