# RuView 사업화 타당성 검증 계획

> 작성일: 2026-03-25
> 목적: WiFi CSI 기반 비접촉 인식 기술의 사업화 가능성을 단계적으로 검증
> 방식: 소프트웨어 시뮬레이션 → 실동작 확인 → 커스터마이징 순으로 리스크 최소화

---

## Phase 1. 개념 및 동작 원리 파악

> 목표: "이 기술이 실제로 동작하는가?" — 하드웨어 없이 소프트웨어만으로 핵심 원리를 이해하고 검증

### 1.1 시뮬레이션 환경 구축

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| Docker 실행 | `docker pull ruvnet/wifi-densepose:latest && docker run -p 3000:3000 -p 3001:3001 ruvnet/wifi-densepose:latest` | 서버 기동 확인, 3000 포트 응답 |
| 웹 UI 접속 | 브라우저에서 `http://localhost:3000` | 대시보드 렌더링, 시뮬레이션 데이터 흐름 |
| Mock CSI 확인 | UI → Sensing 탭에서 시뮬레이션 모드 작동 | RSSI 그래프, 존재 감지 토글 |

**판단 기준**: 시뮬레이션 데이터로 UI가 정상 렌더링되고, 센싱 파이프라인이 동작하면 통과

### 1.2 CSI 신호처리 파이프라인 원리 검증

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| Python 증명 검증 | `cd v1 && python data/proof/verify.py` | `VERDICT: PASS` 출력 (SHA-256 해시 일치) |
| 참조 신호 분석 | `v1/data/proof/sample_csi_data.json` 내용 확인 | 1,000프레임, 3안테나×56서브캐리어 구조 |
| 특징 추출 확인 | verify.py 실행 시 출력되는 amplitude_mean, phase_diff 등 | 7-12종 특징이 결정론적으로 추출되는지 |

**판단 기준**: 동일 입력 → 동일 출력(해시 일치)이 재현되면 파이프라인 무결성 확인

### 1.3 핵심 알고리즘 이해

| 항목 | 확인 파일 | 핵심 개념 |
|------|----------|----------|
| CSI란 무엇인가 | `docs/adr/ADR-013-*` | WiFi 서브캐리어별 진폭/위상 → 인체 반사 패턴 |
| 신호 → 자세 변환 | `v1/src/models/modality_translation.py` | CNN Encoder-Decoder가 CSI 64ch→Visual 256ch 변환 |
| DensePose 동작 | `v1/src/models/densepose_head.py` | 듀얼헤드: 14+1 body parts + UV좌표 |
| 생체신호 추출 | `docs/adr/ADR-021-*` | 밴드패스 필터: 호흡(0.1-0.5Hz), 심박(0.8-2.0Hz) |
| 엣지 처리 계층 | `docs/adr/ADR-039-*` | Tier 0/1/2 엣지 인텔리전스 |

**판단 기준**: CSI → 특징 추출 → 신경망 → 자세 출력의 전체 흐름을 이해하면 통과

### 1.4 Rust 코어 단위 테스트 실행

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| 전체 테스트 | `cd rust-port/wifi-densepose-rs && cargo test --workspace --no-default-features` | 1,031+ passed, 0 failed |
| signal 크레이트 | `cargo test -p wifi-densepose-signal --no-default-features` | RuvSense 14모듈 테스트 통과 |
| vitals 크레이트 | `cargo test -p wifi-densepose-vitals --no-default-features` | 호흡/심박 알고리즘 검증 |
| mat 크레이트 | `cargo test -p wifi-densepose-mat --no-default-features` | 재난 대응 로직 검증 |

**판단 기준**: 핵심 크레이트 테스트 전체 통과 시 알고리즘 구현 무결성 확인

### 1.5 Observatory 3D 시각화 체험

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| Observatory 접속 | 브라우저에서 `http://localhost:3000/observatory.html` | Three.js 3D 씬 렌더링 |
| 시나리오 전환 | 우측 상단 시나리오 셀렉터 | 13개 시나리오 (의료, 재난, 보안 등) |
| 생체신호 HUD | 좌측 패널 확인 | HR, BR, Confidence 실시간 표시 |
| 스타일 프리셋 | 설정 → 렌더링 탭 | 7가지 시각화 스타일 |

**판단 기준**: 시뮬레이션 데이터로 3D 자세 + 생체신호가 실시간 표시되면 통과

---

## Phase 2. 전체 기능 테스트

> 목표: "각 기능이 스펙대로 동작하는가?" — 모든 서브시스템을 체계적으로 검증

### 2.1 센싱 서버 REST API 검증

| 엔드포인트 | 메서드 | 테스트 방법 | 기대 결과 |
|-----------|--------|-----------|----------|
| `/api/v1/status` | GET | `curl localhost:3000/api/v1/status` | JSON 상태 응답 |
| `/api/v1/models` | GET | `curl localhost:3000/api/v1/models` | 모델 목록 |
| `/api/v1/models/active` | GET | | 활성 모델 정보 |
| `/api/v1/recording/start` | POST | | CSI 녹화 시작 |
| `/api/v1/recording/stop` | POST | | 녹화 중지 + .jsonl 저장 |
| `/api/v1/recording/list` | GET | | 녹화 파일 목록 |
| `/api/v1/train/start` | POST | | 학습 시작 |
| `/api/v1/train/status` | GET | | 학습 진행률 |

**판단 기준**: 14개 REST 엔드포인트 모두 정상 응답

### 2.2 WebSocket 실시간 스트리밍 검증

| 항목 | 테스트 방법 | 확인 포인트 |
|------|-----------|------------|
| WS 연결 | `wscat -c ws://localhost:3001/ws` | 연결 성공, 프레임 수신 |
| 자세 스트리밍 | UI Live Demo 탭에서 Start 클릭 | 스켈레톤 실시간 렌더링 |
| 재연결 | 서버 재시작 후 자동 재연결 | 지수 백오프로 복구 |
| 다중 클라이언트 | 브라우저 2개 동시 접속 | 양쪽 모두 수신 |

**판단 기준**: WebSocket 연결/스트리밍/재연결이 안정적이면 통과

### 2.3 Python v1 파이프라인 검증

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| 전체 테스트 | `cd v1 && python -m pytest tests/ -x -q` | 전체 통과 |
| API 서버 기동 | `wifi-densepose start` 또는 Docker python 이미지 | 8080 포트 응답 |
| 추론 파이프라인 | `pytest tests/integration/test_inference_pipeline.py` | CSI→자세 변환 성공 |
| CSI 파이프라인 | `pytest tests/integration/test_csi_pipeline.py` | 신호처리 무결성 |

**판단 기준**: Python v1 테스트 전체 통과 + 추론 파이프라인 정상 동작

### 2.4 UI 기능 전수 검증

| 탭 | 검증 항목 | 확인 포인트 |
|----|----------|------------|
| Dashboard | 시스템 상태, CPU/메모리, 감지 통계 | 실시간 갱신 |
| Hardware | 3×3 안테나 배열 시각화 | CSI 진폭/위상 바 렌더링 |
| Live Demo | 자세 스켈레톤 렌더링 | Start/Stop, 디버그 모드 |
| Sensing | RSSI 스파크라인, 존재/모션 | Gaussian splat 3D |
| Training | CSI 녹화, 모델 관리 | 녹화 시작/중지 |
| Settings | 연결/감지/렌더링/성능 설정 | localStorage 저장 |

### 2.5 Pose-Fusion 듀얼모달 검증

| 모드 | 테스트 방법 | 확인 포인트 |
|------|-----------|------------|
| DUAL | `pose-fusion.html` 접속 | Video + CSI 퓨전 자세 |
| VIDEO | Video 모드 전환 | 웹캠 DensePose |
| CSI | CSI 모드 전환 | 신호 기반 자세 |
| 임베딩 | 우측 패널 PCA 투영 | 128D 임베딩 시각화 |

### 2.6 WASM 브라우저 검증

| 항목 | 실행 방법 | 확인 포인트 |
|------|----------|------------|
| WASM 빌드 | `cd rust-port/wifi-densepose-rs && cargo build -p wifi-densepose-wasm --target wasm32-unknown-unknown` | 빌드 성공 |
| 브라우저 로드 | WASM 모듈 브라우저 임포트 | MAT 기능 동작 |

### 2.7 성능 벤치마크 측정

| 메트릭 | 실행 방법 | 목표값 |
|--------|----------|--------|
| Rust 전체 파이프라인 | `cargo bench` (criterion) | <20µs |
| Python 전체 파이프라인 | `pytest tests/integration/test_inference_speed.py` | <15ms |
| API 응답 시간 | `curl -w "%{time_total}" localhost:3000/api/v1/status` | <100ms |
| WebSocket 지연 | UI FPS 카운터 확인 | >30fps |

**판단 기준**: Rust <20µs, Python <15ms, API <100ms 달성 시 상용 성능 충분

---

## Phase 3. 커스터마이징 테스트

> 목표: "우리 사업 요구에 맞게 변형 가능한가?" — 확장성과 적용 가능성 검증

### 3.1 센싱 파라미터 커스터마이징

| 항목 | 변경 위치 | 테스트 내용 |
|------|----------|-----------|
| 호흡 감지 대역 | `edge_processing.c` → Biquad 계수 | 0.1-0.5Hz → 0.05-0.8Hz 확장 가능성 |
| 심박 감지 대역 | `edge_processing.c` → Biquad 계수 | 0.8-2.0Hz → 0.5-3.0Hz 확장 가능성 |
| 낙상 임계값 | NVS config `fall_thresh` | 15.0 rad/s² 기본값 조정 효과 |
| 존재 감지 민감도 | Sensing server 설정 | 임계값 변경 시 오탐/미탐 비율 |
| 서브캐리어 선택 | `subcarrier_selection.rs` | Top-K 수 변경 → 정확도 vs 속도 |

**판단 기준**: 주요 파라미터 3개 이상 변경 후 동작 확인

### 3.2 UI/UX 커스터마이징

| 항목 | 변경 위치 | 테스트 내용 |
|------|----------|-----------|
| 브랜딩 변경 | `ui/index.html`, `ui/style.css` | 로고, 색상, 제목 교체 |
| 대시보드 레이아웃 | `ui/components/DashboardTab.js` | 메트릭 패널 추가/삭제 |
| Observatory 테마 | `ui/observatory/` 설정 | 커스텀 스타일 프리셋 추가 |
| 알림 커스터마이징 | `ui/services/` | 낙상/이상 감지 시 외부 알림 연동 |
| 한글 로캘 | UI 전체 | i18n 적용 가능성 확인 |

**판단 기준**: 브랜딩 + 레이아웃 변경이 30분 내 가능하면 통과

### 3.3 API 확장 테스트

| 항목 | 변경 위치 | 테스트 내용 |
|------|----------|-----------|
| 커스텀 엔드포인트 | `sensing-server/src/` | 신규 REST 엔드포인트 추가 |
| 외부 시스템 연동 | API 클라이언트 작성 | 3rd-party 대시보드(Grafana 등) 연동 |
| 웹훅 알림 | 이벤트 발생 시 HTTP POST | 낙상/이상 이벤트 → Slack/카카오톡 |
| 데이터 내보내기 | 녹화 데이터 `.jsonl` | CSV/JSON 변환 및 분석 도구 연동 |

**판단 기준**: 커스텀 엔드포인트 1개 + 웹훅 1개 추가 가능하면 통과

### 3.4 하드웨어 구성 변형 테스트

| 구성 | 장비 | 비용 | 테스트 내용 |
|------|------|------|-----------|
| 최소 구성 | ESP32-S3 1개 | ~$9 | 단일 노드 존재감지 + 호흡 |
| 기본 구성 | ESP32-S3 2개 | ~$18 | 듀얼 노드 교차 검증 |
| 풀 구성 | ESP32-S3 4개 + mmWave | ~$51 | 멀티스태틱 메쉬 + 바이탈 퓨전 |
| 디스플레이 추가 | + OLED/AMOLED | ~$5-15 | 현장 표시 (HR/BR/존재) |

**판단 기준**: 최소 구성($9)으로 존재감지가 동작하면 MVP 가능

### 3.5 WASM 모듈 커스터마이징

| 항목 | 테스트 내용 | 확인 포인트 |
|------|-----------|------------|
| 커스텀 알고리즘 | C/Rust → WASM 컴파일 후 RVF 패키징 | ESP32에서 핫로딩 |
| 능력 제한 | RVF manifest의 capabilities 비트마스크 조정 | 특정 API만 허용 |
| CPU 예산 | `max_frame_us` 설정 변경 | 처리 시간 제한 효과 |
| 서명 검증 | Ed25519 키쌍 생성 후 서명 | 무결성 검증 동작 |

**판단 기준**: 커스텀 WASM 모듈 1개를 작성하여 ESP32에서 실행 가능하면 통과

### 3.6 도메인 특화 적용 테스트

| 도메인 | 커스터마이징 내용 | 핵심 검증 |
|--------|----------------|----------|
| **요양원/병원** | 낙상 감지 임계값 + 간호사 알림 | 오탐률 <5%, 알림 지연 <5초 |
| **스마트홈** | 존재 감지 + HVAC 연동 | 방별 재실 판단 정확도 |
| **호텔** | 객실 점유율 + 행복도 스코어링 | 체크인/아웃 자동 감지 |
| **피트니스** | 운동 자세 추정 + 반복 횟수 | 17-keypoint 정확도 |
| **재난대응** | MAT 트리아지 + 생존자 탐지 | 벽 투과 감지 거리 |

**판단 기준**: 1개 이상 도메인에서 PoC 데모 구성 가능하면 통과

---

## 실행 일정 (권장)

| 단계 | 기간 | 전제 조건 | 산출물 |
|------|------|----------|--------|
| **Phase 1.1-1.3** | 1-2일 | Docker 환경 | 원리 이해 보고서 |
| **Phase 1.4-1.5** | 1일 | Rust 1.85+ | 테스트 결과 로그 |
| **Phase 2.1-2.4** | 2-3일 | Docker + Node.js | 기능 체크리스트 |
| **Phase 2.5-2.7** | 1-2일 | WASM toolchain | 성능 벤치마크 |
| **Phase 3.1-3.3** | 3-5일 | 개발 환경 | 커스텀 빌드 |
| **Phase 3.4** | 2-3일 | ESP32-S3 하드웨어 | 하드웨어 PoC |
| **Phase 3.5-3.6** | 3-5일 | 도메인 요구사항 정의 | 도메인 PoC 데모 |
| **합계** | **약 2-3주** | | 사업화 타당성 보고서 |

---

## Go/No-Go 판단 기준

### Phase 1 통과 조건 (개념 검증)
- [ ] Docker 시뮬레이션으로 UI 정상 동작 확인
- [ ] Python 증명 검증 PASS
- [ ] Rust 테스트 1,031+ 통과
- [ ] CSI → 자세 변환 원리 이해

### Phase 2 통과 조건 (기능 검증)
- [ ] REST API 14개 엔드포인트 정상
- [ ] WebSocket 스트리밍 안정 (>30fps)
- [ ] 성능: Rust <20µs, API <100ms
- [ ] WASM 빌드 성공

### Phase 3 통과 조건 (사업화 검증)
- [ ] 파라미터 커스터마이징 3개+ 성공
- [ ] UI 브랜딩 변경 30분 내 완료
- [ ] 커스텀 API 엔드포인트 추가 성공
- [ ] 최소 하드웨어($9)로 존재감지 동작
- [ ] 1개 이상 도메인 PoC 데모 완성

### 최종 판단
- **Phase 1만 통과**: 기술적 가능성 확인 → 추가 연구 필요
- **Phase 2까지 통과**: 기능적 완성도 확인 → 하드웨어 투자 검토
- **Phase 3까지 통과**: 사업화 타당성 확인 → 사업 계획 수립 진행

---

## 참고: 필요 환경

### 소프트웨어 (Phase 1-2)
- Docker Desktop 또는 Docker Engine
- Rust 1.85+ (cargo, rustup)
- Python 3.10+ (pip, venv)
- Node.js 18+ (npm)
- 최신 웹 브라우저 (Chrome/Firefox)
- wscat (`npm install -g wscat`)

### 하드웨어 (Phase 3.4+)
- ESP32-S3 DevKit (8MB flash) × 1-4개 — 약 $9-36
- USB-C 케이블
- (선택) ESP32-C6 + Seeed MR60BHA2 — 약 $15
- (선택) OLED 디스플레이 128×64 — 약 $5
- WiFi AP (2.4GHz/5GHz)
