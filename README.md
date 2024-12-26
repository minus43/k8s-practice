# 쿠버네티스 클러스터 설정 가이드

## 0. 쿠버네티스 설치

### 사전 준비 작업

- 마스터 노드 1대(cpu 4core, memory 2GB, 25GB 디스크)
- 워커 노드 2대(cpu 2core, memory 2GB, 50GB 디스크)
- 네트워크 설정

저는 virtualbox에서 ubuntu 24.04 3대를 사용하고 있습니다.  
네트워크 설정은 어댑터 1은 NAT(호스트 pc의 인터넷 공유), 어댑터 2는 호스트 전용 어댑터로 설정하였습니다.

- 마스터 노드 IP: 192.168.56.2
- 워커 노드 IP: 192.168.56.3, 192.168.56.4
- 네트워크 설정: 192.168.56.0/24

```bash
# 1. 스왑 비활성화
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. 시간 동기화 설정
# 기존 타임 서비스 중지 및 비활성화
sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd

# chrony 설치 및 설정
sudo apt-get update
sudo apt-get install -y chrony
sudo tee /etc/chrony/chrony.conf << EOF
pool ntp.ubuntu.com iburst
makestep 1.0 3
EOF

# chrony 재시작 및 활성화
sudo systemctl restart chronyd
sudo systemctl enable chronyd

# 타임존 설정
sudo timedatectl set-timezone Asia/Seoul

# 시간 동기화 상태 확인
chronyc sources
chronyc tracking

# 3. 커널 모듈 활성화
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 4. 커널 파라미터 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# 5. 방화벽 비활성화
sudo ufw disable
sudo iptables -F
sudo iptables -X

# 6. containerd 설치
sudo apt-get update
sudo apt-get install -y containerd

# containerd 설정
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

# 7. kubeadm, kubelet, kubectl 설치
sudo apt-get update
# apt-transport-https는 더미 패키지일 수 있음; 필요 없을 수 있음
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# /etc/apt/keyrings 디렉토리가 없는 경우 생성 필요, 아래 주석 참고
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# /etc/apt/sources.list.d/kubernetes.list 파일 덮어쓰기
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

이렇게 하고 나서 복제해서 워커 노드를 구성하면 됩니다.

- netplan 설정과 /etc/hosts 설정 필요
- hostname 설정 필요
- ssh 설정, vim 등 에디터 설정 필요

## 1. 클러스터 설치

```bash
# 마스터 노드 초기화(calico cni를 사용하기 때문에 192.168.0.0/16 사용)
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address 192.168.56.2

# kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Calico CNI 설치
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
sudo systemctl restart kubelet

# 클러스터 상태 확인
kubectl get nodes

# 워커 노드 조인
#kubeadm init에서 출력된 명령어(kubeadm join~) 복사해서 워커 노드에서 실행

# 워커 노드 조인 후에는 클러스터 상태를 확인해서 Ready 상태인지 확인
```

## 2. 기본 구성요소 설치

### OpenEBS 설치 (스토리지 클래스)

OpenEBS는 쿠버네티스의 스토리지 솔루션으로, 컨테이너화된 애플리케이션을 위한 영구 저장소를 제공합니다.

- 로컬 디스크를 사용한 스토리지 관리
- 동적 볼륨 프로비저닝 지원
- 데이터 지속성 보장

```bash
# OpenEBS 설치
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml

# 설치 확인
kubectl get pods -n openebs
kubectl get storageclass

# 기본 스토리지 클래스 설정
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Nginx Ingress Controller

Nginx Ingress Controller는 쿠버네티스 클러스터로 들어오는 외부 트래픽을 관리하는 도구입니다.

- HTTP/HTTPS 트래픽 라우팅
- SSL/TLS 종단 처리
- URL 기반 서비스 라우팅

```bash
# 저장소 추가
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 설치
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443

# 설치 확인
kubectl get svc -n ingress-nginx
```

## 3. 애플리케이션 설치

### 쿠버네티스 대시보드

쿠버네티스 클러스터를 웹 UI로 관리할 수 있는 공식 대시보드입니다.

- 클러스터 상태 모니터링
- 리소스 관리 및 문제 해결
- 기본적인 클러스터 운영 기능 제공

```bash
# 대시보드 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# 대시보드 접근을 위한 ServiceAccount 생성
cat <<EOF | kubectl apply -f -
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
EOF

# 토큰 생성
kubectl -n kubernetes-dashboard create token admin-user

# 대시보드 인그레스 설정
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: dashboard.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
EOF
```

### Harbor 레지스트리

Harbor는 컨테이너 이미지를 저장하고 관리하는 엔터프라이즈급 레지스트리입니다.

- 컨테이너 이미지 저장 및 배포
- RBAC 기반 접근 제어
- 이미지 취약점 스캔
- Helm 차트 저장소 기능

```bash
# Harbor helm repo 추가 (없는 경우)
helm repo add harbor https://helm.goharbor.io
helm repo update

# Harbor 설치
helm install harbor harbor/harbor \
  --namespace harbor --create-namespace \
  --set expose.type=ingress \
  --set expose.ingress.hosts.core=harbor.local \
  --set expose.ingress.className=nginx \
  --set persistence.persistentVolumeClaim.registry.size=10Gi \
  --set persistence.persistentVolumeClaim.chartmuseum.size=5Gi \
  --set persistence.persistentVolumeClaim.jobservice.size=1Gi \
  --set persistence.persistentVolumeClaim.database.size=1Gi \
  --set persistence.persistentVolumeClaim.redis.size=1Gi \
  --set persistence.persistentVolumeClaim.trivy.size=5Gi \
  --set harborAdminPassword=Harbor12345

# 설치 상태 확인
kubectl get pods -n harbor
kubectl get ingress -n harbor
kubectl get svc -n harbor

# Harbor 상태 확인
kubectl get pods -n harbor
kubectl get pvc -n harbor    # 스토리지 할당 확인
```

### ArgoCD

ArgoCD는 쿠버네티스를 위한 선언적 GitOps 지속적 배포 도구입니다.

- Git 저장소 기반 애플리케이션 배포
- 자동 동기화 및 상태 모니터링
- 다중 클러스터 관리
- 롤백 및 버전 관리

```bash
# ArgoCD 네임스페이스 생성
kubectl create namespace argocd

# ArgoCD 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD CLI 설치 (선택사항)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# ArgoCD Ingress 설정
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF
```

### Jenkins

Jenkins는 지속적 통합 및 배포(CI/CD)를 위한 자동화 서버입니다.

- 빌드 자동화
- 배포 파이프라인 구성
- 플러그인을 통한 확장성
- 다양한 개발 도구 통합

```bash
# Jenkins 저장소 추가
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Jenkins 설치 (기본 설정)
helm install jenkins jenkins/jenkins \
  --namespace jenkins --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=5Gi

# Jenkins Ingress 설정
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: jenkins.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
EOF

# 초기 비밀번호 확인
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password
```

### Prometheus & Grafana

#### Prometheus

시계열 데이터베이스 기반의 모니터링 시스템입니다.

- 메트릭 수집 및 저장
- 알림 규칙 설정
- 강력한 쿼리 언어(PromQL)
- 서비스 디스커버리

```bash
# Prometheus helm repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Prometheus Stack 설치 (Grafana 포함)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.resources.requests.cpu=200m \
  --set prometheus.prometheusSpec.resources.requests.memory=512Mi \
  --set prometheus.prometheusSpec.resources.limits.cpu=500m \
  --set prometheus.prometheusSpec.resources.limits.memory=1Gi \
  --set grafana.resources.requests.cpu=100m \
  --set grafana.resources.requests.memory=256Mi \
  --set grafana.resources.limits.cpu=200m \
  --set grafana.resources.limits.memory=512Mi \
  --set alertmanager.resources.requests.cpu=50m \
  --set alertmanager.resources.requests.memory=128Mi \
  --set prometheusOperator.resources.requests.cpu=100m \
  --set prometheusOperator.resources.requests.memory=128Mi

# 설치 상태 확인
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Prometheus & Grafana Ingress 설정
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: prometheus.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
EOF

# 인그레스 상태 확인
kubectl get ingress -n monitoring
```

## 4. DNS 설정 및 접속 정보

### DNS 설정

```bash
# /etc/hosts 파일에 추가
# 워커노드 IP로 설정 (kubectl get pods -n ingress-nginx -o wide로 확인)
192.168.56.3 dashboard.local
192.168.56.3 harbor.local
192.168.56.3 notary.local
192.168.56.3 argocd.local
192.168.56.3 jenkins.local
192.168.56.3 prometheus.local
192.168.56.3 grafana.local

# 또는 다른 워커노드 사용 시
# 192.168.56.4 dashboard.local
# 192.168.56.4 harbor.local
# 192.168.56.4 notary.local
# 192.168.56.4 argocd.local
# 192.168.56.4 jenkins.local
```

### 서비스 접속 정보

1. 쿠버네티스 대시보드

- URL: https://dashboard.local:30443
- 인증: kubectl -n kubernetes-dashboard create token admin-user

2. Harbor 레지스트리

- URL: http://harbor.local:30443
- ID: admin
- PW: Harbor12345

3. ArgoCD

- URL: https://argocd.local:30443
- ID: admin
- PW: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

4. Jenkins

- URL: https://jenkins.local:30443
- ID: admin
- PW: kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password

5. Prometheus

- URL: http://prometheus.local:30080
- 인증: 없음

6. Grafana

- URL: http://grafana.local:30080
- ID: admin
- PW: prom-operator

## 5. 구성요소 삭제 가이드

### 1. Nginx Ingress Controller 삭제

```bash
# Nginx Ingress 삭제
helm uninstall nginx-ingress -n ingress-nginx
# 네임스페이스 삭제
kubectl delete namespace ingress-nginx
```

### 2. 쿠버네티스 대시보드 삭제

```bash
# 대시보드 삭제
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
# ServiceAccount 및 ClusterRoleBinding 삭제
kubectl delete serviceaccount admin-user -n kubernetes-dashboard
kubectl delete clusterrolebinding admin-user
# 네임스페이스 삭제
kubectl delete namespace kubernetes-dashboard
```

### 3. Harbor 레지스트리 삭제

```bash
# Harbor 삭제
helm uninstall harbor -n harbor
# PVC 삭제 (데이터 삭제 주의)
kubectl delete pvc --all -n harbor
# 네임스페이스 삭제
kubectl delete namespace harbor
```

### 4. ArgoCD 삭제

```bash
# ArgoCD 삭제
kubectl delete -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd
# 네임스페이스 삭제
kubectl delete namespace argocd
```

### 5. Jenkins 삭제

```bash
# Helm 릴리스 삭제
helm uninstall jenkins -n jenkins
# PVC 삭제 (데이터 삭제 주의)
kubectl delete pvc --all -n jenkins
# 네임스페이스 삭제
kubectl delete namespace jenkins
```

### 6. OpenEBS 삭제

```bash
# OpenEBS 삭제 (주의: 모든 PV 데이터가 삭제됨)
kubectl delete -f https://openebs.github.io/charts/openebs-operator.yaml
# 스토리지클래스 삭제
kubectl delete storageclass openebs-hostpath
# 네임스페이스 삭제
kubectl delete namespace openebs
```

### 7. Prometheus & Grafana 삭제

```bash
# Prometheus Stack 삭제
helm uninstall prometheus -n monitoring

# PVC 삭제 (데이터 삭제 주의)
kubectl delete pvc --all -n monitoring

# 네임스페이스 삭제
kubectl delete namespace monitoring
```

### 8. 전체 클러스터 초기화

```bash
# 마스터 노드에서
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d/*
sudo rm -rf $HOME/.kube/config

# 워커 노드에서
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d/*
sudo rm -rf /etc/kubernetes/*
```

주의사항:

1. 삭제 전 필요한 데이터는 반드시 백업
2. PVC/PV 삭제 시 데이터가 영구적으로 삭제됨
3. 순서대로 삭제하는 것이 안전함 (의존성 있는 리소스 먼저 삭제)
