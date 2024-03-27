```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: node-a
  namespace: cran1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zttools
  serviceName: node-a-deployment
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: oam@output
      labels:
        app: zttools
    spec:
      containers:
      - image: rcp-docker-testing-virtual.artifactory-espoo2.int.net.nokia.com/nwmgmt/docker-nwmgmttest:8.38.0-8-gd54794cd-21709190228057-3823
        imagePullPolicy: IfNotPresent
        securityContext:
          capabilities:
            add: ["NET_ADMIN"]
        name: node-a
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo hello; sleep 10;done"]
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
            openshift.io/sriov_netdevice_ens43f1: "1"
          requests:
            cpu: 200m
            memory: 256Mi
            openshift.io/sriov_netdevice_ens43f1: "1"
        securityContext:
          privileged: true
          runAsUser: 0
        volumeMounts:
        - mountPath: /tmp/hostfolder
          name: hostfolder
          readOnly: false
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
      - name: hostfolder
        hostPath:
          path: /tmp/zttools/nodea
          type: DirectoryOrCreate
```
