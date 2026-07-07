# 05 ConfigMap & Secret

설정을 이미지 안에 굽지 말고 **밖으로 분리**한다.  
같은 이미지를 dev·staging·prod에 재사용하고, 설정만 환경별로 갈아끼우는 것이 12-Factor의 핵심(Config).

- **ConfigMap** — 민감하지 않은 설정 (URL, 포트, 기능 플래그)
- **Secret** — 민감한 값 (비밀번호, 토큰, 인증서)

---

## ConfigMap

키-값 설정 묶음. 여러 방식으로 생성한다.

```bash
# literal
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# 파일에서
kubectl create configmap app-config --from-file=config.properties

# 매니페스트
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |            # 파일 통째로도 가능
    server:
      port: 8080
      timeout: 30s
```

### 주입 방법 1 — 환경변수

```yaml
spec:
  containers:
    - name: app
      image: myapp
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
      # 또는 전체를 한 번에 주입
      envFrom:
        - configMapRef:
            name: app-config
```

### 주입 방법 2 — 볼륨 마운트 (파일로)

```yaml
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config      # 각 key가 파일로 마운트됨
```

→ `/etc/config/APP_ENV`, `/etc/config/config.yaml` 파일로 생성됨.

| 방식 | 특징 |
|------|------|
| **환경변수** | 간단, 앱 재시작해야 반영. 대량 설정엔 부적합 |
| **볼륨 파일** | 설정 파일 통째로, ConfigMap 갱신 시 파일 자동 갱신(약간의 지연) |

---

## Secret

민감 데이터용. ConfigMap과 구조는 같지만 값이 **base64 인코딩**되고, 접근 제어(RBAC)·암호화 대상이 된다.

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# TLS 인증서용
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# 도커 레지스트리 인증용
kubectl create secret docker-registry regcred \
  --docker-server=... --docker-username=... --docker-password=...
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=          # base64("admin")
  password: czNjcjN0          # base64("s3cr3t")
# stringData로 평문 작성도 가능 (apply 시 자동 인코딩)
stringData:
  api-key: my-plain-key
```

### 주입 (ConfigMap과 동일)

```yaml
spec:
  containers:
    - name: app
      image: myapp
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      # 또는 파일로 마운트 → /etc/secret/username, /etc/secret/password
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-secret
```

```bash
# 값 확인 (복호화)
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## ⚠️ Secret은 "암호화"가 아니다

**base64는 인코딩일 뿐 암호화가 아니다.** 누구나 디코딩할 수 있다. 진짜 보안을 위해선 추가 조치가 필요하다.

| 조치 | 설명 |
|------|------|
| **etcd 암호화** | 저장 시 암호화 (EncryptionConfiguration) |
| **RBAC 제한** | Secret 읽기 권한을 최소한으로 |
| **Git에 커밋 금지** | 평문/base64 Secret을 리포에 올리지 말 것 |
| **Sealed Secrets** | 암호화된 Secret을 Git에 안전 저장 (Bitnami) |
| **External Secrets** | AWS Secrets Manager·Vault에서 동적 주입 |

> GitOps(10장)에서 특히 중요: 매니페스트를 Git에 올리므로 Secret은 반드시 Sealed Secrets나 External Secrets Operator로 처리한다. 평문 Secret을 커밋하는 것은 대표적 사고 원인.

---

## 설정과 이미지 분리 원칙

```
❌ 이미지에 설정을 굽는다 → 환경마다 다른 이미지 → 재현성·이식성 붕괴
✅ 이미지는 하나, 설정(ConfigMap/Secret)만 환경별로 교체
```

```
같은 이미지 myapp:1.0
  ├─ dev     : configmap-dev   + secret-dev
  ├─ staging : configmap-stg   + secret-stg
  └─ prod    : configmap-prod  + secret-prod
```

---

## 배운 점

- 설정은 이미지 밖으로 → 같은 이미지를 환경별 설정만 바꿔 재사용
- ConfigMap(일반)/Secret(민감), 주입은 **env** 또는 **볼륨 파일** 2가지
- `envFrom`으로 전체 주입, `configMapKeyRef`/`secretKeyRef`로 개별 주입
- **Secret의 base64는 암호화가 아니다** → RBAC·etcd 암호화·Sealed/External Secrets
- GitOps에선 평문 Secret 커밋 금지가 철칙
