# [실습1] **쿠버네티스 클러스터 구축 실습**

## 1. 공통 진행(master, work1, work2)

### 0) 서버 정보
- 서버 정보
```bash
https://workshop.clush.net:10820/
```
- 계정 정보
```bash
root / clush
```
- ssh 연결
```bash
ssh root@[IP주소]
```
- 핑 테스트
```bash
ping [IP주소]
```
---
### 1) OS 정보 확인
- 코어 확인
```bash
nproc
``` 
```bash
# 출력 결과
2
```
- 메모리 확인
```bash
free -h
```
```bash
# 출력 결과
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       227Mi       3.2Gi       1.0Mi       407Mi       3.4Gi
Swap:          3.8Gi          0B       3.8Gi
```
- MAC 확인
```bash
ifconfig -a
```
- product uuid 확인
```bash
sudo cat /sys/class/dmi/id/product_uuid
```
또는
```bash
sudo cat /sys/class/dmi/product_uuid
```
---
### 2) 메모리 swap 기능 비활성화
- swap 임시 비활성화
```bash
sudo swapoff -a 
```
- swap 영구 비활성화
```bash
sudo sed -i '/swap/s/^/#/' /etc/fstab
```
- 메모리 상태 확인(swap 0 확인)
```bash
sudo free -m
```
```bash
# 출력 결과
               total        used        free      shared  buff/cache   available
Mem:            3911         199        3487           1         224        3488
Swap:              0           0           0
```
- swap 메모리 상태 확인(출력값이 없으면 swap 메모리 비활성화 상태)
```bash
sudo swapon -s
```
- 만일 위에 명령어로 swap 기능 비활성화가 되지 않으면 아래 명령어를 실행
```bash
# swap unit 조회
systemctl list-unit-files --type swap

# swap unit(dev-파티션이름.swap)을 mask해서 비활성화
systemctl mask [swap unit명]
ex) systemctl mask dev-sda3.swap

# 비활성화 적용 확인
sudo systemctl list-unit-files --type swap
```
---
### 3) 방화벽 설정
- 방화벽 비활성화
```bash
sudo ufw disable
```
```bash
# 출력 결과
Firewall stopped and disabled on system startup
```
---
### 4) 네트워크 설정
- /etc/modules-load.d/k8s.conf 파일 생성
```bash
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```
```bash
# 출력 결과
br_netfilter
```
- /etc/sysctl.d/k8s.conf 파일 생성
```bash
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```bash
# 출력 결과
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
- 시스템 재시작 없이 stysctl 파라미터 반영
```bash
sudo sysctl --system
```
- IP 포워딩을 활성화
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```
```bash
# 출력 결과
net.ipv4.ip_forward=1
```
- 변경사항 적용
```bash
sudo sysctl -p
```
```bash
# 출력 결과
net.ipv4.ip_forward = 1
```
- 출력값이 1이면 IP 포워딩이 활성화된 상태
```bash
cat /proc/sys/net/ipv4/ip_forward
```
```bash
# 출력 결과
1
```
---
### 5) 컨테이너 런타임 설치

**(1) Docker 패키지 저장소 추가**
- apt 업데이트
```bash
sudo apt-get update
```
- 필수 패키지 설치
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```
- 공개키 다운로드
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
- 저장소 등록
```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- 저장소 적용을 위한 apt 업데이트
```bash
sudo apt-get update
```
---
**(2) Containerd 패키지 설치**
- containerd 패키지 설치(y 입력하여 진행)
```bash
sudo apt-get install containerd
```
- 설치 확인
```bash
sudo systemctl status containerd
```
```bash
# 출력 결과(active 확인)
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
---
**(3) Containerd Config 옵션 설정**
- containerd 구성 파일 생성
```bash
sudo mkdir -p /etc/containerd
```
- containerd 기본 설정값으로 config.toml 생성
```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
- config.toml 파일 수정
```bash
vi /etc/containerd/config.toml
```
```bash
# cgroup driver(runc) 사용하기 설정
139번째 줄
SystemdCgroup = false 를 true로 변경

> :set nu
> 139 + shift + g 
> i 입력하여 수정
> :wq 로 저장 후 종료
```
- 수정사항 적용 및 재실행
```bash
sudo systemctl restart containerd
```
- 상태 확인
```bash
sudo systemctl status containerd
```
---
### 6) **kubeadm 설치**
- keyrings를 담을 폴더 생성
```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```
- 공개키 다운로드
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
- 저장소 등록
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
- apt 업데이트
```bash
sudo apt-get update
```
- 패키지 설치
```bash
sudo apt-get install -y kubelet kubeadm kubectl
```
- 업데이트 hold 설정
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
- kubelet 바로 실행
```bash
sudo systemctl enable --now kubelet
```
- 설치 확인
```bash
systemctl status kubelet
```
---
## 2. 마스터 노드 진행
- 쿠버네티스 클러스터를 초기화하여 새로운 마스터 노드를 생성하는 명령어(Calico)
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
```bash
# 출력 결과
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
- .kube 폴더 생성
```bash
mkdir -p $HOME/.kube
```
- config 생성
```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
- 권한 변경
```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Calico CNI 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml
```
---
## 3. 워커 노드 진행
- 복사해둔 내용 붙여넣기
```bash
kubeadm join 192.168.1.18:6443 --token xxxxx \
        --discovery-token-ca-cert-hash sha256:xxxxx
```
---
## 4. 클러스터 정상 작동 확인
- node 확인
```bash
kubectl get node -o wide
```
- alias 등록
```bash
alias k="kubectl"
```
- alias 확인
```bash
alias k
```
---
# [실습2] 웹 애플리케이션 배포 **실습**

## 1. 2048 게임 배포

### 1) Deployment 생성
- deployment-2048.yaml 파일 생성
```bash
vi deployment-2048.yaml
```
- 파일 내용 기입
    - 🔽 deployment-2048.yaml 파일
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: deployment-2048
    spec:
    replicas: 2   # pod 2개 배포
    selector:
        matchLabels:
        app.kubernetes.io/name: app-2048
    template:
        metadata:
        labels:
            app.kubernetes.io/name: app-2048
        spec:
        containers:
            - name: app-2048
            image: alexwhen/docker-2048
            ports:
                - containerPort: 80
    ```
    ```bash
    > i 입력하여 파일내용 기입
    > :wq 입력하여 저장 후 종료
    ```
- Deployment 배포
```bash
kubectl apply -f deployment-2048.yaml
```
---
### 2) Service 생성
- service-2048.yaml 파일 생성
```bash
vi service-2048.yaml
```
- 파일 내용 기입
    - 🔽 service-2048.yaml 파일
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: deployment-2048
    spec:
    selector:
        app.kubernetes.io/name: app-2048
    ports:
        - protocol: TCP
        port: 80
        targetPort: 80
    type: ClusterIP
    ```
    ```bash
    > i 입력하여 파일내용 기입
    > :wq 입력하여 저장 후 종료
    ```
- Service 배포
```bash
kubectl apply -f service-2048.yaml
```
---
### 3) 상태 확인하기
- 모든 네임스페이스의 리소스(Pod, Service 등) 전체 상태 확인
```bash
kubectl get all -A
```
- 서비스 확인
```bash
kubectl get svc
```
- Pod 확인
```bash
kubectl get po -o wide
```
- 상세정보 확인
```bash
kubectl describe pod/[pod명]
```
---
## 2.  쿠버네티스 대시보드 배포

### 1) 서비스 설치
- 서비스 설치
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
- 정상적으로 서비스가 만들어졌는지 확인
```bash
kubectl get svc -n kubernetes-dashboard
```
---
### 2) NodePort 설정
- 외부 접속을 위해 NodePort 설정
```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```
- 파일 수정
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: "2023-12-26T07:55:00Z"
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "496361"
  uid: 227af817-5a33-4ce8-a3dd-adb43030d376
spec:
  clusterIP: 10.97.19.146
  clusterIPs:
  - 10.97.19.146
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31000 # <<<<<<<<<<<< 수정
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort # <<<<<<<<<<<< 수정
status:
  loadBalancer: {}
```
- 변경사항 확인
```bash
kubectl get svc -n kubernetes-dashboard
```
- 예시 링크
```
https://192.168.0.11:31000/#login
```
---
### 3) 관리자 계정 생성
- 설치 파일 생성
```bash
vi dashboard-admin.yaml
```
- dashboard-admin.yaml 입력
    - 🔽 dashboard-admin.yaml 파일
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
    name: admin-user
    namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    name: admin-user
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
    subjects:
    - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
    ```
- 설치
```bash
kubectl apply -f dashboard-admin.yaml
```
- 만들어진 대시보드 토큰 값 생성
```bash
kubectl -n kubernetes-dashboard create token admin-user
```
- 쿠버네티스 대시보드 실행
```bash
kubectl proxy
```
---
### 4) Metrics Server 설치
- Metrics Server 설치
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- Metrics Server 상태 확인
```bash
kubectl get deployment metrics-server -n kube-system
```
- Metrics Server가 작동하지 않을 때의 일반적인 해결책 (오류상황 발생시 시도)
    - 로그 확인
    ```bash
    kubectl logs -l k8s-app=metrics-server -n kube-system
    ```
    - ClusterRoleBinding 설정
    ```bash
    kubectl create clusterrolebinding metrics-server:system:auth-delegator --clusterrole=system:auth-delegator --serviceaccount=kube-system:metrics-server
    ```
    - kubelet 인증 문제 해결
        - Metrics Server Deployment 수정
        ```bash
        kubectl edit deployment metrics-server -n kube-system
        ```
        - 내용 추가
        ```yaml
        spec:
            containers:
            - args:
                - --kubelet-insecure-tls
        ```
    - Metrics Server 상태 확인 후 다시 시도
    ```bash
    kubectl top pod -n auw-ai
    ```
- Custom Resource Definitions (CRDs) 활성화 (선택사항)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/crd.yaml
```
