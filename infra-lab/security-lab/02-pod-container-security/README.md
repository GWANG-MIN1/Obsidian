# 02 파드·컨테이너 보안

컨테이너는 **호스트 커널을 공유하는 격리된 프로세스**다. → `../docker-labs/01-container-basics/`  
그래서 컨테이너 안에서 root면 커널 취약점 하나로 노드까지 갈 수 있다. `securityContext`는 그 표면을 좁히는 도구다.

---

## securityContext — 두 단계

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:              # 파드 수준 — 모든 컨테이너에 적용
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001              # 마운트된 볼륨의 그룹 소유권
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      securityContext:          # 컨테이너 수준 — 파드 설정을 덮어쓴다
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

| 설정 | 위치 | 효과 |
|---|---|---|
| `runAsNonRoot` | 파드/컨테이너 | root(UID 0)로 시작하면 **파드가 뜨지 않는다** |
| `runAsUser` | 파드/컨테이너 | 실행 UID 지정 |
| `fsGroup` | **파드만** | 볼륨의 그룹 소유권 (권한 문제 해결) |
| `allowPrivilegeEscalation` | **컨테이너만** | setuid로 권한 상승 차단 |
| `privileged` | **컨테이너만** | 호스트 자원 전체 접근 (사실상 root) |
| `readOnlyRootFilesystem` | **컨테이너만** | 루트 FS 쓰기 금지 |
| `capabilities` | **컨테이너만** | 커널 권한 세분화 |
| `seccompProfile` | 파드/컨테이너 | 시스템콜 필터 |

> `fsGroup`은 **파드 수준에만** 있고 `capabilities`·`readOnlyRootFilesystem`은 **컨테이너 수준에만** 있다. 이 비대칭 때문에 매니페스트를 쓰다 자주 틀린다.

---

## 비-root 실행

```dockerfile
# 이미지에서 (권장 — 런타임 설정을 잊어도 안전하다)
FROM node:20-alpine
RUN addgroup -S app -g 10001 && adduser -S app -G app -u 10001
USER 10001
```

```yaml
# 매니페스트에서
securityContext:
  runAsNonRoot: true      # UID 0 이면 기동 거부
  runAsUser: 10001
```

> **`runAsNonRoot: true`만 두고 `runAsUser`를 안 주면**, 이미지에 `USER`가 숫자로 지정돼 있어야 한다. `USER app`처럼 이름으로만 지정하면 kubelet이 UID를 판정하지 못해 `CreateContainerConfigError`가 난다.  
> 이미지 쪽과 매니페스트 쪽 **양쪽에 거는 게 안전하다.** 이미지가 교체돼도 정책이 남는다.

### 포트 문제

```
비-root 는 1024 미만 포트를 바인딩할 수 없다
  nginx 기본 80 → 비-root 로 못 뜬다
  해결: 8080 으로 바꾼 이미지 사용 (nginxinc/nginx-unprivileged 등)
        또는 capabilities.add: ["NET_BIND_SERVICE"]  ← 최소한의 예외
```

---

## Capabilities

리눅스는 root 권한을 **약 40개 조각(capability)** 으로 나눠 놨다. 컨테이너 런타임은 기본적으로 그중 일부를 준다.

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]                    # 전부 버리고
    add: ["NET_BIND_SERVICE"]        # 꼭 필요한 것만 되돌려 받는다
```

| capability | 무엇을 허용하나 | 위험도 |
|---|---|---|
| `NET_BIND_SERVICE` | 1024 미만 포트 바인딩 | 낮음 |
| `CHOWN` | 파일 소유자 변경 | 낮음 |
| `NET_RAW` | raw 소켓 (ping, 패킷 스니핑) | **중간** — 기본 부여됨 |
| `SYS_ADMIN` | 마운트 등 광범위 | **매우 높음 (사실상 root)** |
| `SYS_PTRACE` | 다른 프로세스 디버깅·메모리 읽기 | 높음 |
| `SYS_MODULE` | 커널 모듈 로드 | **매우 높음** |

> **`drop: ["ALL"]`에서 시작한다.** 대부분의 애플리케이션은 아무것도 필요 없다.  
> `NET_RAW`는 기본으로 붙는데 ARP 스푸핑에 쓰일 수 있어 실제로 버리는 게 좋다.  
> `SYS_ADMIN`을 요구하는 서드파티 차트를 만나면 **왜 필요한지 먼저 확인한다.** 대부분 게으른 기본값이다.

---

## allowPrivilegeEscalation과 privileged

```yaml
allowPrivilegeEscalation: false      # setuid 바이너리로 권한 상승 차단
privileged: false                    # 기본값이지만 명시한다
```

```
privileged: true 의 실제 의미
  = 모든 capability + 호스트 디바이스 접근 + seccomp/AppArmor 해제
  = "컨테이너 안이지만 사실상 노드의 root"
```

> **`privileged: true`가 필요한 워크로드는 극히 드물다.** CNI 플러그인, 노드 에이전트, 스토리지 드라이버 정도다.  
> 앱 컨테이너에 이게 붙어 있으면 거의 항상 잘못된 것이다.

```bash
# 클러스터에서 privileged 컨테이너 찾기
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].securityContext.privileged==true)
  | "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## readOnlyRootFilesystem

```yaml
containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
      - name: tmp
        mountPath: /tmp              # 쓸 곳은 명시적으로 열어준다
      - name: cache
        mountPath: /var/cache
volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

> **공격자가 파드를 장악해도 바이너리를 심을 수 없다.** 효과가 큰 설정인데 도입 비용이 낮다.  
> 다만 대부분의 프로세스는 어딘가에 쓴다(로그·임시파일·PID). `emptyDir`로 필요한 경로만 열어준다.  
> **PVC를 끄면서 이걸 켜면 프로세스가 기동조차 못 한다** — Loki가 `/var/loki`에 쓰지 못해 죽은 사례가 그것이다. → `../observability-lab/06-logging/`

---

## seccomp

시스템콜 자체를 필터링한다.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault      # 런타임의 기본 프로파일 (권장 시작점)
    # type: Localhost
    # localhostProfile: profiles/my-app.json
```

| type | 의미 |
|---|---|
| `RuntimeDefault` | 컨테이너 런타임의 기본 차단 목록 (~44개 위험 syscall 차단) |
| `Localhost` | 노드의 커스텀 프로파일 |
| `Unconfined` | 필터 없음 (기본값이었으나 지양) |

> **`RuntimeDefault`만 켜도 상당수의 커널 익스플로잇 경로가 막힌다.** 호환성 문제도 거의 없어서 우선 적용 대상이다.  
> AppArmor·SELinux는 파일·자원 접근을 강제 제어(MAC)한다. 관리형 노드에서는 손대기 어려운 경우가 많아 우선순위가 낮다.

---

## 그 외 위험한 설정

```yaml
spec:
  hostNetwork: true       # 노드 네트워크 네임스페이스 공유 → 다른 파드 트래픽 접근
  hostPID: true           # 노드의 모든 프로세스가 보인다
  hostIPC: true           # 노드 IPC 공유
  volumes:
    - name: host
      hostPath:
        path: /            # ← 노드 파일시스템 전체 마운트. 사실상 탈출
```

| 설정 | 왜 위험한가 | 정당한 사례 |
|---|---|---|
| `hostNetwork` | 노드 IP·인터페이스 직접 사용 | CNI, 로드밸런서 |
| `hostPID` | 다른 프로세스 메모리 접근 | 노드 모니터링, **kube-bench** |
| `hostPath: /` | 노드 파일시스템 읽기·쓰기 | 거의 없음 |

> **`hostPath` 마운트가 컨테이너 탈출의 가장 흔한 실제 경로다.** `/var/run/docker.sock`을 마운트하면 그 순간 노드 전체를 제어할 수 있다.  
> 정당하게 필요한 워크로드(kube-bench)는 **네임스페이스를 분리하고 정책 예외로 명시**한다. → `04-kyverno/`

---

## 정책으로 강제하기

한 번 잘 쓰는 것보다 **모든 워크로드에 강제되는 것**이 중요하다.

```yaml
# Kyverno — 권한 제한 (실제 정책)
validate:
  failureAction: Audit
  message: >-
    Containers must set allowPrivilegeEscalation=false, must not be
    privileged, and must drop ALL capabilities.
  pattern:
    spec:
      containers:
        - securityContext:
            allowPrivilegeEscalation: false
            =(privileged): false        # 조건부: 설정돼 있다면 false 여야 한다
            capabilities:
              drop:
                - ALL
```

> `=(privileged)`의 `=()`는 Kyverno의 **조건부 앵커**다. "이 필드가 존재하면 이 값이어야 한다"는 뜻으로, 필드 자체를 필수로 만들지는 않는다. → `04-kyverno/`  
> 강제 수단은 두 가지다: **Pod Security Admission**(내장, 표준 프로파일) 또는 **Kyverno**(유연, 커스텀). → `03-pod-security-standards/`

---

## 안전한 기본 템플릿

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      automountServiceAccountToken: false    # API 접근이 필요 없으면 끈다
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: myapp:1.4.2                 # :latest 금지
          securityContext:
            allowPrivilegeEscalation: false
            privileged: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 256Mi }      # CPU limit 은 두지 않는다
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

> **`automountServiceAccountToken: false`가 의외로 중요하다.** 기본값은 true라, API 서버를 쓰지 않는 앱에도 토큰이 마운트된다. 파드가 뚫리면 그 토큰이 공격자에게 넘어간다. → `05-rbac-least-privilege/`  
> CPU limit을 두지 않는 이유는 스로틀링 때문이다 — 관측성 스택에서와 같은 판단이다. → `../observability-lab/08-kube-prometheus-stack/`

---

## 배운 점

- 컨테이너는 커널을 공유한다 — **컨테이너 root는 노드 root로 가는 출발점**
- `securityContext`는 **파드 수준과 컨테이너 수준**이 있고 쓸 수 있는 필드가 다르다
- `fsGroup`은 파드에만, `capabilities`·`readOnlyRootFilesystem`은 컨테이너에만
- 비-root는 **이미지(`USER`)와 매니페스트 양쪽에** 거는 게 안전하다
- `USER`를 이름으로만 지정하면 `runAsNonRoot` 검사에서 기동 실패한다
- 비-root는 **1024 미만 포트를 못 연다** — 8080으로 바꾸거나 `NET_BIND_SERVICE`만 추가
- **`capabilities.drop: ["ALL"]`에서 시작**하고 필요한 것만 되돌려 받는다
- `NET_RAW`는 기본 부여되지만 ARP 스푸핑에 쓰일 수 있어 버리는 게 좋다
- `SYS_ADMIN`은 사실상 root — 요구하는 차트는 이유를 먼저 확인
- **`privileged: true`가 정당한 앱 컨테이너는 거의 없다**
- `readOnlyRootFilesystem`은 비용 대비 효과가 크다 — 쓸 경로만 `emptyDir`로 연다
- **PVC를 끄면서 읽기전용 FS를 켜면 프로세스가 기동조차 못 한다**
- **`seccompProfile: RuntimeDefault`** 는 호환성 문제가 적고 효과가 크다
- `hostNetwork`·`hostPID`·`hostPath`는 탈출 경로 — **`hostPath` 마운트가 가장 흔하다**
- 정당한 예외(kube-bench)는 네임스페이스를 분리하고 정책에서 명시적으로 제외
- **`automountServiceAccountToken: false`** — API를 안 쓰는 앱에 토큰을 주지 않는다
- 한 번 잘 쓰는 것보다 **모든 워크로드에 강제되는 것**이 중요하다
