# Model
## azure-aks-ingress-lab-20250619
https://labs.msaez.io/#/courses/cna-full/2c7ffd60-3a9c-11f0-833f-b38345d437ae/deploy-my-app-2024

## 쿠버네티스 Ingress (경로 및 가상 호스트 기반) 실습
- 이 실습은 Azure AKS 클러스터에서 마이크로서비스 외부 접속을 위한 Kubernetes Ingress를 구성하고 배포하는 방법을 다룹니다. 
- Helm을 이용한 NGINX Ingress Controller 설치부터 경로 기반 라우팅, 그리고 가상 호스트 기반 라우팅까지 구현합니다.
- 외부 HTTP/S 트래픽이 내부 서비스로 어떻게 라우팅되는지, 그리고 DNS/hosts 파일 설정의 중요성을 이해합니다.

## 사전 준비
Azure 계정 및 구독, Gitpod 워크스페이스, Spring Boot 애플리케이션 코드

![스크린샷 2025-06-19 110349](https://github.com/user-attachments/assets/f486f7a2-0f06-4954-9995-7c7967251b31)
![스크린샷 2025-06-19 110644](https://github.com/user-attachments/assets/90339631-cdc6-40bd-ba6a-4f30219310a4)
![스크린샷 2025-06-19 111250](https://github.com/user-attachments/assets/3e1bfb93-c19b-4f6d-9298-4c58a68c26d6)
![스크린샷 2025-06-19 115427](https://github.com/user-attachments/assets/f5d2daf0-faa8-4809-a90d-7bef793cead7)
![스크린샷 2025-06-19 115430](https://github.com/user-attachments/assets/65f26941-236e-4986-818a-62fbbb166cd8)
![스크린샷 2025-06-19 115443](https://github.com/user-attachments/assets/6b93be7d-30de-4fa4-b851-6fe9ee00e84a)
![스크린샷 2025-06-19 115620](https://github.com/user-attachments/assets/50c9caec-4090-4209-84da-5a2dcd99232c)
![스크린샷 2025-06-19 115847](https://github.com/user-attachments/assets/8d8f6e30-a9cc-482f-bd81-adbcedcab3dd)
![스크린샷 2025-06-19 115932](https://github.com/user-attachments/assets/ca496824-600e-401f-a60c-19e1fba6fab7)
![스크린샷 2025-06-19 115949](https://github.com/user-attachments/assets/ac105f20-36a4-43e9-bd79-bc2f2ee8963b)
![스크린샷 2025-06-19 115959](https://github.com/user-attachments/assets/b59cf549-379d-4057-8782-946ef28bb222)

---

## 실습 단계별 상세 설명

1. 사전 준비 및 환경 설정
목표: 쿠버네티스 실습 환경을 초기화하고, Azure AKS 클러스터에 접속할 수 있도록 기본 설정을 완료합니다. 이전에 사용했던 리소스들을 정리하여 실습 간의 간섭을 방지합니다.  
```
# 환경 초기화 스크립트 실행 (init.sh 스크립트가 존재한다고 가정)
./init.sh

# Azure 계정 로그인 및 AKS 클러스터 자격 증명 설정
az login --use-device-code
# (프롬프트에 따라 웹 브라우저에서 인증 진행 후 터미널에 '1' 입력 후 Enter)
az aks get-credentials --resource-group a071098-rsrcgrp --name a071098-aks --overwrite

# 이전 실습에서 생성되었을 수 있는 Kubernetes 리소스 정리
kubectl delete deploy,svc,hpa --all 2>/dev/null || true
kubectl delete statefulset my-kafka 2>/dev/null || true
kubectl delete pod my-kafka-client 2>/dev/null || true
kubectl delete pvc data-my-kafka-0 2>/dev/null || true
kubectl delete pod siege 2>/dev/null || true

# Nginx Ingress Controller 관련 네임스페이스 및 Helm 릴리스 삭제 (이전에 설치된 경우)
helm uninstall ingress-nginx -n ingress-basic 2>/dev/null || true
kubectl delete namespace ingress-basic 2>/dev/null || true

# 정리 후 현재 클러스터 상태 확인
kubectl get all
```
2. 애플리케이션 Deployment 및 Service 배포
목표: Ingress를 통해 외부로 노출할 마이크로서비스들 (order, delivery, product)을 ClusterIP 타입의 Service로 클러스터 내부에 배포합니다.  
```
# order 서비스 Deployment 및 ClusterIP Service 배포
kubectl create deploy order --image=ghcr.io/acmexii/order-liveness:latest
kubectl expose deploy order --type=ClusterIP --port=8080 --target-port=8080

# delivery 서비스 Deployment 및 ClusterIP Service 배포 (이미지 필요 시 대체)
kubectl create deploy delivery --image=ghcr.io/acmexii/delivery-liveness:latest
kubectl expose deploy delivery --type=ClusterIP --port=8080 --target-port=8080

# product 서비스 Deployment 및 ClusterIP Service 배포 (이미지 필요 시 대체)
kubectl create deploy product --image=ghcr.io/acmexii/product-liveness:latest
kubectl expose deploy product --type=ClusterIP --port=8080 --target-port=8080

# 배포된 서비스 및 파드 상태 확인
kubectl get all
```
3. Helm 및 NGINX Ingress Controller 설치
목표: Helm 패키지 매니저를 사용하여 Kubernetes 클러스터에 NGINX Ingress Controller를 설치합니다.  
```
# Helm Repository 추가 및 업데이트
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Ingress Controller를 위한 네임스페이스 'ingress-basic' 생성
kubectl create namespace ingress-basic 2>/dev/null || true

# Helm이 설치되어 있지 않다면 설치 스크립트 실행
if ! command -v helm &> /dev/null
then
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
    chmod 700 get_helm.sh
    ./get_helm.sh
fi

# Azure AKS에 Nginx Ingress Controller 배포
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-basic \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

# Nginx Ingress Controller 배포 상태 확인
kubectl get all -n ingress-basic
```
4. 초기 Ingress 리소스 생성 (경로 기반 라우팅)
목표: path (경로)를 기반으로 서비스를 라우팅하는 ingress.yaml 파일을 생성하고 배포합니다. host 필드를 비워두어 모든 호스트(*)에 대해 동작하도록 설정합니다.  
```
# ingress.yaml 파일 생성 (path-based routing)
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: "Ingress"
metadata:
  name: "shopping-ingress"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: ""
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order
                port:
                  number: 8080
          - path: /deliveries
            pathType: Prefix
            backend:
              service:
                name: delivery
                port:
                  number: 8080
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product
                port:
                  number: 8080
EOF
```
```
# Ingress 리소스 배포
kubectl create -f ingress.yaml

# Ingress의 ADDRESS 할당 대기 (ADDRESS가 할당되고 HOSTS에 '*'가 뜰 때까지 관찰)
kubectl get ingress shopping-ingress -w
```
5. Ingress Controller 공인 IP 확인 및 경로 기반 접속 테스트
목표: Ingress Controller에 할당된 공인 IP 주소(ADDRESS)를 확인하고, 이를 사용하여 경로 기반 라우팅이 정상적으로 동작하는지 테스트합니다.  
```
# Ingress Controller의 ADDRESS 확인
kubectl get ingress

# 할당된 ADDRESS를 통해 경로 기반으로 서비스 접속 테스트
# (위 kubectl get ingress 결과에서 ADDRESS를 확인하여 [INGRESS_ADDRESS]를 대체)
echo "웹 브라우저 또는 curl 명령으로 다음 주소에 접속해보세요:"
echo "http://[INGRESS_ADDRESS]/orders"
echo "http://[INGRESS_ADDRESS]/deliveries"
echo "http://[INGRESS_ADDRESS]/products"
```
6. Ingress 리소스 수정 (가상 호스트 기반 라우팅)
목표: ingress.yaml 파일을 수정하여 가상 호스트(Virtual Host) 기반 라우팅을 정의합니다. 요청의 Host 헤더에 따라 order.service.com은 order 서비스로, delivery.service.com은 delivery 서비스로 라우팅되도록 설정합니다.  
```
# ingress.yaml 파일을 가상 호스트 기반 라우팅 설정으로 수정
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: "Ingress"
metadata:
  name: "shopping-ingress"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: "order.service.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order
                port:
                  number: 8080
    - host: "delivery.service.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: delivery
                port:
                  number: 8080
EOF
```
7. 수정된 Ingress 리소스 적용 및 상태 확인
목표: 수정된 ingress.yaml 파일을 클러스터에 다시 apply하여 shopping-ingress 리소스를 업데이트합니다. 업데이트 후 kubectl get ingress를 통해 HOSTS 컬럼에 새로 정의된 도메인들이 나타나는지 확인합니다.  
```
# 수정된 Ingress 리소스 클러스터에 적용
kubectl apply -f ingress.yaml

# Ingress 리소스 업데이트 상태 확인 (HOSTS 컬럼에 도메인들이 뜰 때까지 관찰)
kubectl get ingress shopping-ingress -w
```
8. 가상 호스트 기반 Ingress 접속 테스트 (로컬 hosts 파일 수정)
목표: 가상 호스트 기반 Ingress에 접속하기 위해 로컬 컴퓨터의 hosts 파일을 수정하여 테스트 도메인 이름이 Ingress Controller의 공인 IP (ADDRESS)로 해석되도록 매핑합니다.  
```
echo "가상 호스트 테스트를 위해 로컬 컴퓨터의 hosts 파일 수정이 필요합니다."
echo "Windows: C:\Windows\System32\drivers\etc\hosts"
echo "Linux/macOS: /etc/hosts"
echo ""
echo "--- hosts 파일에 다음 내용을 추가하고 저장하세요 (IP는 Ingress ADDRESS 사용) ---"
echo "[INGRESS_ADDRESS] order.service.com"
echo "[INGRESS_ADDRESS] delivery.service.com"
echo "------------------------------------------------------------------"
echo ""
echo "hosts 파일 수정 후 웹 브라우저에서 다음 주소로 접속하여 테스트하세요:"
echo "http://order.service.com/"
echo "http://delivery.service.com/"
```
설명: hosts 파일은 로컬 컴퓨터에서 특정 도메인 이름을 IP 주소로 직접 매핑하는 역할을 합니다. 이는 DNS 서버에 실제 도메인을 등록하지 않고도 개발/테스트 목적으로 가상 호스트를 테스트할 때 유용합니다.  
