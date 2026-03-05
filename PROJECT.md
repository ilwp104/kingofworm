# King of Worm - 프로젝트 분석

## 1. 개요

| 항목 | 내용 |
|------|------|
| 이름 | King of Worm |
| 버전 | 1.0.0 |
| 타입 | 실시간 멀티플레이어 웹 게임 |
| 컨셉 | 뱀(웜)을 조종해 먹이를 먹고 성장하며, 다른 뱀과 경쟁하여 1위를 차지하는 게임 |
| GitHub | https://github.com/ilwp104/kingofworm |

## 2. 기술 스택

| 구분 | 기술 |
|------|------|
| 런타임 | Node.js 20 (Alpine) |
| 서버 | Express 4.18 |
| 실시간 통신 | WebSocket (`ws` 8.16) |
| 렌더링 | HTML5 Canvas 2D |
| 배포 | Docker / Docker Compose |
| 수익화 | Google AdSense |

## 3. 프로젝트 구조

```
kingofworm/
├── server.js              # 게임 서버 (Express + WebSocket + 게임 로직)
├── package.json           # 의존성 (express, ws)
├── Dockerfile             # Docker 이미지
├── docker-compose.yml     # 배포 설정
├── localGame.html         # 로컬 싱글플레이 (단일 파일, 서버 불필요)
└── public/
    ├── index.html         # 클라이언트 HTML (UI, CSS, AdSense)
    └── game.js            # 클라이언트 JS (렌더링, 입력, WebSocket)
```

## 4. 아키텍처

### 두 가지 모드

| 모드 | 파일 | 설명 |
|------|------|------|
| 멀티플레이 | `server.js` + `public/` | 서버가 게임 로직 처리, 클라이언트는 렌더링만 |
| 싱글플레이 | `localGame.html` | 하나의 HTML에 로직+렌더링 전부 포함 |

### 멀티플레이 통신 흐름

```
[클라이언트 game.js]                    [서버 server.js]
    |                                        |
    |-- join {name} ----------------------->|  접속
    |<-- welcome {id, worldSize, palettes} -|
    |                                        |
    |-- input {angle, boost} (50ms) ------->|  입력
    |                                        |  게임 틱 (30Hz)
    |<-- state {snakes, foods, lb, mm} -----|  상태 전송 (15Hz)
    |                                        |
    |-- respawn {name} -------------------->|  리스폰
    |<-- death {score, length, kills} ------|  사망 알림
```

### 네트워크 최적화

- 키 이름 1글자 축약 (`t`, `s`, `f`, `y`, `lb`, `mm`)
- 뱀 세그먼트 최대 60개로 다운샘플링
- 좌표 정수화 `(x + 0.5) | 0`
- VIEW_RANGE(1800px) 내 데이터만 전송
- 게임 로직 30Hz / 전송 15Hz 분리

## 5. 게임 상수

| 상수 | 값 | 설명 |
|------|----|------|
| WORLD_SIZE | 6000 | 맵 크기 (6000×6000) |
| TICK_RATE | 30 | 게임 루프 (30 FPS) |
| SEND_RATE | 15 | 전송 주기 (15 FPS) |
| FOOD_COUNT | 800 | 먹이 수 |
| AI_COUNT | 15 | AI 뱀 수 |
| INITIAL_LENGTH | 20 | 초기 길이 |
| BASE_SPEED | 3.2 | 기본 속도 |
| BOOST_SPEED | 6 | 부스트 속도 |
| TURN_RATE | 0.12 | 회전 보간 계수 |
| VIEW_RANGE | 1800 | 가시 범위 |
| GRID_CELL | 200 | 충돌 그리드 셀 크기 |

## 6. 게임 로직

### Snake

- **이동:** 매 틱 머리 세그먼트 추가 + 꼬리 제거
- **부스트:** 길이 20↑일 때 속도 2배, 세그먼트 2개씩 소모 (30% 확률 먹이 드롭)
- **사망:** 경계 충돌 또는 다른 뱀 몸통 충돌 → 세그먼트 3개마다 큰 먹이 드롭
- **점수:** `세그먼트 수 - 20`
- **두께:** `min(6 + length × 0.08, 22)` — 성장할수록 굵어짐

### 충돌 감지 (Spatial Grid)

- 200px 셀 공간 해시 그리드로 O(n) 충돌 검사
- 인덱스 8부터 2개 간격으로 세그먼트 등록 (머리 근처 스킵)
- 머리-몸통 충돌만 검사 (머리끼리 충돌 없음)

### AI

- **먹이 탐색:** 가장 가까운 먹이를 향해 이동
- **위험 감지:** 120px 내 큰 뱀 → 도주 + 부스트
- **경계 회피:** 가장자리 400px 이내 → 중앙 방향 회전
- **순환 방지:** 같은 먹이 주위 2바퀴(4π) 이상 → 후퇴 모드 (40~60틱)
- **스폰:** 50% 확률로 플레이어 근처 (800~2000px)

### 먹이

| 종류 | 반경 | 성장 | 발생 조건 |
|------|------|------|-----------|
| 일반 | 5 | +1 | 자동 보충 (800개 유지) |
| 큰 먹이 | 9 | +3 | 뱀 사망 / 부스트 드롭 |

## 7. 렌더링

### 카메라

- 플레이어 머리를 부드럽게 추적 (보간 0.15)
- 길이에 비례 줌아웃: `zoom = max(0.45, 1 - length × 0.0008)`

### 최적화

- shadowBlur 대신 반투명 원으로 글로우 효과
- 먹이를 색상별 그룹으로 배치 렌더링 (fillStyle 변경 최소화)
- sin/cos 먹이 흔들림 애니메이션

### 뱀 렌더링 순서

1. 글로우 (반투명 20% 외곽)
2. 바디 (2색 교대 패턴, 12가지 팔레트)
3. 머리 (글로우 + 코어)
4. 눈 (흰자 + 동공, 진행 방향 오프셋)
5. 이름 (자신 또는 길이 25↑)
6. 왕관 (1위에게 금색 왕관 + 보석)

### UI

| 화면 | 요소 |
|------|------|
| 시작 | 이름 입력, PLAY, 최고/최근 점수, 조작법 |
| 인게임 | 점수/길이/킬, 리더보드(TOP 10), 미니맵, HOME |
| 게임 오버 | 최종 점수, 최고 점수, PLAY AGAIN |
| 1위 표시 | 화면 밖 1위 뱀 방향 화살표 (근접 시 빨간 깜빡임) |

### 모바일

- 터치 모드: **Touch** (화면 터치) / **Joystick** (가상 스틱)
- 부스트 버튼 (우하단)
- 반응형 CSS (768px 이하)
- 자동 전체화면

## 8. AdSense 광고

| 위치 | ID | 크기 | 비고 |
|------|----|------|------|
| 시작 화면 | `ad-start` | 728×90 | |
| 인게임 | `ad-ingame` | 320×50 | 모바일 숨김 |
| 게임 오버 | `ad-gameover` | 336×280 | |

## 9. 배포

### Docker Compose

```yaml
포트: 3000
메모리: 2GB~8GB
CPU: 4~8코어
NODE_OPTIONS: --max-old-space-size=6144
재시작: unless-stopped
```

### Dockerfile

```
node:20-alpine → npm install --production → node server.js
```

## 10. WebSocket 프로토콜

### 클라이언트 → 서버

| 타입 | 필드 | 설명 |
|------|------|------|
| join | `{type, name}` | 게임 참가 |
| input | `{type, a, b}` | 방향(angle) + 부스트(bool), 50ms 간격 |
| respawn | `{type, name}` | 리스폰 |

### 서버 → 클라이언트

| 타입 | 필드 | 설명 |
|------|------|------|
| w | `{t, id, ws, p}` | 접속 성공 (ID, 월드크기, 팔레트) |
| s | `{t, s, f, y, lb, mm}` | 상태 (뱀, 먹이, 내ID, 리더보드, 미니맵) |
| d | `{t, sc, l, k}` | 사망 (점수, 길이, 킬) |

### 데이터 포맷

```
뱀: [id, name, segments[x,y,...], paletteIdx, length, angle, boosting, score, kills]
먹이: [x, y, color, radius]
미니맵: [x, y, paletteIdx, isPlayer]
```

## 11. 코드 규모

| 파일 | 라인 | 역할 |
|------|------|------|
| server.js | 547 | 서버 (게임 로직 + WS + Express) |
| public/game.js | 703 | 클라이언트 (렌더링 + 입력 + 통신) |
| public/index.html | 245 | HTML + CSS + AdSense |
| localGame.html | 507 | 싱글플레이 (올인원) |
| **합계** | **~2,000** | |
