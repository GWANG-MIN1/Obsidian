# 02. persistence 를 끄자 Loki 가 죽었다

- **발생**: 2026-07-20 · Phase 2 검증 중 (수정 커밋 21:04 `c5dbd9e`)
- **증상 한 줄**: `loki-0` 이 `1/2 CrashLoopBackOff`, 재시작 8회 — 기동하자마자 exit 1
- **원문**: 저장소 [`docs/troubleshooting/02-loki-read-only-filesystem.md`](https://github.com/GWANG-MIN1/eks-gitops-platform/blob/main/docs/troubleshooting/02-loki-read-only-filesystem.md)

---

## 현상

```
mkdir /var/loki: read-only file system
error initialising module: ruler-storage
```

## 환경

- loki 차트 6.55.0 · `deploymentMode: SingleBinary` · `storage.type: filesystem`
- `singleBinary.persistence.enabled: false` — EBS CSI 드라이버가 없어서 **의도적으로** 끔 → [결정 05](../decisions/05-관측성-데이터를-영속화하지-않는다.md)
- 컨테이너는 read-only 루트 파일시스템 (차트의 보안 기본값)

## 진단

1. CrashLoop 이라 `kubectl -n observability logs loki-0 -c loki` 부터 — **첫 줄이 이미 답이었다.**
2. `describe pod loki-0` 의 Mounts 를 보니 `/etc/loki/config` · `/rules` · `/tmp` 는 있는데 **`/var/loki` 가 없었다.**

반증된 가설은 기록에 없다. 로그 한 줄이 곧장 마운트 목록으로 이끌었다.

## 원인

세 조건이 겹쳤다.

| 조건 | 누가 정했나 |
|---|---|
| persistence 를 끔 | 나 — 결정 05 |
| persistence 가 켜져 있을 때만 `/var/loki` 에 PVC 를 꽂음 | 차트 템플릿 |
| 루트 파일시스템이 read-only | 차트 보안 기본값 |

각각은 합리적인데 셋이 만나면 **Loki 가 데이터를 쓸 경로가 없다.**
정적 검사가 못 잡는 이유도 여기 있다. 렌더링된 매니페스트는 문법적으로 완벽하고,
*"이 경로에 쓰기 가능한 볼륨이 있는가"* 는 프로세스가 떠서 `mkdir` 를 해 봐야 드러난다.

## 해결

values 에 쓰기 가능한 임시 볼륨을 명시하고 push 만 했다. ArgoCD 가 StatefulSet 을 다시 굴려 `loki-0` 이 `2/2 Running` 이 됐다.

```yaml
singleBinary:
  persistence:
    enabled: false
  extraVolumes:
    - name: loki-data
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-data
      mountPath: /var/loki
```

**클러스터 장애를 Git 커밋으로 고친 첫 사례**다. 계획해 둔 push→sync 데모(21:45)보다 41분 먼저 실전으로 루프가 증명됐다.

## 재발 방지

✅ **코드에 들어갔다.** values 에 emptyDir 가 있고, 주석이 원인까지 *"verified on the live cluster"* 로 적었다.
07-22 · 07-23 재생성에서 다시 안 겪었다.

남은 것 —

- **emptyDir 에 `sizeLimit` 이 없고 Loki 에 보존 · 수집 한도(`limits_config`)도 없다.** 로그가 노드 디스크를 한도 없이 쓴다.
  하루짜리 클러스터라 안 드러났을 뿐이다. → [결정 05](../decisions/05-관측성-데이터를-영속화하지-않는다.md)
- **같은 스위치가 차트마다 다르게 동작한다는 건 문서화하지 않았다.** Prometheus 와 Grafana 도 저장 공간을 안 줬지만
  그쪽은 오퍼레이터 · 차트가 emptyDir 를 대신 붙여서 문제가 없었다(Phase 3 에서 정상 기동). Loki 6.x 만 **아무것도 안 붙였다.**

## 배운 점

- **"persistence 를 끈다"는 "볼륨이 필요 없다"가 아니다.** 영속성이 필요 없어도 프로세스는 여전히 디스크에 쓴다.
  → [observability-lab/06](../../../infra-lab/observability-lab/06-logging/README.md) *"🔧 실제로 겪은 것 — read-only 파일시스템"*
- **values 의 `enabled: false` 한 줄은 같아 보여도, 끈 뒤에 무엇이 남는지는 차트 템플릿이 정한다.** 기본값을 끌 때는 그 기본값이 해 주던 일을 따라가 봐야 한다.
- **CrashLoop 은 `logs -c <container>` 부터.** 이번엔 첫 줄이 원인이었다.
