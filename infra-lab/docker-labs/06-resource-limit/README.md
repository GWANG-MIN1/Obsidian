# 06 컨테이너 자원 할당 제한

Docker는 컨테이너가 사용할 수 있는 메모리와 CPU 자원을 제한하는 옵션을 제공한다.  
제한을 설정하지 않으면 컨테이너가 호스트 자원을 무제한으로 사용할 수 있어 다른 컨테이너나 호스트에 영향을 줄 수 있다.

---

## 컨테이너 메모리 제한 (`--memory`)

컨테이너가 사용할 수 있는 최대 메모리를 지정한다.  
제한을 초과하면 OOM(Out of Memory) Killer에 의해 프로세스가 종료된다.

```bash
# 최대 512MB로 제한
docker run -d \
  --name myapp \
  --memory 512m \
  nginx

# 메모리 + 스왑 합산 제한 (--memory-swap)
# --memory-swap = 메모리 + 스왑 총합
# 아래 예시: 메모리 512m, 스왑 512m (스왑 실질 사용량 = 1g - 512m = 512m)
docker run -d \
  --name myapp \
  --memory 512m \
  --memory-swap 1g \
  nginx

# 스왑 완전 비활성화
docker run -d \
  --name myapp \
  --memory 512m \
  --memory-swap 512m \
  nginx
```

| 단위 | 설명 |
|------|------|
| `b` | 바이트 |
| `k` | 킬로바이트 |
| `m` | 메가바이트 |
| `g` | 기가바이트 |

---

## 컨테이너 CPU 제한

### `--cpu-shares` — CPU 상대적 가중치

절대적인 CPU 수를 지정하는 게 아니라 컨테이너 간 상대적인 CPU 사용 비율을 지정한다.  
기본값은 1024이며, 경합이 없을 때는 제한 없이 사용 가능하다.

```bash
# 기본값(1024)의 절반 가중치
docker run -d --name low-priority --cpu-shares 512 nginx

# 기본값(1024)의 두 배 가중치
docker run -d --name high-priority --cpu-shares 2048 nginx
```

> `--cpu-shares`는 CPU 경합이 발생할 때만 효과가 있다. 다른 컨테이너가 CPU를 쓰지 않으면 낮은 가중치 컨테이너도 풀 사용 가능.

---

### `--cpuset-cpus` — 사용할 CPU 코어 고정

컨테이너가 실행될 수 있는 CPU 코어를 특정 코어로 고정한다.

```bash
# CPU 0번, 1번 코어만 사용
docker run -d --name myapp --cpuset-cpus="0,1" nginx

# CPU 0번~3번 코어 사용 (범위 지정)
docker run -d --name myapp --cpuset-cpus="0-3" nginx

# CPU 2번 코어만 단독 사용
docker run -d --name myapp --cpuset-cpus="2" nginx
```

> **CPU를 많이 소모하는 워크로드를 수행해야 한다면 `--cpuset-cpus` 옵션을 사용하는 것이 좋다.**  
> CPU 친화성(CPU Affinity)을 보장하여 컨테이너가 항상 동일한 코어에서 실행되기 때문에,  
> CPU 캐시 미스(Cache Miss)나 컨텍스트 스위칭(Context Switching)처럼 성능을 하락시키는 요인을 최소화할 가능성이 높아지기 때문이다.

---

### `--cpu-period` & `--cpu-quota` — CFS 스케줄러 기반 CPU 제한

Linux CFS(Completely Fair Scheduler)의 주기와 할당량을 직접 지정한다.  
`--cpu-period`(주기, 마이크로초) 내에서 `--cpu-quota`(할당량, 마이크로초)만큼만 CPU를 사용할 수 있다.

```bash
# period 100ms 중 50ms 사용 → CPU 50% 제한
docker run -d \
  --name myapp \
  --cpu-period=100000 \
  --cpu-quota=50000 \
  nginx

# period 100ms 중 200ms 사용 → CPU 200% (2코어 상당)
docker run -d \
  --name myapp \
  --cpu-period=100000 \
  --cpu-quota=200000 \
  nginx
```

> CPU 사용률 = `--cpu-quota` / `--cpu-period`  
> 예: 50000 / 100000 = 0.5 → 50%

---

### `--cpus` — CPU 개수 직접 지정 (권장)

`--cpu-period`와 `--cpu-quota`를 내부적으로 자동 계산해주는 간편한 옵션.  
실수값으로 사용할 CPU 코어 수를 직접 지정할 수 있다.

```bash
# CPU 0.5코어 사용 제한 (단일 코어의 50%)
docker run -d --name myapp --cpus="0.5" nginx

# CPU 1.5코어 사용 제한
docker run -d --name myapp --cpus="1.5" nginx

# CPU 2코어 사용 제한
docker run -d --name myapp --cpus="2" nginx
```

> `--cpus="1.5"` 는 내부적으로 `--cpu-period=100000 --cpu-quota=150000`과 동일하다.

---

## 메모리 + CPU 복합 제한 예시

```bash
docker run -d \
  --name myapp \
  --memory 1g \
  --memory-swap 1g \
  --cpus="2" \
  --cpuset-cpus="0,1" \
  nginx
```

---

## CPU 제한 옵션 비교

| 옵션 | 방식 | 특징 |
|------|------|------|
| `--cpu-shares` | 상대적 가중치 | 경합 시에만 효과, 유연함 |
| `--cpuset-cpus` | 코어 고정 | CPU 친화성 보장, 고성능 워크로드에 적합 |
| `--cpu-period` + `--cpu-quota` | CFS 직접 제어 | 세밀한 조정 가능 |
| `--cpus` | CFS 자동 계산 | 가장 간단하고 직관적, 권장 방식 |
