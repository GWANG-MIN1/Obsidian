# 06 스토리지

컨테이너 파일시스템은 Pod가 죽으면 사라진다(ephemeral).  
데이터를 영속화하려면 **Volume**으로 스토리지를 붙이고, 클러스터 규모에선 **PV/PVC**로 스토리지를 추상화한다.

---

## Volume — Pod에 스토리지 붙이기

Volume은 Pod의 컨테이너들이 공유·영속화할 수 있는 저장 공간. 컨테이너보다 오래 산다(Pod 생명주기 기준).

| Volume 타입 | 수명 | 용도 |
|-------------|------|------|
| **emptyDir** | Pod와 동일 (Pod 삭제 시 소멸) | 컨테이너 간 임시 공유, 캐시 |
| **hostPath** | 노드 디스크 | 노드 파일 접근 (테스트·데몬용, 프로덕션 지양) |
| **configMap / secret** | - | 설정·시크릿을 파일로 마운트 (05장) |
| **persistentVolumeClaim** | 독립적 | **영속 데이터 (표준)** |

```yaml
# emptyDir — 같은 Pod 내 컨테이너 간 공유
spec:
  containers:
    - name: writer
      image: busybox
      volumeMounts:
        - name: shared
          mountPath: /data
    - name: reader
      image: busybox
      volumeMounts:
        - name: shared
          mountPath: /data
  volumes:
    - name: shared
      emptyDir: {}
```

---

## PV / PVC — 스토리지 추상화

프로덕션에선 스토리지 제공(인프라)과 사용(앱)을 분리한다.

```
관리자 관점                      개발자 관점
┌──────────────────┐           ┌──────────────────┐
│ PersistentVolume │  ◀─bind─  │      PVC          │  ◀── Pod가 마운트
│ (실제 스토리지)  │           │ (필요 용량 요청)  │
│ EBS 100Gi 등     │           │ "10Gi RWO 주세요" │
└──────────────────┘           └──────────────────┘
```

| 오브젝트 | 관점 | 역할 |
|----------|------|------|
| **PersistentVolume (PV)** | 관리자/인프라 | 실제 스토리지 자원 (EBS, NFS 등) |
| **PersistentVolumeClaim (PVC)** | 개발자/앱 | "이만큼 필요해요"라는 요청 |
| **StorageClass** | 관리자 | PV를 **동적 자동 생성**하는 템플릿 |

> 개발자는 PV(실제 디스크)를 몰라도 된다. **PVC로 "10Gi 짜리 읽기-쓰기 볼륨"만 요청**하면 Kubernetes가 조건 맞는 PV를 붙여준다(bind). 이 추상화가 이식성의 핵심.

### PVC 정의 & 사용

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3      # StorageClass 지정 (동적 프로비저닝)
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: data
          mountPath: /var/lib/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl get pv       # PersistentVolume 목록·상태 (Bound/Available)
kubectl get pvc      # Claim 상태
kubectl get sc       # StorageClass 목록
```

---

## 접근 모드 (Access Mode)

볼륨을 **몇 개 노드가 어떻게** 마운트할 수 있는지.

| 모드 | 약자 | 의미 |
|------|------|------|
| **ReadWriteOnce** | RWO | 단일 **노드**에서 읽기-쓰기 (대부분의 블록 스토리지, EBS) |
| **ReadOnlyMany** | ROX | 여러 노드에서 읽기 전용 |
| **ReadWriteMany** | RWX | 여러 노드에서 읽기-쓰기 (NFS, EFS 등 파일 스토리지) |
| **ReadWriteOncePod** | RWOP | 단일 **Pod**만 읽기-쓰기 (가장 엄격) |

> EBS 같은 블록 스토리지는 RWO(한 노드 전용)라, 여러 노드의 Pod가 동시에 써야 하면 EFS(RWX) 같은 파일 스토리지가 필요하다. StatefulSet의 각 Pod가 전용 PVC를 갖는 이유이기도 하다.

---

## 동적 프로비저닝 (StorageClass)

PV를 미리 수동으로 만들지 않고, PVC가 생기면 **자동으로 PV를 생성**한다. 클라우드에서 표준.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com     # AWS EBS CSI 드라이버
parameters:
  type: gp3
reclaimPolicy: Delete            # PVC 삭제 시 실제 볼륨도 삭제
volumeBindingMode: WaitForFirstConsumer  # Pod 스케줄 후 볼륨 생성 (AZ 정합)
allowVolumeExpansion: true       # 용량 확장 허용
```

```
PVC 생성 → StorageClass의 provisioner 호출 → 클라우드 볼륨(EBS) 생성 → PV 자동 생성 → bind
```

### Reclaim Policy — PVC 삭제 시 PV 처리

| 정책 | 동작 |
|------|------|
| **Delete** | PV·실제 스토리지까지 삭제 (동적 프로비저닝 기본) |
| **Retain** | PV 보존, 데이터 남김 (수동 정리 필요, 중요 데이터) |
| **Recycle** | (폐기됨) |

---

## 배운 점

- 컨테이너 FS는 휘발성 → **Volume**으로 영속화, `emptyDir`는 Pod 수명 임시 공유
- 프로덕션은 **PVC(요청) ↔ PV(실제)** 분리 → 개발자는 실제 디스크를 몰라도 됨
- 접근 모드 RWO(단일노드)/RWX(다중노드) 구분 → EBS vs EFS 선택 기준
- **StorageClass = 동적 프로비저닝** → PVC만 만들면 PV 자동 생성 (EKS 표준)
- `reclaimPolicy: Retain`으로 중요 데이터 보호
