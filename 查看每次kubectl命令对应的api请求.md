
```
kubectl get pod -v=6
```

kubectl get pod -v=6中的-v=6是指定了日志输出的详细级别。这里的-v选项是kubectl命令中的一个标志，用于设置日志的级别，=6表示设置级别为6，即最高详细级别。

在这种情况下，-v=6会导致kubectl命令输出非常详细的日志信息，包括底层HTTP请求和响应等。
```
[core@master0 ~]$ oc get pod  -v=6
I0325 03:02:00.748260 3540176 loader.go:373] Config loaded from file:  /var/home/core/.kube/config
I0325 03:02:00.773222 3540176 round_trippers.go:553] GET https://api.hztt-ecp-10-70-30-210.ocp.hz.nsn-rdnet.net:6443/api/v1/namespaces/default/pods?limit=500 200 OK in 9 milliseconds
No resources found in default namespace.
```