# [실습1] **쿠버네티스 클러스터 구축 실습**

## 1. 공통 진행(master, work1, work2)

### 0) 서버 정보

```bash
# 서버 정보
master : 192.168.1.18
work1 : 192.168.1.12
work2 : 192.168.1.22

# 계정 정보
root / clush1234

# ssh 연결
ssh root@192.168.1.z18

# 핑 테스트
ping 192.168.1.18
```

### 1) OS 정보 확인

```bash
# 코어 확인
nproc
>> 2

# 메모리 확인
free -h
>>
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       227Mi       3.2Gi       1.0Mi       407Mi       3.4Gi
Swap:          3.8Gi          0B       3.8Gi

# MAC 확인
ifconfig -a

# product uuid 확인
sudo cat /sys/class/dmi/product_uuid
# 또는 
sudo cat /sys/class/dmi/id/product_uuid

master >> 3b8ee11d-2c6e-c444-b32c-512322b467fa
work1 >> 92777eb4-f05a-be44-bdd2-721b88435e91
work2 >> c9dace7d-7e9b-bc4d-aed9-5a9ee1affe39
```

### 2) 메모리 swap 기능 비활성화

```bash
# swap 임시 비활성화
sudo swapoff -a 

# swap 영구 비활성화
sudo sed -i '/swap/s/^/#/' /etc/fstab

# 메모리 상태 확인(Swap 0 확인)
sudo free -m
>>root@master:~# sudo free -m
               total        used        free      shared  buff/cache   available
Mem:            3911         199        3487           1         224        3488
Swap:              0           0           0

# swap 메모리 상태 확인, 출력값이 없으면 swap 메모리 비활성화 상태
sudo swapon -s

# 만일 위에 명령어로 swap 기능 비활성화가 되지 않으면 아래 명령어를 실행
#swap unit 조회
systemctl list-unit-files --type swap

# swap unit(dev-파티션이름.swap)을 mask해서 비활성화
systemctl mask [swap unit명]
ex) systemctl mask dev-sda3.swap

# 비활성화 적용 확인
sudo systemctl list-unit-files --type swap
```

### 3) 방화벽 설정

```bash
# 방화벽 비활성화
sudo ufw disable
>> Firewall stopped and disabled on system startup
```

### 4) 네트워크 설정

```bash
# /etc/modules-load.d/k8s.conf 파일 생성
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
>> br_netfilter

 
# /etc/sysctl.d/k8s.conf 파일 생성
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
>>net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# 시스템 재시작 없이 stysctl 파라미터 반영
sudo sysctl --system

# 
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
>> net.ipv4.ip_forward=1

sudo sysctl -p
>> net.ipv4.ip_forward = 1

cat /proc/sys/net/ipv4/ip_forward
>> 1
```

### 5) 컨테이너 런타임 설치

**(1) Docker 패키지 저장소 추가**

```bash
# apt 업데이트
sudo apt-get update

# 필수 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# 공개키 다운로드
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 저장소 등록
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#저장소 적용을 위한 apt 업데이트
sudo apt-get update
```

**(2) Containerd 패키지 설치**

```bash
#containerd 패키지 설치
sudo apt-get install containerd
>> y 입력하여 진행

#설치 확인
sudo systemctl status containerd
>> active 확인
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-03-18 01:56:12 UTC; 24s ago
       Docs: https://containerd.io
    Process: 3388 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 3390 (containerd)
      Tasks: 8
     Memory: 14.3M
        CPU: 99ms
     CGroup: /system.slice/containerd.service
             └─3390 /usr/bin/containerd
...
```

**(3) Containerd Config 옵션 설정**

```bash
# containerd 구성 파일 생성
sudo mkdir -p /etc/containerd

#containerd 기본 설정값으로 config.toml 생성
sudo containerd config default | sudo tee /etc/containerd/config.toml

#config.toml 파일 수정
vi /etc/containerd/config.toml

> :set nu
> 139 + shift + g 
> i 입력하여 수정
> :wq 로 저장 후 종료

# cgroup driver(runc) 사용하기 설정
139             SystemdCgroup = false
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
   SystemdCgroup = true

#수정사항 적용 및 재실행
sudo systemctl restart containerd
```

### 6) **kubeadm 설치**

```bash
# keyrings를 담을 폴더 생성
sudo mkdir -p -m 755 /etc/apt/keyrings

# 공개키 다운로드
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 저장소 등록
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# apt 업데이트
sudo apt-get update

# 패키지 설치
sudo apt-get install -y kubelet kubeadm kubectl

# 업데이트 hold 설정
sudo apt-mark hold kubelet kubeadm kubectl

# kubelet 바로 실행
sudo systemctl enable --now kubelet

# 설치 확인
systemctl status kubelet
```

---

---

---

## 2. 마스터 노드 진행

```bash
# 쿠버네티스 클러스터를 초기화하여 새로운 마스터 노드를 생성하는 명령어
# Calico
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
>>
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.18:6443 --token xxxxxxx \
        --discovery-token-ca-cert-hash sha256:xxxxxxxx

```

```bash
#
mkdir -p $HOME/.kube

#
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

#
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml
```

---

---

---

## 3. 워커 노드 진행

```bash
# 복사해둔 내용 붙여넣기
kubeadm join 192.168.1.18:6443 --token xxxxx \
        --discovery-token-ca-cert-hash sha256:xxxxx
```

---

---

---

## 4. 클러스터 정상 작동 확인

```bash
# node 확인
kubectl get node -o wide
```

---

# [실습2] 웹 애플리케이션 배포 **실습**

## 1. Nginx 배포

### 1) Deployment 생성

```bash
# nginx-deploy.yaml 파일 생성
touch nginx-deploy.yaml

# 편집
vi nginx-deploy.yaml
> i 입력하여 nginx-deploy.yaml 파일내용 기입

# Deployment 배포
kubectl apply -f nginx-deploy.yaml
>> deployment.apps/nginx created
```

**🔽 nginx-deploy.yaml 파일**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2  # Nginx Pod 2개 배포
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest  # 최신 Nginx 이미지 사용
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
            requests:
              cpu: "250m"
              memory: "128Mi"
```

### 2) Service 생성

```bash
# nginx-service.yaml 파일 생성
touch nginx-service.yaml

# 편집
vi nginx-service.yaml
> i 입력하여 nginx-deploy.yaml 파일내용 기입

# Service 배포
kubectl apply -f nginx-service.yaml
>> service/nginx created
```

**🔽 nginx-service.yaml 파일**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80            # Service 내부 포트
      targetPort: 80      # Pod의 컨테이너 포트
      nodePort: 31234     # 외부에서 접근할 수 있는 포트 (30000~32767 범위)
```

### 3) 상태 확인하기

```bash
#
kubectl get all -A
>>
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
default       pod/nginx-74c9c64d6f-99tg5                     1/1     Running   0          93s
default       pod/nginx-74c9c64d6f-zt7x5                     1/1     Running   0          93s
kube-system   pod/calico-kube-controllers-654b5d4859-xnr6x   1/1     Running   0          105m
kube-system   pod/calico-node-565z6                          1/1     Running   0          103m
kube-system   pod/calico-node-9xkbt                          1/1     Running   0          102m
kube-system   pod/calico-node-kcvhf                          1/1     Running   0          105m
kube-system   pod/coredns-668d6bf9bc-8rbmb                   1/1     Running   0          113m
kube-system   pod/coredns-668d6bf9bc-dcmp8                   1/1     Running   0          113m
kube-system   pod/etcd-master                                1/1     Running   0          113m
kube-system   pod/kube-apiserver-master                      1/1     Running   0          113m
kube-system   pod/kube-controller-manager-master             1/1     Running   0          113m
kube-system   pod/kube-proxy-62p25                           1/1     Running   0          102m
kube-system   pod/kube-proxy-sx7p9                           1/1     Running   0          113m
kube-system   pod/kube-proxy-vxth5                           1/1     Running   0          103m
kube-system   pod/kube-scheduler-master                      1/1     Running   0          113m

NAMESPACE     NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP                  113m
default       service/nginx        NodePort    10.111.246.251   <none>        80:31234/TCP             74s
kube-system   service/kube-dns     ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   113m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node   3         3         3       3            3           kubernetes.io/os=linux   105m
kube-system   daemonset.apps/kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   113m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
default       deployment.apps/nginx                     2/2     2            2           93s
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           105m
kube-system   deployment.apps/coredns                   2/2     2            2           113m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
default       replicaset.apps/nginx-74c9c64d6f                     2         2         2       93s
kube-system   replicaset.apps/calico-kube-controllers-654b5d4859   1         1         1       105m
kube-system   replicaset.apps/coredns-668d6bf9bc                   2         2         2       113m

#
kubectl get  svc -A
>>
NAMESPACE     NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP                  117m
default       nginx        NodePort    10.111.246.251   <none>        80:31234/TCP             5m28s
kube-system   kube-dns     ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   117m

#
kubectl get po -n default
>>
NAME                     READY   STATUS    RESTARTS   AGE
nginx-74c9c64d6f-99tg5   1/1     Running   0          6m1s
nginx-74c9c64d6f-zt7x5   1/1     Running   0          6m1s

#
kubectl describe pod/nginx-74c9c64d6f-99tg5
>>
Name:             nginx-74c9c64d6f-99tg5
Namespace:        default
Priority:         0
Service Account:  default
Node:             work2/192.168.1.22
Start Time:       Tue, 18 Mar 2025 04:17:29 +0000
Labels:           app=nginx
                  pod-template-hash=74c9c64d6f
Annotations:      cni.projectcalico.org/containerID: edb02e048343869a551b927d6b9b7aafc116b4c99b84bd41cd1dbc6930aec276
                  cni.projectcalico.org/podIP: 192.168.123.1/32
                  cni.projectcalico.org/podIPs: 192.168.123.1/32
Status:           Running
IP:               192.168.123.1
IPs:
  IP:           192.168.123.1
Controlled By:  ReplicaSet/nginx-74c9c64d6f
Containers:
  nginx:
    Container ID:   containerd://2c5c763245ca339e7218b38516679d9c24411661ddc46c361330fea7d19785c9
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:706959c57d76560ed77ea37b7e0a203388b5b12c153c4f820449b9d43d292f53
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 18 Mar 2025 04:17:38 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  256Mi
    Requests:
      cpu:        250m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-5n85z (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-5n85z:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  6m56s  default-scheduler  Successfully assigned default/nginx-74c9c64d6f-99tg5 to work2
  Normal  Pulling    6m55s  kubelet            Pulling image "nginx:latest"
  Normal  Pulled     6m47s  kubelet            Successfully pulled image "nginx:latest" in 8.68s (8.68s including waiting). Image size: 72180980 bytes.
  Normal  Created    6m47s  kubelet            Created container: nginx
  Normal  Started    6m47s  kubelet            Started container nginx

```

## 2.  나만의 블로그 만들기

모든 작업은 mysql 네임스페이스에서 진행

### 0) Namespace 생성

```bash
# namespace 조회
kubectl get namespaces

# namespace 생성
kubectl create namespace mysql
```

### 1) PV 와 PVC 생성

```yaml
# wordpress
kind: PersistentVolume
apiVersion: v1
metadata:
  namespace: mysql
  name: pv0001
  labels:
    type: local
spec:
  capacity:
    storage: 25Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data001/pv0001" 
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: mysql
  name: mysql-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi 
---
# mysql
kind: PersistentVolume
apiVersion: v1
metadata:
  namespace: mysql
  name: pv0002
  labels:
    type: local
spec:
  capacity:
    storage: 25Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data001/pv0002"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: mysql
  name: wordpress-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
# PV와 PVC 생성
kubectl apply -f pv.yaml 
```

### 2) **MySQL Root 패스워드 저장**

```bash
kubectl create secret generic mysql-password --from-literal='password=admin' --namespace="mysql"
```

```bash
# 패스워드 목록 확인
kubectl get secrets -n mysql
>>
NAME             TYPE     DATA   AGE
mysql-password   Opaque   1      23h

# 패스워드 확인
kubectl get secret mysql-password -o jsonpath='{.data.password}' -n mysql; echo

# 패스워드 디코딩
kubectl get secret mysql-password -o jsonpath='{.data.password}' -n mysql | base64 --decode; echo
```

### 3) **MySQL Pod 배포**

**🔽 mysql-deployment.yaml 파일**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: mysql
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-password
                  key: password
            - name: MYSQL_DATABASE # 구성할 database명
              value: k8sdb
            - name: MYSQL_USER # database에 권한이 있는 user
              value: k8suser
            - name: MYSQL_ROOT_HOST # 접근 호스트
              value: '%'  
            - name: MYSQL_PASSWORD # database에 권한이 있는 user의 패스워드
              value: admin
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-volumeclaim
```

```bash
# mysql pod 배포
kubectl apply -f mysql-deployment.yaml 
```

### 4) **MySQL Service 생성**

**🔽 mysql-service.yaml 파일**

```yaml
apiVersion: v1

kind: Service
metadata:
  namespace: mysql
  name: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
  selector:
    app: mysql 
```

```bash
# mysql 서비스 생성
kubectl apply -f mysql-service.yaml 
```

### 5) **Wordpress Pod 배포**

**🔽 wordpress-deployment.yaml 파일**

```yaml
apiVersion: apps/v1

kind: Deployment
metadata:
  namespace: mysql
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - image: wordpress
          name: wordpress
          env:
          - name: WORDPRESS_DB_HOST
            value: mysql:3306
          - name: WORDPRESS_DB_NAME
            value: k8sdb
          - name: WORDPRESS_DB_USER
            value: k8suser
          - name: WORDPRESS_DB_PASSWORD
            value: admin
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: wordpress-volumeclaim 
```

```bash
# wordpress pod 배포
kubectl apply -f wordpress-deployment.yaml 
```

### 6) **Wordpress Service 배포**

**🔽 wordpress-service.yaml 파일**

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  labels:
    app: wordpress
  name: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: wordpress
```

```bash
# wordpress service 배포
kubectl apply -f wordpress-service.yaml 
```
