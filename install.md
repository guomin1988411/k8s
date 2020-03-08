1 环境介绍
	采用一台master节点，两台node节点，配置/etc/hosts 本地解析

master	172.20.0.20
node1	172.20.0.21
mode2	172.20.0.22
docker版本	Engine:
  Version:          19.03.5
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.12
kubeletes 版本	Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2",

master 和node1、node2 配置免密码登陆
[root@master ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:+ax8EXx4CgOAGUkp8HPr9J6zpDgfC5gSY5bduzbESLQ root@master
The key's randomart image is:
+---[RSA 2048]----+
|o.o*..           |
|..=.  .          |
| .+ o  . . .     |
|  oE..  o.+ o    |
|o+..=.  So =     |
|o= + +.  oo      |
|+ . +.o   o.     |
|. .o Ooo ..      |
|  .o=.=oo.       |
+----[SHA256]-----+
[root@master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.20.0.20 master master.test.com
172.20.0.21 node1 node1.test.com
172.20.0.22 node2 node2.test.com
[root@master ~]# ssh-copy-id node1
[root@master ~]# ssh-copy-id node2
[root@master ~]# scp /etc/hosts node1:/etc/
[root@master ~]# scp /etc/hosts node2:/etc/

2 配置yum及安装
2.1 docker 镜像

# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
Sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ee.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]


2.2 kubernetes
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet
ps: 由于官网未开放同步方式, 可能会有索引gpg检查失败的情况, 这时请用 yum install -y --nogpgcheck kubelet kubeadm kubectl 安装

2.3 配置docker镜像加速器

vi /etc/docker/daemon.json
{
        "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","https://registry.docker-cn.com"]
}
{
    "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"]
}
systemctl daemon-reload
systemctl restart docker

3 kubeletes初始化
3.1 kubeletes禁用swap参数
所有节点
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
3.2 初始化
3.2.1 初始化master节点
[root@master ~]# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.17.2 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=Swap

节点手动下载kubeletes依赖镜像方法
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers

3.2.2 配置环境变量
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

3.2.3 检查节点状态
[root@master ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
[root@master ~]# kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   NotReady   master   15m   v1.17.2

1 kubelets 依赖的image列表
[root@master ~]# docker image ls
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
registry.aliyuncs.com/google_containers/kube-proxy                v1.17.2             cba2a99699bd        3 weeks ago         116MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.17.2             41ef50a5f06a        3 weeks ago         171MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.17.2             da5fd66c4068        3 weeks ago         161MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.17.2             f52d4c527ef2        3 weeks ago         94.4MB
registry.aliyuncs.com/google_containers/coredns                   1.6.5               70f311871ae1        3 months ago        41.6MB
registry.aliyuncs.com/google_containers/etcd                      3.4.3-0             303ce5db0e90        3 months ago        288MB
registry.aliyuncs.com/google_containers/pause                     3.1                 da86e6ba6ca1        2 years ago         742kB
[root@master ~]# kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-9d85f5447-2ts9z          0/1     Pending   0          8m40s
coredns-9d85f5447-6kb94          0/1     Pending   0          8m40s
etcd-master                      1/1     Running   0          8m56s
kube-apiserver-master            1/1     Running   0          8m56s
kube-controller-manager-master   1/1     Running   0          8m56s
kube-proxy-lskwl                 1/1     Running   0          8m40s
kube-scheduler-master            1/1     Running   0          8m56s
[root@master ~]# kubectl get pods -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
coredns-9d85f5447-2ts9z          0/1     Pending   0          8m58s   <none>        <none>   <none>           <none>
coredns-9d85f5447-6kb94          0/1     Pending   0          8m58s   <none>        <none>   <none>           <none>
etcd-master                      1/1     Running   0          9m14s   172.20.0.20   master   <none>           <none>
kube-apiserver-master            1/1     Running   0          9m14s   172.20.0.20   master   <none>           <none>
kube-controller-manager-master   1/1     Running   0          9m14s   172.20.0.20   master   <none>           <none>
kube-proxy-lskwl                 1/1     Running   0          8m58s   172.20.0.20   master   <none>           <none>
kube-scheduler-master            1/1     Running   0          9m14s   172.20.0.20   master   <none>           <none>

2 节点状态NotReady原因及解决方法
备注： 这里status NotReady的原因是缺少flannel 插件，安装方法如下
Kubernetes v1.7+ kubectl
apply –f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
网站无法访问的话 https://github.com/coreos/flannel
https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml 下载下来，用这个文件安装
[root@master ~]# kubectl  create  -f kube-flannel.yaml
[root@master ~]# kubectl  create  -f kube-flannel.yaml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created

flannel镜像下载很慢，可以手动下载
[root@master ~]# docker image pull quay.io/coreos/flannel:v0.11.0-amd64
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   32m   v1.17.2
[root@master ~]# systemctl is-active kubelet.service
active

3.2.4 节点加入master
kubeadm join 172.20.0.20:6443 --token q1yyi5.128orqobeye031e2     --discovery-token-ca-cert-hash sha256:0d2584eba67c67e46178a2847f04c4918c0501e8970fbccb80aa906bd4344bd6 --ignore-preflight-errors=Swap
[root@master ~]# kubectl  get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   Ready      master   36m   v1.17.2
node1    NotReady   <none>   60s   v1.17.2
node2    NotReady   <none>   21s   v1.17.2
[root@master ~]# kubectl  get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   53m   v1.17.2
node1    Ready    <none>   17m   v1.17.2
node2    Ready    <none>   16m   v1.17.2




[root@docker-master ~]# kubectl exec -it pod-cm-3 -- /bin/sh
/ # cd /etc/nginx/conf.d/
/etc/nginx/conf.d # cat www.conf
server {

        server_name myapp.test.com;
        listen 80;
        root /data/web/html;

 }
/etc/nginx/conf.d # nginx -T
# configuration file /etc/nginx/conf.d/www.conf:
server {

        server_name myapp.test.com;
        listen 80;
        root /data/web/html;

 }
/etc/nginx/conf.d # mkdir -p /data/web/html
/etc/nginx/conf.d # vi /data/web/html/index.html
[root@docker-master volumes]# kubectl get pods -o wide | grep cm-3
pod-cm-3                            1/1     Running   0          30m     10.244.1.9   node2   <none>           <none>
[root@docker-master volumes]# curl 10.244.1.9
<h1> Nginx Server configured by CM</h1>
[root@docker-master volumes]# kubectl edit cm nginx-www
            listen 8080;

在pod里面看到 端口已经更改了
/etc/nginx/conf.d # cat www.conf
server {

        server_name myapp.test.com;
        listen 8080;
        root /data/web/html;

 }
看到nginx监听的还是80
/etc/nginx/conf.d # netstat -tupln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1/nginx: master pro
重新reload一下，正常
/etc/nginx/conf.d # nginx  -s reload
2020/02/10 13:54:08 [notice] 32#32: signal process started
/etc/nginx/conf.d # netstat -tupln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      1/nginx: master pro
[root@docker-master volumes]# curl 10.244.1.9:8080
<h1> Nginx Server configured by CM</h1>

[root@docker-master volumes]# kubectl create secret generic mysql-root-password --from-literal=password=MyP@ss123
secret/mysql-root-password created
[root@docker-master volumes]# kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-jfssz   kubernetes.io/service-account-token   3      5d10h
mysql-root-password   Opaque                                1      10s
[root@docker-master volumes]# kubectl describe  secrets mysql-root-password
Name:         mysql-root-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  9 bytes
[root@docker-master volumes]# kubectl get secrets  mysql-root-password  -o yaml
apiVersion: v1
data:
  password: TXlQQHNzMTIz
kind: Secret
metadata:
  creationTimestamp: "2020-02-10T14:25:07Z"
  name: mysql-root-password
  namespace: default
  resourceVersion: "1130074"
  selfLink: /api/v1/namespaces/default/secrets/mysql-root-password
  uid: c2f8bafa-bb01-4da1-86a7-c692760b0e5e
type: Opaque
[root@docker-master volumes]# echo TXlQQHNzMTIz | base64 -d
MyP@ss123
[root@docker-master volumes]#

环境变量传递方式
[root@docker-master configmap]# kubectl apply -f pod-secret.yaml
pod/pod-secret-1 created
[root@docker-master configmap]# kubectl exec pod-secret-1 -- printenv | grep -i mysql
MYSQL_ROOT_PASSWORD=MyP@ss123
[root@docker-master configmap]# cat pod-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-1
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations:
    magedu.com/created-by: "cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      containerPort: 80
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-root-password
          key: password




cd /etc/kubernetes/pki/
[root@docker-master pki]# (umask 077; openssl genrsa -out dashboard.key 2048)
Generating RSA private key, 2048 bit long modulus
.............................................................................+++
..............................................+++
[root@docker-master pki]# openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=magedu/CN=dashboard"
[root@docker-master pki]# openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 3650
Signature ok
subject=/O=magedu/CN=dashboard
Getting CA Private Key
[root@docker-master pki]# kubectl create secret generic dashboard-cert -n kube-system --from-file=dashboard.crt=./dashboard.crt --from-file=dashboard.key=./dashboard.key
secret/dashboard-cert created
[root@docker-master pki]# kubectl get secrets -n kube-system | grep dashboard
dashboard-cert                                   Opaque                                2      26s


kubenetes token查看方法
[root@docker-master ~]# kubectl -n kubernetes-dashboard get secret
NAME                               TYPE                                  DATA   AGE
default-token-c4pqd                kubernetes.io/service-account-token   3      27m
kubernetes-dashboard-certs         Opaque                                0      27m
kubernetes-dashboard-csrf          Opaque                                1      27m
kubernetes-dashboard-key-holder    Opaque                                2      27m
kubernetes-dashboard-token-b442w   kubernetes.io/service-account-token   3      27m
[root@docker-master ~]# kubectl -n kubernetes-dashboard describe secrets kubernetes-dashboard-token-b442w
Name:         kubernetes-dashboard-token-b442w
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: 1ed5539b-227f-47cf-a339-f018aa30a83c

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImtuLVRDWXpSU2Z3UGM3bzFNQmtkY2tNa1hMWWtfa2UyQUFLQjBabFd1eGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1iNDQydyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjFlZDU1MzliLTIyN2YtNDdjZi1hMzM5LWYwMThhYTMwYTgzYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.rFeetj78SVawkEHHYCwgIdlYWkMYPGMahwYt2dS7qOe3AuK5-bqkyaJe9Z3CLE19xXCvlUd5T_taWNMaYPIRDKJEQbz93OpPtKjfR6jv3tfVHxBMzI8oJvnqI9zqvg4MmHd0AjifybmFFlwDWC_JyzHN3xnho79-FYgNnIc1VUehxsiD5NnIMyS33CTrzV5wODfVEy3lHuiro3W8LkCaMVHrAbE-ogxc0BCZi_OmPKbNNvdUoATi3OeZ3-BiCwUPA1EcZtSyxfig7TtJ9Mgr9J0cPQxNE1VW8YQwORv-n0DjwDySfxWp7eMZUxJ8x0KDm8gz8QoyZ_15qXW-xCTjhQ

4 dashboard安装
4.1 下载yaml
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc5/aio/deploy/recommended.yaml  --no-check-certificate

4.2 更改配置文件yaml
更改service type: NodePort nodePort: 30000
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000
  selector:
    k8s-app: kubernetes-dashboard
注释默认的kubenetes-dashboard-certs
#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs
#  namespace: kubernetes-dashboard
#type: Opaque
修改image 策略IfNotPresent，然后在所有节点docker pull image
          image: kubernetesui/dashboard:v2.0.0-rc5
          imagePullPolicy: IfNotPresent
          image: kubernetesui/metrics-scraper:v1.0.3
          imagePullPolicy: IfNotPresent

4.3 手动创建certs
[root@docker-master ~]# mkdir dashboard-certs
[root@docker-master ~]# cd dashboard-certs/
[root@docker-master dashboard-certs]# openssl genrsa -out dashboard.key 2048
Generating RSA private key, 2048 bit long modulus
..................................................................................................................................................+++
..............+++
e is 65537 (0x10001)
[root@docker-master dashboard-certs]# openssl req -days 36000 -new -out dashboard.csr -key dashboard.key -subj '/CN=dashboard-cert'
[root@docker-master dashboard-certs]# openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt
Signature ok
subject=/CN=dashboard-cert
Getting Private key


4.4 手动创建namespace
[root@docker-master dashboard-certs]# kubectl create namespace kubernetes-dashboard
namespace/kubernetes-dashboard created
[root@docker-master dashboard-certs]# kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kubernetes-dashboard
secret/kubernetes-dashboard-certs created
[root@docker-master dashboard-certs]# kubectl create  -f ../recommended.yaml
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
Error from server (AlreadyExists): error when creating "../recommended.yaml": namespaces "kubernetes-dashboard" already exists
[root@docker-master ~]# kubectl get pods -A  | grep dashboard
kubernetes-dashboard   dashboard-metrics-scraper-7b8b58dc8b-8x7r2   1/1     Running   0          3h4m
kubernetes-dashboard   kubernetes-dashboard-79f5cc5bb6-k5j4x        1/1     Running   0          3h4m
[root@docker-master dashboard-certs]# kubectl get service -A
NAMESPACE              NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                  8d
kube-system            kube-dns                    ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   8d
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.106.106.102   <none>        8000/TCP                 61s
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.107.48.193    <none>        443:30000/TCP


4.5 创建dashboard admin账号
[root@docker-master dashboard-certs]# vi dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: dashboard-admin
  namespace: kubernetes-dashboard
[root@docker-master dashboard-certs]# kubectl create  -f dashboard-admin.yaml
serviceaccount/dashboard-admin created

4.6 分配admin角色
[root@docker-master dashboard-certs]# vi dashboard-admin-bind-cluster-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-bind-cluster-role
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
[root@docker-master dashboard-certs]# kubectl create -f dashboard-admin-bind-cluster-role.yaml
获取admin token令牌
[root@docker-master dashboard-certs]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin | awk '{print $1}')
Name:         dashboard-admin-token-kp9pt
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: e9eded8d-0b78-4246-8e67-f963ee543043

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImtuLVRDWXpSU2Z3UGM3bzFNQmtkY2tNa1hMWWtfa2UyQUFLQjBabFd1eGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4ta3A5cHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZTllZGVkOGQtMGI3OC00MjQ2LThlNjctZjk2M2VlNTQzMDQzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.YzQQfi7-Xuct5eRVnd49E1kVl2tCnFvvdbwbv3kpq0D6uSbRZcA6nTO6egKuUwAqcK6n6NOaJlrs-omeiQ27u_BmLGvvDdQa-jtvvHL3QJxKgN7parH4bCKcILb4Xcp4_mUsCLJpLzBLGUqPqbTfhv0D7Dm_Om2LuQ5_FDQz1Qiq8H1v7GQ9AEGZpFl4f_2MYYhED3PrOsysUumpjqRPyNk8tgsK2lZvBrT_Mt1YbsP7RDKiQhmA_nZ82LiDLEzitVzB028Dw1Os7T-Cvi25lnuvgNDrd5XieELCZcGHIBXW7bSe8Rs0iiouxPcTquaYbMk98C3Lqv_A1NLAZqtXxQ

完毕  访问任意节点的https:x.x.x.x:30000 访问服务
