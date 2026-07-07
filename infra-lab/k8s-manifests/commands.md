# kubectl 명령어 레퍼런스

## 클러스터 & 컨텍스트
```
kubectl version --short                              # 클라이언트/서버 버전
kubectl cluster-info                                 # API 서버 주소 확인
kubectl get nodes                                    # 노드 목록
kubectl get nodes -o wide                            # 노드 IP/OS/런타임 포함
kubectl config get-contexts                          # 컨텍스트 목록
kubectl config use-context 컨텍스트명                # 컨텍스트 전환
kubectl config set-context --current --namespace=ns  # 기본 네임스페이스 지정
```

## 리소스 조회
```
kubectl get pods                                     # 파드 목록 (현재 ns)
kubectl get pods -A                                  # 전체 네임스페이스
kubectl get pods -o wide                             # 노드/IP 포함
kubectl get all                                      # 주요 리소스 한눈에
kubectl get pod 파드명 -o yaml                       # 매니페스트 전체 출력
kubectl describe pod 파드명                          # 이벤트 포함 상세 (디버깅 1순위)
kubectl get events --sort-by=.lastTimestamp         # 이벤트 시간순
kubectl get pods --watch                             # 상태 변화 실시간 감시
kubectl get pods -l app=web                          # 레이블 셀렉터 필터
```

## 생성 & 적용 (선언형)
```
kubectl apply -f manifest.yaml                       # 매니페스트 적용 (생성/갱신)
kubectl apply -f ./dir/                              # 디렉터리 전체 적용
kubectl apply -k ./kustomize/                        # Kustomize 적용
kubectl diff -f manifest.yaml                        # 적용 전 변경점 미리보기
kubectl delete -f manifest.yaml                      # 매니페스트로 삭제
```

## 생성 (명령형 / 빠른 실습)
```
kubectl run nginx --image=nginx                      # 파드 하나 실행
kubectl create deployment web --image=nginx --replicas=3
kubectl create namespace dev                         # 네임스페이스 생성
kubectl expose deployment web --port=80 --type=ClusterIP
# --dry-run으로 매니페스트 뼈대 생성 (YAML 학습에 유용)
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
```

## 로그 & 디버깅
```
kubectl logs 파드명                                  # 로그 확인
kubectl logs -f 파드명                               # 실시간 스트리밍
kubectl logs 파드명 -c 컨테이너명                    # 멀티 컨테이너 중 특정 컨테이너
kubectl logs --previous 파드명                       # 재시작 직전 컨테이너 로그
kubectl exec -it 파드명 -- bash                      # 컨테이너 접속
kubectl exec 파드명 -- env                           # 일회성 명령 실행
kubectl port-forward 파드명 8080:80                  # 로컬 → 파드 포트 포워딩
kubectl port-forward svc/web 8080:80                 # 서비스로 포워딩
kubectl cp 파드명:/경로/파일 ./로컬파일              # 파일 복사
kubectl debug -it 파드명 --image=busybox             # 디버그 임시 컨테이너
```

## 스케일 & 롤아웃
```
kubectl scale deployment web --replicas=5            # 복제 수 변경
kubectl set image deployment/web nginx=nginx:1.27    # 이미지 교체 (롤링 업데이트)
kubectl rollout status deployment/web                # 롤아웃 진행 상태
kubectl rollout history deployment/web               # 리비전 이력
kubectl rollout undo deployment/web                  # 직전 버전으로 롤백
kubectl rollout undo deployment/web --to-revision=2  # 특정 리비전으로 롤백
kubectl rollout restart deployment/web               # 파드 순차 재시작
```

## 설정 & 수정
```
kubectl edit deployment web                          # 에디터로 실시간 수정
kubectl label pod 파드명 env=prod                    # 레이블 추가
kubectl annotate pod 파드명 note="설명"              # 어노테이션 추가
kubectl set env deployment/web KEY=VALUE             # 환경변수 설정
kubectl patch deployment web -p '{"spec":{"replicas":4}}'  # 부분 수정
```

## ConfigMap & Secret
```
kubectl create configmap app-config --from-literal=KEY=VALUE
kubectl create configmap app-config --from-file=config.properties
kubectl create secret generic db-secret --from-literal=password=1234
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d  # 값 복호화
```

## 네임스페이스 & RBAC
```
kubectl get namespaces
kubectl get pods -n kube-system                      # 특정 네임스페이스 조회
kubectl create serviceaccount ci-bot                 # 서비스어카운트 생성
kubectl auth can-i create pods                       # 내 권한 확인
kubectl auth can-i list secrets --as=system:serviceaccount:default:ci-bot
kubectl create role reader --verb=get,list --resource=pods
kubectl create rolebinding read-binding --role=reader --serviceaccount=default:ci-bot
```

## 리소스 & 오토스케일
```
kubectl top nodes                                    # 노드 리소스 사용량 (metrics-server 필요)
kubectl top pods                                     # 파드 리소스 사용량
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70
kubectl get hpa                                      # HPA 상태 확인
```

## 삭제 & 정리
```
kubectl delete pod 파드명
kubectl delete deployment web
kubectl delete -f manifest.yaml
kubectl delete pod -l app=web                        # 레이블로 일괄 삭제
kubectl delete namespace dev                         # 네임스페이스째 삭제 (내부 리소스 전부)
kubectl delete pod 파드명 --grace-period=0 --force   # 강제 삭제 (주의)
```

## 스토리지
```
kubectl get pv                                       # PersistentVolume 목록
kubectl get pvc                                      # PersistentVolumeClaim 목록
kubectl get storageclass                             # StorageClass 목록
```

## 로컬 클러스터 (minikube / kind)
```
minikube start --driver=docker                       # 로컬 클러스터 시작
minikube status
minikube dashboard                                   # 웹 대시보드
minikube service web --url                           # 서비스 접근 URL
minikube addons enable ingress                       # Ingress 컨트롤러 활성화
minikube delete                                      # 클러스터 삭제

kind create cluster --name lab                       # kind 클러스터 생성
kind get clusters
kind delete cluster --name lab
```

## Helm
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm install myapp bitnami/nginx                     # 차트 설치
helm install myapp ./mychart -f values.yaml          # 로컬 차트 + values
helm upgrade myapp ./mychart --set replicaCount=3    # 업그레이드
helm rollback myapp 1                                # 이전 리비전으로 롤백
helm list                                            # 릴리스 목록
helm uninstall myapp                                 # 릴리스 제거
helm template ./mychart                              # 렌더링 결과 미리보기 (설치 안 함)
```

## 유용한 출력 포맷
```
kubectl get pods -o json                             # JSON 출력
kubectl get pod 파드명 -o jsonpath='{.status.phase}' # 특정 필드만 추출
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
kubectl explain pod.spec.containers                  # 필드 스펙 설명 (문서 없이 참고)
```
