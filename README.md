# 쿠버네티스 클러스터 설정 가이드

이 프로젝트는 쿠버네티스 클러스터 구축과 주요 애플리케이션 설치에 대한 가이드를 제공합니다.

## 환경 요구사항

### 하드웨어

- 마스터 노드 1대: CPU 4core, Memory 2GB, 25GB 디스크
- 워커 노드 2대: CPU 2core, Memory 2GB, 50GB 디스크

### 소프트웨어

- OS: Ubuntu 24.04
- Kubernetes: v1.32
- Container Runtime: containerd

### 네트워크

- 마스터 노드: 192.168.56.2
- 워커 노드: 192.168.56.3, 192.168.56.4
- 네트워크 대역: 192.168.56.0/24

## 빠른 시작

1. [쿠버네티스 설치](https://github.com/minus43/k8s-practice/wiki/Kubernetes-Installation)
2. [클러스터 초기 설정](https://github.com/minus43/k8s-practice/wiki/Cluster-Setup)
3. [스토리지 설정](https://github.com/minus43/k8s-practice/wiki/OpenEBS-Setup)
4. [인그레스 컨트롤러 설정](https://github.com/minus43/k8s-practice/wiki/Ingress-Controller)

## 주요 애플리케이션

- [쿠버네티스 대시보드](https://github.com/minus43/k8s-practice/wiki/Kubernetes-Dashboard)
- [Harbor 레지스트리](https://github.com/minus43/k8s-practice/wiki/Harbor-Registry)
- [ArgoCD](https://github.com/minus43/k8s-practice/wiki/ArgoCD-Setup)
- [Jenkins](https://github.com/minus43/k8s-practice/wiki/Jenkins-Setup)
- [Prometheus & Grafana](https://github.com/minus43/k8s-practice/wiki/Monitoring-Setup)

## 운영 가이드

- [DNS 설정 및 접속 정보](https://github.com/minus43/k8s-practice/wiki/Access-Guide)
- [구성요소 삭제 가이드](https://github.com/minus43/k8s-practice/wiki/Uninstallation-Guide)
- [트러블슈팅](https://github.com/minus43/k8s-practice/wiki/Troubleshooting)

## 문서 구성

자세한 설치 및 설정 방법은 [Wiki](https://github.com/minus43/k8s-practice/wiki)를 참조하세요.

## 라이선스

이 프로젝트는 MIT 라이선스 하에 배포됩니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.
