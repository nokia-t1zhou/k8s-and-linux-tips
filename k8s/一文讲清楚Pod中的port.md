创建docker images时用EXPOSE指定容器将要监听的端口，这仅仅是个声明，没有实际的效果，并不会在真正运行的Pod中打开端口。

#### 通过docker创建的container
要想真正的开放container的端口，需要在docker run命令中加"-p"选项，"-p"的目的是想把docker的一个内部端口映射到host，有2种主要的用法：
```
Flag value	                Description
-p 8080:80	                Map TCP port 80 in the container to port 8080 on the Docker host.
-p 192.168.1.100:8080:80	Map TCP port 80 in the container to port 8080 on the Docker host for connections to host IP 192.168.1.100.
```
不管用哪一种，都会在host上创建一个iptable rule来实现port mapping，如果没有指定IP地址，默认绑定到host上的所有IP。
所以访问container内部的port的方法是： host IP: port, 而不是container IP:port。

#### K8s创建的Pod
YAML中的container.port.containerPort:  仅仅用来声明container开放监听的端口，就算没有声明，外部仍然可以访问Pod中container的所有端口。kubernetes认为Pod不是一个私有网络，所以pod的port是对cluster内部完全开放的，所有port都可以通过pod ip+port来访问，而不需要做任何额外配置
POD YAML中的container.port.hostPort 指定在host上和container.port.containerPort mapping的端口，这个参数声明后，可以通过host IP:hostPort来访问Pod中的container内部的服务（在container.port.hostIP没有声明的情况下，所有host IP都可以用）
POD YAML中的container.port.hostIP需要和container.port.hostPort搭配使用（对应docker run的 "-p" 选项）。

####  Service YAML中的port和targetport关系
Port是和cluster IP一起使用，通过clusterIP:port可以访问到这个service对应的POD的服务
Targetport就是container 中socket监听的端口
当clusterIP为None（headless service），targetport不需要填（就算要填，也必须和port取值相同）headless service并不会在iptable rule中创建任何rule ，所以headless service不是NAT。


