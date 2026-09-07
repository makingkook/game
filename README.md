# STICK DEATHMATCH — 온라인 1vs1 졸라맨 데스매치

서로 다른 컴퓨터에서 같은 주소로 접속하면 자동으로 매칭되어 실시간으로 대전하는
웹소켓 기반 온라인 대전 게임입니다. 서버가 물리/전투 연산을 전담(authoritative)하고,
브라우저는 매 틱 전달되는 상태를 받아 그려주기만 하는 구조입니다.

## 폴더 구조

```
stickfight-multi/
├── 웹서버.py          ← FastAPI 웹 서버 (정적 파일 서빙 + WebSocket 게임 로직)
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── public/            ← 정적 웹사이트 (브라우저에 그대로 전달되는 파일)
│   ├── index.html
│   ├── style.css
│   └── game.js
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

## 동작 방식

- 첫 번째, 두 번째로 접속한 클라이언트가 각각 **Player 1 / Player 2**로 배정됩니다.
  세 번째 이후 접속자는 **관전자(spectator)**로 입장합니다.
- 각 클라이언트는 `/ws` WebSocket을 통해 자신의 키 입력 상태만 서버로 전송합니다.
- 서버는 초당 60틱으로 물리·충돌·전투를 계산한 뒤, 결과 상태를 모든 접속자에게 즉시 브로드캐스트합니다.
- 두 플레이어가 모두 접속하면 자동으로 대전이 시작됩니다(별도의 "시작" 버튼 없음).
- 3판 2선승. 매치가 끝나면 아무 플레이어나 "다시 시작"을 누르면 새 매치가 시작됩니다.

## 조작법 (양쪽 플레이어 동일 — 각자 자기 키보드 사용)

| 동작 | 키 |
|---|---|
| 이동 | A / D 또는 ← / → |
| 점프 | W / ↑ / Space |
| 약공격 | F 또는 J |
| 강공격(차지) | G 또는 K (누르고 있다가 떼면 발동) |

---

## ⚠️ 중요: 상태 저장 구조와 스케일링 한계

`웹서버.py`는 대전방(게임 상태)을 **프로세스 메모리**에 들고 있는 단일 `GameRoom` 하나만 지원합니다.
즉, **파드(Pod)를 2개 이상으로 늘리면 두 플레이어가 서로 다른 파드에 연결되어 매칭되지 않습니다.**
`k8s/deployment.yaml`에는 `replicas: 1`로 고정해두었으니 그대로 유지하세요.

여러 개의 동시 대전방(멀티 룸)이나 다중 파드 확장이 필요하다면:
- 방(room) 코드/ID를 도입해 클라이언트가 방을 선택하게 하고
- Redis 등 외부 저장소로 방 상태를 옮기거나, 방 단위로 sticky routing을 적용하는 구조로 확장하면 됩니다.
(현재 버전은 "친구끼리 접속해서 즉석으로 1:1 대전"하는 간단한 데모 수준입니다.)

---

## 로컬 실행

```bash
pip install -r requirements.txt
uvicorn 웹서버:app --host 0.0.0.0 --port 8080
```

브라우저 두 개(또는 같은 네트워크의 다른 컴퓨터 두 대)에서 `http://<서버IP>:8080` 접속.

---

## Docker 빌드 & 로컬 테스트

```bash
docker build -t stick-deathmatch:latest .
docker run --rm -p 8080:8080 stick-deathmatch:latest
```

---

## ECR에 이미지 업로드

```bash
$env:AWS_ACCOUNT_ID="950274644703"
$env:AWS_REGION="ap-northeast-1"
$env:REPO="game"

# 리포지토리 생성 (최초 1회)
aws ecr create-repository --repository-name $env:REPO --region $env:AWS_REGION

# 로그인
aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# 태그 & 푸시
docker build -t $REPO:latest .
docker tag $REPO:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO:latest
```

---

## EKS 배포

1. `k8s/deployment.yaml`의 `image:` 값을 방금 푸시한 ECR URI로 수정
   ```yaml
   image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/stick-deathmatch:latest
   ```
2. kubeconfig 연결
   ```bash
   aws eks update-kubeconfig --name <클러스터이름> --region $AWS_REGION
   ```
3. 배포
   ```bash
   kubectl apply -f k8s/deployment.yaml
   kubectl apply -f k8s/service.yaml
   ```
4. 외부 접속 주소 확인 (LoadBalancer가 프로비저닝될 때까지 1~2분 소요)
   ```bash
   kubectl get service stick-deathmatch -w
   # EXTERNAL-IP 컬럼에 표시되는 ELB/NLB 주소로 접속
   ```
5. 서로 다른 컴퓨터에서 `http://<EXTERNAL-IP>` 로 각각 접속하면 자동 매칭되어 대전이 시작됩니다.

### 참고사항

- `service.yaml`은 `type: LoadBalancer` + `aws-load-balancer-type: nlb` annotation을 사용합니다.
  클러스터에 **AWS Load Balancer Controller**가 설치되어 있지 않다면 annotation을 지워도 되며,
  이 경우 기본 in-tree provider가 Classic ELB를 생성합니다(WebSocket 통신에는 문제 없습니다).
- 도메인 + HTTPS(wss://)를 쓰고 싶다면 ALB Ingress(ACM 인증서)나 NLB + ACM으로 TLS를 종단하면 됩니다.
  ALB를 쓰는 경우 WebSocket은 기본 지원되며, idle timeout을 게임 핑 주기보다 넉넉히(예: 300초) 설정하는 것을 권장합니다.
- `readinessProbe` / `livenessProbe`는 `/healthz`를 사용합니다.
- 리소스 요청/제한은 `deployment.yaml`에서 필요에 맞게 조정하세요.

### 정리

```bash
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
```
