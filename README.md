# 🍎 Apple Silicon Mac 환경 Judge0 구축 가이드

이 문서는 Apple Silicon(M1, M2, M3 등) 칩셋을 사용하는 Mac 환경에서 Judge0 인프라를 구축할 때 발생하는 cgroup version 이슈 및 Architecture(amd64/arm64) 불일치 문제를 해결하기 위한 가이드이다.
아래 순서대로 설정을 마친 후 `docker-compose up -d` 로 한 번에 judge0 환경 설정을 마칠 수 있다.

---

## 🛠 1. Docker Desktop cgroup v1 강제 설정

Judge0의 Sandbox(Isolate)는 cgroup v1 환경에서 안정적으로 동작한다.

(이 부분은 linux를 자세히 모르면 이해 안갈거니까 그러려니 하고 넘어가자.)

결론적으로는 Docker Desktop의 기본값인 v2를 v1으로 하향 조정해야 한다.

1. **설정 파일 열기** 

```bash
code "~/Library/Group Containers/group.com.docker/settings.json"
```

2. **값 수정**
`deprecatedCgroupv1` 항목을 찾아 `false`에서 true 로 변경. (없으면 그대로 추가해주면

```json
{
  "deprecatedCgroupv1": true,
  ...
}
```

3. **Docker Desktop 재시작**
    - Docker를 완전히 종료(Quit)한 후 다시 실행해야 설정이 적용된다.

---

## 📥 2. 프로젝트 클론 및 설정 파일 복사

지금 레포에 올려놓은 `docker-compose.yml` 및 `judge0.conf` 파일은 이미 수정해놓은 것이다.

클론해서 그대로 사용하면 되지만, 기록용으로 아래 공식 설정에서의 변경점만 서술해놓겠다.

---

## 🐳 3. docker-compose.yml 수정

Apple Silicon Mac에서 amd64 기반의 Judge0 이미지를 실행하기 위해 `platform: linux/amd64` 옵션을 모든 서비스에 추가해야 한다.

```yaml
services:
  server:
    image: judge0/judge0:latest
    platform: linux/amd64  # <--- 추가
    volumes:
    ports:
      - "2358:2358"
      - ./judge0.conf:/etc/judge0.conf:ro
    # ... 생략

  worker:
    image: judge0/judge0:latest
    platform: linux/amd64  # <--- 추가
    privileged: true
    volumes:
      - ./judge0.conf:/etc/judge0.conf:ro
    # ... 생략

  db:
    image: postgres:16.2
    platform: linux/amd64  # <--- 추가
    # ... 생략

  redis:
    image: redis:7.2.4
    platform: linux/amd64  # <--- 추가
    # ... 생략
```

---

## ⚙️ 4. judge0.conf 주요 설정

아래 둘은 default 를 줄 수가 없다. 명시적으로 입력한다.

- **REDIS_PASSWORD**
- **POSTGRES_PASSWORD**

그 외의 설정들은 기본 값들이 있지만 worker의 수 같은 경우 default로 두면 (CPU 코어 개수 * 2배)로 자동 설정되므로 일단은 2개로 설정했다.

```yaml
################################################################################
# Judge0 Workers Configuration
################################################################################
# 0.1초마다 큐를 확인
INTERVAL=0.1
# 동시에 처리할 작업 수
COUNT=2 # <--- 빈칸으로 두면 CPU 코어 수의 2배로 자동 설정됨
```

---

## 🚀 5. 실행 및 정상 작동 확인

모든 설정이 완료되었다면 아래 명령어로 시스템을 가동해보자.

```yaml
# 백그라운드 실행
docker compose up -d

# Worker 로그 실시간 확인 (가장 에러가 잦은 부분은 isloate 환경을 직접 제어하는 worker)
docker compose logs -f worker
```

### 추가 확인 사항: Rosetta 2 설치 여부

현재 `docker-compose.yml`에서 `platform: linux/amd64`를 사용할건데 apple silicon mac에서 이를 돌리려면 **Rosetta 2**가 반드시 필요하다. 만약 아직 설치하지 않았다면 터미널에 다음 명령어를 입력해 설치해줘야한다.

```bash
softwareupdate --install-rosetta
```

모든 container가 up 상태면 API 명세에 적어둔 endpoint로 필수 파라미터 담아서 test 해보면 된다.

일단 아래 문서 열리는지 부터 확인해볼것.

```json
[GET] http://localhost:2358/docs
```
