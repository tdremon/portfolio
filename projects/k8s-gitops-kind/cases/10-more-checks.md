# 그 밖에 확인한 것

> 앞의 사례는 결론이 분명한 것을 한 쪽씩 정리했다. 여기에는 **그 밖에 직접 확인한 것**을 한 줄씩 남긴다. 공통 [환경](../README.md#환경).

## 워크로드 종류와 스케줄링

| 주제 | 확인한 것 |
|---|---|
| 리소스 쿼터 | 쿼터가 가득 찬 네임스페이스에서 5번째 파드는 Pending이 아니라 **파드 오브젝트가 아예 안 만들어진다.** 원인은 ReplicaSet 이벤트의 `exceeded quota`에 있다. 롤링 업데이트는 60초 뒤 timeout으로 멈췄고 기존 파드 4개는 유지됐다 |
| 쿼터 크기 | 쿼터를 올려도 바로 풀리지 않고 약 40초 뒤에 진행됐다. 크기는 `(최대 replicas + maxSurge) × requests`로 잡는다(이 앱은 (4 + 2) × 250m = 1500m) |
| LimitRange | 기본값(100m/64Mi)을 넣고 최대값(500m) 초과는 거부 |
| DaemonSet | toleration이 없으면 워커 2대에만 뜨고 추가하면 3대가 된다. 파드를 지우면 **같은 노드에** 3.0초 만에 다시 뜬다(개수가 아니라 노드당 1개를 유지). 롤링 업데이트는 한 번에 노드 1개씩 |
| nodeAffinity | `required`는 맞는 노드가 없으면 Pending, 라벨을 붙이자 **같은 파드**가 0.96초 만에 Running. `preferred`는 다른 노드가 비어 있어도 기본 분산을 이기고 한 노드에 몰렸다(weight 100) |
| Job / CronJob | 실패하는 Job은 파드 3개를 만들고 `BackoffLimitExceeded`. 완료 후 자동 삭제 시간(120초, 300초)이 설정과 맞았다. `Forbid`는 겹침 없이 첫 Job이 끝난 0.014초 뒤에 새 Job이 만들어졌고, `Allow`는 약 40초 겹쳤다 |
| StatefulSet | 순서대로 생성(0번 뒤 1번), 파드를 지우면 **같은 이름·같은 노드·같은 볼륨**으로 4.0초 만에 복구(IP는 바뀜). scale-in은 큰 번호부터이고 StatefulSet을 지워도 PVC는 남는다. 파드별 DNS 이름 풀이는 확인하지 못했다(조회가 실패했고 원인은 확인하지 않음) |

## 패키징과 GitOps

| 주제 | 확인한 것 |
|---|---|
| Helm 4와 Helm 3 예제의 차이 | 실패 시 자동 롤백 옵션이 `--atomic`이 아니라 **`--rollback-on-failure`**. 인터넷 예제 대부분이 v3 기준이라 옵션 확인이 필요하다 |
| `helm upgrade`와 values | `--set replicaCount=3` 뒤에 다른 값만 주고 upgrade하면 replicas가 **1로 되돌아온다.** 이전 `--set`은 이어지지 않으므로 `-f` 파일을 매번 넘기는 방식을 기준으로 삼았다 |
| Helm 4와 kubectl 충돌 | `kubectl scale`로 바꾼 필드가 있으면 `helm upgrade`가 **조용히 덮어쓰지 않고 충돌로 실패**한다(`--force-conflicts`로 덮어씀). chart가 건드리지 않는 필드는 유지된다 |
| 앱 Helm chart | 기존 매니페스트를 chart로 옮기고 설정을 values로 분리한 구성을 검토·검증했다. PDB와 Redis는 토글, 값 검증용 스키마, ConfigMap이 바뀌면 파드가 롤링되도록 체크섬 어노테이션. `lint`, 템플릿 렌더링, 서버 dry-run으로 검증했다 |
| Argo CD | Helm으로 설치하고 앱 repo와 배포 repo를 분리했다. 자동 sync(prune, selfHeal), `replicaCount` 변경 커밋과 `git revert`로 배포·롤백(sync 이력 3건 확인), `kubectl scale`로 바꾼 값이 되돌려지는 selfHeal 확인. **자동 sync가 켜진 앱은 `argocd app rollback`이 막히므로 `git revert`가 기본 롤백**이다 |
| 이미지 | multi-stage, non-root, 모델 가중치를 빌드 시점에 포함, 이미지 1.24GB. `docker stop`이 1.2초에 끝나 종료 신호가 전달됨을 확인 |

## 관측과 운영

| 주제 | 확인한 것 |
|---|---|
| 지표 수집과 대시보드 | 앱 `/metrics`를 ServiceMonitor로 파드 5개 모두 수집, 대시보드 9개 패널을 JSON으로 저장하고 ConfigMap으로 로드해 클러스터를 지워도 복원된다. ServiceMonitor는 선택 조건(라벨, 포트 이름, 네임스페이스)이 어긋나도 에러 없이 **조용히 무시**된다 |
| 클러스터 재현 | 설정 파일 하나로 같은 클러스터를 다시 만든다(노드 3개, 워커별 zone 라벨, 80/443 포트 매핑) |
| 실험 도구 | 부하 발생(순차, 동시 N개), 클러스터 변화를 시간순으로 기록, 작업 시점 표시 스크립트를 구성해 모든 실험을 **로그로 남기고 다시 집계**할 수 있게 했다 |
| 운영 함정 17가지 | 증상 → 원인 → 회피 순으로 정리했다. 예: Service 이름 접두사의 환경변수가 앱 설정과 충돌, ConfigMap을 고쳐도 실행 중인 파드는 그대로, 같은 태그로 덮어쓴 이미지는 갱신되지 않음, 이벤트는 약 1시간만 보관, **yaml에서 `replicas` 줄을 지우고 `apply`하면 파드가 1개로 줄 수 있음**(HPA와 같이 쓸 때) |
