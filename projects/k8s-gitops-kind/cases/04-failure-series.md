# 장애 11건을 일부러 재현하며, 증상에서 원인까지 좁혀 가는 순서를 정리했다

> 잘못된 이미지 태그, 설정 누락, 메모리 부족, 기동 지연, 노드 장애 같은 **운영에서 자주 만나는 장애를 직접 일으키고** `증상 → 원인 → 진단 → 해결` 순서로 기록했다. 핵심은 **종료 코드와 이벤트 문구로 원인을 갈라 내는 것**이다.

**조건** 장애마다 별도 실험, 앱 파드 2~5개, 부하를 계속 보내며 서비스에 영향이 있는지 함께 관찰 · 공통 [환경](../README.md#환경)

## 장애 11건 요약

| 장애 | 증상과 원인 | 진단 → 해결 |
|---|---|---|
| 없는 이미지 태그 | 새 파드가 `ErrImagePull` → `ImagePullBackOff`, 롤아웃이 끝나지 않음. 레지스트리에 없는 태그 | `describe`의 `Failed to pull image ... NotFound` → `rollout undo` |
| 롤링 중 5초 지연 | 종료가 시작된 파드로 간 요청이 연결 타임아웃 5초 뒤 재시도됨 | 응답 1초 초과 요청만 추려 종료 시각과 대조 → `preStop: sleep 5` ([상세](03-graceful-shutdown.md)) |
| 설정(ConfigMap) 누락 | 새 파드만 `CreateContainerConfigError`, 기존 파드는 유지 | 이벤트 확인 → 복구 후 `apply` ([상세](03-graceful-shutdown.md)) |
| 스케줄 불가(Pending) 3종 | selector 불일치, taint, requests 초과가 각각 다른 이벤트 문구 | `FailedScheduling` 한 줄 ([상세](07-scheduling-storage.md)) |
| 캐시(Redis) 장애 | 캐시가 죽어도 앱이 캐시 없이 직접 추론해 API는 계속 응답(앱 설계) | 로그의 `cache get failed` ([상세](08-cache-degradation.md)) |
| 서비스 환경변수 충돌 | Service를 만들자 자동 주입된 `REDIS_PORT=tcp://...`가 앱 설정과 겹쳐 기동 직후 죽음 | 앱 로그 → `enableServiceLinks: false`와 변수 이름 변경 |
| 메모리 부족(OOMKilled) | 메모리 limit을 256Mi로 낮추자 모델 로딩 중 `OOMKilled` → `CrashLoopBackOff` | `describe`의 Last State(종료 코드 137). **앱 로그에는 흔적이 없다** → `rollout undo`, limit은 실제 사용량보다 여유 있게 |
| 잘못된 설정값 | `STARTUP_DELAY=abc`로 시작 직후 종료 코드 1, `CrashLoopBackOff` | `logs --previous`의 트레이스백 → `rollout undo`. **OOM(137)과 종료 코드로 갈린다** |
| startup probe 없는 기동 | liveness가 `/readyz`이고 모델 로딩이 60초면 약 40초마다 재시작을 반복, 영영 Ready가 안 됨 | `Liveness probe failed: statuscode 503` → startup probe를 로딩보다 길게(2초 × 40 = 80초) |
| 노드 장애 | 워커를 멈추자 54초 뒤 `NotReady`, 약 5분 뒤 파드가 다른 노드에 재생성(교체 파드 21초 뒤 Ready) | `get nodes` → `describe node`의 unreachable taint. 그 사이 남은 파드 1개가 서비스, 느린 요청 5.00초가 6건 |
| HPA scale-out 중 리소스 부족 | 파드 20개를 요청하자 13개만 Running, 7개 Pending | `Insufficient cpu` → HPA를 원래 값으로 복원 |

노드를 비울 때(drain)의 서비스 영향은 장애 목록과 별개로 [05](05-node-drain.md)에서 다룬다.

## 내가 내린 판단

- **원인은 종료 코드와 이벤트 문구로 갈린다.** OOM은 종료 코드 137과 Last State, 설정 오류는 종료 코드 1과 `logs --previous`, probe 문제는 `Liveness probe failed`, 스케줄 문제는 `FailedScheduling`, 이미지 문제는 `Failed to pull image`다. 로그부터 보면 OOM은 놓친다.
- **롤아웃 장애(잘못된 태그, 설정, 메모리, 기동 지연)는 새 파드만 실패하고 기존 파드가 서비스를 지켰다.** 롤링 설정(`maxUnavailable`)과 readiness가 방화벽 역할이다. 다만 멈춘 롤아웃은 알림이 없으면 눈에 띄지 않는다.
- **노드 장애는 자동으로 복구되지만 5분이 걸린다.** 그동안 남은 파드가 받으므로 replicas를 2 이상, 가능하면 여러 노드에 분산해야 한다.

**한계** 장애마다 1회 재현한 기록이다. 없는 태그 실험은 부하 없이 해서 요청 실패 여부를 확인하지 못했다. CPU limit과 스레드 수 불일치(throttling)는 측정값이 남지 않아 이 목록에서 제외했다. **다음 단계(EKS):** Pending 파드가 노드 증설의 트리거가 되는 Cluster Autoscaler/Karpenter.
