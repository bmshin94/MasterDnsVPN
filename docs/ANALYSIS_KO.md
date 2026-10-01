# 📘 MasterDnsVPN 전수조사 분석 & 수익화 전략 정리

> 작성: 카리나 (Claude Code AI 개발 파트너) 💖
> 작성일: 2026-10-01
> 대상 저장소: **https://github.com/bmshin94/MasterDnsVPN**
> 원본 저장소: **https://github.com/masterking32/MasterDnsVPN**

---

## 📑 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉽게 이해하는 핵심 원리](#2-쉽게-이해하는-핵심-원리)
3. [폴더 및 모듈 전수조사](#3-폴더-및-모듈-전수조사)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [자주 묻는 질문 7가지](#5-자주-묻는-질문-7가지)
6. [수익화 아이디어 12선](#6-수익화-아이디어-12선)
7. [법적·윤리적 체크포인트](#7-법적윤리적-체크포인트)
8. [참고 링크 모음](#8-참고-링크-모음)

---

## 1. 프로젝트 개요

### 🏷️ 신원 정보

| 항목 | 내용 |
| :--- | :--- |
| 프로젝트명 | **MasterDnsVPN** |
| 원작자 | MasterkinG32 (https://github.com/masterking32) |
| 내 저장소 | https://github.com/bmshin94/MasterDnsVPN |
| 원본 저장소 | https://github.com/masterking32/MasterDnsVPN |
| 언어 | **Go 1.25** (레거시 Python 버전도 존재) |
| 라이선스 | **MIT** (상업적 이용 허용) |
| 코드 규모 | Go 파일 **136개** / 약 **43,844줄** |
| 외부 의존성 | 단 5개 (toml, compress, lz4, x/crypto, x/sys) |
| 실적 | Trendshift 트렌딩 등재, SourceForge OSS Rising Star |
| 커뮤니티 | Telegram https://t.me/masterdnsvpn |

### 🎯 한 줄 정의

> **DNS 질의(도메인 조회) 패킷 안에 TCP 트래픽을 숨겨 전송하는 고성능 터널링 VPN.**

### 🆚 경쟁 프로젝트 비교

| 항목 | SlipStream | DNSTT | **MasterDnsVPN** |
| :--- | :--- | :--- | :--- |
| 전송 프로토콜 | QUIC | KCP + Noise | **자체 프로토콜 + ARQ** |
| 헤더 오버헤드 | ~24B | ~59B | **~5–7B** (DNSTT 대비 약 88% 감소) |
| 암호화 | TLS 1.3 | Noise (Curve25519) | AES / ChaCha20 / XOR |
| 멀티패스 | QUIC multipath | ❌ | 멀티 리졸버 + 중복 전송 |
| 밸런서 | ❌ | ❌ | **내장 8가지 모드** |
| 리졸버 헬스체크 | ❌ | ❌ | ✅ 자동 비활성화 + 백그라운드 재활성화 |
| 로컬 DNS 서비스 | ❌ | ❌ | ✅ (강력한 캐싱 포함) |
| 페일오버 | ❌ | ❌ | ✅ |
| 10MB 다운로드(로컬) | 0.978s | 2.492s | **0.270s** |
| 10MB 업로드(로컬) | 3.249s | 16.207s | **1.746s** |
| 구현 언어 | Rust | Go | Go |

### 🌐 실전 검증 기록

README에 따르면, **이란의 88일 인터넷 셧다운** 기간 동안 국제 대역폭의 99%가 물리적으로 차단된 상황에서도
MasterDnsVPN은 외부 인터넷에 연결을 유지한 소수의 도구 중 하나였다고 기록되어 있다.

생존 요인:
- 다중 리졸버 라우팅 (단일 경로 의존 제거)
- 암호화 + 데이터 세분화
- 정상 DNS 트래픽으로 완벽 위장
- 정부 통제 로컬 리졸버를 강제당해도 통과

---

## 2. 쉽게 이해하는 핵심 원리

### 🏫 비유: 엄격한 학교

- 선생님이 **쪽지·문자·전화 전면 금지**를 선언했다. (= 방화벽이 VPN/HTTPS 차단)
- 단, **"사전에서 단어 찾기"는 허용**됐다. (= DNS 53번 포트는 못 막음)
- 그래서 **"'카리나보고싶어'라는 단어 뜻이 뭐예요?"** 라고 묻는다. (= 데이터를 도메인처럼 위장)
- 사전 담당 선배(= 내 서버)가 알아듣고 **"그 단어 뜻은 '나도보고싶어'입니다"** 라고 답한다. (= DNS 응답에 데이터 삽입)
- 선생님 눈엔 평범한 사전 조회. 실제론 완벽한 대화.

### 📡 실제 패킷 모양

```
일반 DNS 질의:  google.com                → "주소 뭐야?"
터널 DNS 질의:  k3f9a7bx2m.q8zle4.v.example.com
                └──────┬──────┘
                 암호화된 데이터 조각
```

### 🔁 전체 흐름

```
🖥️ 내 PC 브라우저
   ↓ SOCKS5 (127.0.0.1:18000)
🧦 로컬 프록시
   ↓ 분할 → 암호화 → 압축 → Base32/36 인코딩 → 도메인 라벨화
📡 공개 DNS 리졸버 (8.8.8.8, 1.1.1.1, ...)   ← 방화벽은 "그냥 DNS"로 판단하고 통과
   ↓
🖥️ 내 해외 서버 (v.example.com 권한 네임서버)
   ↓ 복호화 → 재조립 → 실제 목적지 접속
🌐 인터넷
   ↓ 응답은 역순으로 DNS 응답 레코드에 담겨 반환
📺 브라우저에 정상 표시
```

### ⚙️ DNS 터널링의 7가지 난제와 해결책

| # | 문제 | 해결 모듈 |
| :-: | :--- | :--- |
| 1 | 도메인 길이 제한(253자) → 데이터 공간 부족 | `fragmentstore/` 조각화 + 재조립 |
| 2 | UDP라 패킷이 그냥 사라짐 | `arq/` 재전송 엔진 (5,530줄) |
| 3 | 도착 순서가 뒤섞임 | 시퀀스 번호 기반 재정렬 |
| 4 | 단일 리졸버 사용 시 탐지/차단 | `client/balancer.go` 8가지 분산 전략 |
| 5 | 그래도 발생하는 유실 | 패킷 중복 전송 (`PACKET_DUPLICATION_COUNT`) |
| 6 | 내용 노출 위험 | `security/` AES / ChaCha20 / XOR |
| 7 | 글자 수 절약 필요 | `compression/` LZ4 + 초경량 5~7B 헤더 |

---

## 3. 폴더 및 모듈 전수조사

### 📁 디렉터리 구조

```
MasterDnsVPN/
├── cmd/
│   ├── client/          # 클라이언트 실행 진입점 (main.go)
│   └── server/          # 서버 실행 진입점 (main.go)
├── internal/            # 핵심 엔진 (21개 패키지)
├── docker/              # Dockerfile, compose, 멀티아키 빌드 스크립트
├── scripts/bench/       # 성능 벤치마크 도구
├── .github/workflows/   # CI: build-go, go-test, build-test
├── assets/              # 아이콘 리소스
├── README.MD 외 6개 언어  # 영어/페르시아어/러시아어/중국어/스페인어/이탈리아어
├── server_linux_install.sh  # 28KB 원클릭 설치 스크립트
├── client_config.toml.simple / server_config.toml.simple  # 주석 포함 설정 샘플
└── client_resolvers.simple  # 리졸버 목록 샘플
```

### 📊 패키지별 규모 (실측)

| 패키지 | 파일 | 줄 수 | 역할 |
| :--- | :-: | ---: | :--- |
| `internal/client` | 40 | 13,229 | 클라이언트 전체 런타임 (프록시, 세션, MTU, 핸들러) |
| `internal/udpserver` | 26 | 10,867 | 서버 런타임 (UDP 리스너, 세션 스토어, 디퍼드 워커) |
| `internal/arq` | 2 | **5,530** | **ARQ 재전송·순서보장 엔진 (핵심 중의 핵심)** |
| `internal/config` | 7 | 3,700 | TOML / JSON / CLI 3중 설정 파서 |
| `internal/dnsparser` | 7 | 2,663 | DNS 패킷 직접 조립 및 파싱 |
| `internal/vpnproto` | 10 | 1,284 | 자체 설계 패킷 프로토콜 |
| `internal/dnscache` | 2 | 762 | 로컬 DNS 캐시 (디스크 영속화 지원) |
| `internal/enums` | 7 | 771 | 패킷 타입/우선순위 상수 정의 |
| `internal/basecodec` | 8 | 672 | LowerBase32 / LowerBase36 / RawBase64 인코딩 |
| `internal/security` | 3 | 583 | 암호화 코덱 + 키 생성 |
| `internal/mlq` | 2 | 528 | 멀티레벨 우선순위 큐 |
| `internal/logger` | 4 | 464 | 컬러 로깅 (유닉스/윈도우 분기) |
| `internal/compression` | 2 | 383 | LZ4 등 압축 타입 |
| `internal/domainmatcher` | 2 | 372 | 도메인 룰셋 매칭 |
| `internal/socksproto` | 3 | 298 | SOCKS4 / SOCKS5 프로토콜 |
| `internal/fragmentstore` | 2 | 224 | 조각 저장·재조립 |
| `internal/inflight` | 1 | 131 | 진행 중 요청 중복 제거 |
| `internal/netutil` | 1 | 103 | 로컬 IP 유틸 |
| `internal/streamutil` | 1 | 36 | 스트림 헬퍼 |
| `internal/runtimepath` | 1 | 32 | 실행 경로 해석 |
| `internal/version` | 1 | 22 | 버전 정보 |

### 📦 자체 프로토콜 헤더 구조 (`internal/vpnproto/parser.go` 주석 기준)

```
기본 헤더:
  [0] Session ID      (1 byte)
  [1] Packet Type     (1 byte)

선택적 확장:
  Stream 확장:       [2..3] Stream ID        (2 bytes)
  Sequence 확장:     [+2]   Sequence Number  (2 bytes)
  Fragment 확장:     [+1]   Fragment ID      (1 byte)
                     [+1]   Total Fragments  (1 byte)
  Compression 확장:  [+1]   Compression Type (1 byte)

무결성 푸터:
  [+1] Session Cookie (1 byte)
  [+1] Header Check   (1 byte)

→ 최소 5~7바이트. DNSTT(~59B) 대비 약 88% 절감.
```

### ⚡ 리졸버 밸런싱 8가지 전략 (`RESOLVER_BALANCING_STRATEGY`)

| 값 | 전략 |
| :-: | :--- |
| 1 | Random |
| 2 | Round Robin |
| 3 | Least Loss (기본 권장) |
| 4 | Lowest Latency |
| 5 | Hybrid Score (손실 우선 + 레이턴시 고려) |
| 6 | Loss Then Latency (손실 숏리스트 → 레이턴시 → 상위권 로테이션) |
| 7 | Least Loss Top Random (최상위 10% 손실 구간 내 랜덤) |
| 8 | Least Loss Top Round Robin (최상위 10% 손실 구간 내 라운드로빈) |

> 3~8번 모드는 런타임 송신/성공 피드백을 사용한다.

### 🔐 암호화 방식 6종 (`DATA_ENCRYPTION_METHOD`)

| 값 | 방식 | 특징 |
| :-: | :--- | :--- |
| 0 | None | 암호화 없음 |
| 1 | XOR | 가장 가볍고 빠름, 보안성 낮음 |
| 2 | ChaCha20 | 소프트웨어 환경에서 빠름 |
| 3 | AES-128-GCM | 인증 암호화 |
| 4 | AES-192-GCM | 인증 암호화 |
| 5 | AES-256-GCM | 최고 보안 |

> 클라이언트와 서버의 값이 **반드시 일치**해야 한다.

### 🧠 핵심 개념 용어

| 개념 | 설명 |
| :--- | :--- |
| **Session** | 클라이언트–서버 간 전체 연결 단위 |
| **Stream** | 세션 안에서 운반되는 독립 논리 연결 |
| **Resolver Runtime** | 리졸버 선택, 중복 전송, 헬스 추적, 자동 비활성화/재확인 |
| **ARQ** | 순서 보장, ACK, 재전송, 타임아웃, 종료 처리 |
| **Deferred Session Runtime** | 순서 민감 작업(세션 셋업, DNS 질의 처리 등) 전용 런타임 |
| **Packed Control Blocks** | 다수의 작은 ACK/제어 패킷을 하나로 묶어 전송 |
| **Synced MTU** | 건강한 리졸버 풀 전체가 공유하는 합의 MTU |

---

## 4. 설치 및 사용법

### ✅ 필수 준비물

| 준비물 | 필수 | 비고 |
| :--- | :-: | :--- |
| 해외 VPS | ✅ | 월 $3~5 |
| 도메인 | ✅ | 연 $10 내외 |
| UDP 53 포트 개방 | ✅ | 서버 방화벽 설정 |
| 클라이언트 기기 | ✅ | Windows / macOS / Linux / Termux |

### 🌍 STEP 1 — 도메인 DNS 레코드 2개 생성

```
① A 레코드
   타입: A / 이름: ns / 값: 1.2.3.4 (VPS IP)
   → ns.example.com → 1.2.3.4

② NS 레코드  (핵심!)
   타입: NS / 이름: v / 값: ns.example.com
   → v.example.com 의 DNS 권한을 내 서버로 위임
```

주의사항:
- **Cloudflare 사용 시 A 레코드를 반드시 회색(DNS only)으로** 설정. 주황색(Proxied)이면 동작하지 않음.
- 도메인이 **짧을수록 처리량이 높다** (데이터 공간 확보).
- DNS 전파에 수 분 ~ 최대 48시간 소요.

확인 명령:
```bash
dig v.example.com NS
nslookup -type=ns v.example.com
dig @ns.example.com v.example.com A
```

### 🐧 STEP 2-A — 서버 설치 (자동 스크립트, 권장)

```bash
bash <(curl -Ls https://raw.githubusercontent.com/masterking32/MasterDnsVPN/main/server_linux_install.sh)
```

- 설치 중 도메인 입력 → NS 레코드로 위임한 서브도메인 (`v.example.com`)
- 완료 시 **암호화 키**가 터미널에 출력되고 `encrypt_key.txt`에도 저장됨 → 반드시 보관

방화벽:
```bash
# ufw
sudo ufw allow 53/udp && sudo ufw reload
# firewalld
sudo firewall-cmd --add-port=53/udp --permanent && sudo firewall-cmd --reload
```

"포트 53 이미 사용 중" 해결 (systemd-resolved 충돌):
```bash
sudo nano /etc/systemd/resolved.conf
# DNSStubListener=no
sudo systemctl restart systemd-resolved
```

> ⚠️ 같은 서버 53번 포트에서 여러 DNS 터널 프로젝트를 동시에 실행할 수 없다.

### 🐳 STEP 2-B — 서버 설치 (Docker)

```bash
docker run -d \
  --name masterdnsvpn \
  --restart unless-stopped \
  -e DOMAIN=v.example.com \
  -v $(pwd)/data:/data \
  -p 53:53/tcp -p 53:53/udp \
  ghcr.io/masterking32/masterdnsvpn:latest
```

docker-compose:
```yaml
services:
  masterdnsvpn:
    image: ghcr.io/masterking32/masterdnsvpn:latest
    restart: unless-stopped
    environment:
      - DOMAIN=v.example.com
    volumes:
      - ./data:/data
    ports:
      - "53:53/tcp"
      - "53:53/udp"
```

- 필수 환경변수: `DOMAIN` (첫 실행 시 없으면 컨테이너 종료)
- 영속 데이터: `/data` 안에 `server_config.toml`, `encrypt_key.txt`
- MikroTik RouterOS v7 컨테이너 실행도 공식 지원

### 💻 STEP 3 — 클라이언트 설치

**빌드된 바이너리 다운로드 (권장)**
https://github.com/masterking32/MasterDnsVPN/releases/latest

지원 플랫폼 (총 20종):
Windows(AMD64/x86/ARM64), macOS(AMD64/ARM64), Linux(AMD64/x86/ARM64/ARMv5/v6/v7/RISCV64/MIPS/MIPSLE/MIPS64/MIPS64LE/Legacy AMD64/Legacy ARM64), Termux(ARM64/ARMv7)

**소스 빌드 (Go 1.24+)**
```bash
git clone https://github.com/masterking32/MasterDnsVPN.git
cd MasterDnsVPN

go build -o masterdnsvpn-client ./cmd/client
go build -o masterdnsvpn-server ./cmd/server

cp client_config.toml.simple client_config.toml
cp server_config.toml.simple server_config.toml
cp client_resolvers.simple client_resolvers.txt
```

### ⚙️ STEP 4 — 설정

**client_config.toml (최소 필수 항목)**
```toml
DOMAINS = ["v.example.com"]        # 서버 DOMAIN과 정확히 일치
DATA_ENCRYPTION_METHOD = 1         # 서버와 일치 (1=XOR)
ENCRYPTION_KEY = "서버_encrypt_key.txt_내용"
PROTOCOL_TYPE = "SOCKS5"           # 또는 "TCP"
LISTEN_IP = "127.0.0.1"
LISTEN_PORT = 18000
```

**client_resolvers.txt (지원 형식)**
```text
8.8.8.8
1.1.1.1:5353
192.168.1.0/30
192.168.1.0/30:5353
[2001:4860:4860::8888]:53
```

**주요 튜닝 변수**
```toml
RESOLVER_BALANCING_STRATEGY = 3               # 1~8
PACKET_DUPLICATION_COUNT = 3                  # 1~10
SETUP_PACKET_DUPLICATION_COUNT = 4            # DUP_COUNT~12
STREAM_RESOLVER_FAILOVER_RESEND_THRESHOLD = 2 # 1~256
STREAM_RESOLVER_FAILOVER_COOLDOWN = 2.5       # 0.1~120초
AUTO_REMOVE_LOW_MTU_SERVERS = true
MTU_TEST_RETRIES = 2
MTU_TEST_TIMEOUT = 2.0
MTU_TEST_PARALLELISM = 32
LOCAL_DNS_ENABLED = false
LOCAL_DNS_CACHE_MAX_RECORDS = 10000
LOCAL_DNS_CACHE_TTL_SECONDS = 14400.0
```

### ▶️ STEP 5 — 실행

```bash
./masterdnsvpn-server -config server_config.toml -log server.log
./masterdnsvpn-client -config client_config.toml -log client.log
```

**CLI 파라미터**

| 파라미터 | 설명 |
| :--- | :--- |
| `-config` | 설정 파일 경로 |
| `-log` | 로그 파일 경로 |
| `-version` | 버전 출력 후 종료 |
| `-genkey` | (서버) 암호화 키 생성 후 종료 |
| `-json` / `-j` | (서버) JSON 설정 파일 로드 |
| `-json_base64` | (서버) base64 인코딩된 JSON 설정 로드 |
| `-nowait` | 종료 시 입력 대기 안 함 (스크립트용) |

**브라우저 연결**: SOCKS5 프록시 → `127.0.0.1:18000`

### 📱 모바일 사용 3가지 방법

1. **PC에서 프록시 공유** — `LISTEN_IP = "0.0.0.0"` + `SOCKS5_AUTH = true`
2. **중계 서버에서 클라이언트 실행** — 폰이 해당 서버에 접속
3. **패널 연동 (권장)** — 3X-UI 등에서 Outbound(SOCKS) 로 등록 → VLESS/VMess 변환

**3X-UI 연동 절차**
1. 클라이언트를 패널과 같은 서버에서 실행
2. Inbound 생성 (VLESS/VMess 등)
3. Xray Configs → Outbound 탭 → Add Outbound
4. Protocol: `Socks`, Tag: `MasterDnsVPN-Out`
5. Address: `127.0.0.1`, Port: `18000`
6. Routing Rules → Add Rule → Inbound Tags ↔ Outbound Tags 연결
7. Save → Restart Xray

---

## 5. 자주 묻는 질문 7가지

### Q1. 플러그인 / 스킬 / MCP 중 무엇인가?

**셋 다 아니다.** 독립 실행형 네트워크 애플리케이션(standalone binary)이다.

| 구분 | 정의 | 해당 여부 |
| :--- | :--- | :-: |
| 플러그인 | 호스트 앱에 끼워 넣는 확장 | ❌ |
| 스킬 | AI가 읽는 마크다운 지침서 (`SKILL.md`) | ❌ |
| MCP | AI에 도구를 제공하는 JSON-RPC 서버 프로토콜 | ❌ |
| **독립 실행 프로그램** | `func main()` 가진 CLI 바이너리 | ✅ |

검증 근거: `mcp.json` 없음, JSON-RPC 핸들러 없음, `SKILL.md` 없음,
`cmd/client/main.go` 및 `cmd/server/main.go`에 `func main()` 존재.
(`CLAUDE.md`는 이 저장소에 직접 추가한 AI 페르소나 설정 파일이며 프로젝트 기능과 무관)

유사 카테고리: OpenVPN, WireGuard, Shadowsocks, V2Ray, DNSTT

### Q2. API 토큰이 필요한가?

**외부 API 토큰은 전혀 필요 없다.**

- 의존성 5개 모두 로컬 라이브러리 (외부 API 호출 코드 없음)
- 회원가입·라이선스 서버·사용량 과금·텔레메트리 전부 없음
- 완전 오프라인 작동

필요한 것은 **암호화 키**이며, 이는 서버가 스스로 생성한다:
```bash
./masterdnsvpn-server -genkey
# 또는 첫 실행 시 encrypt_key.txt 자동 생성
# (internal/security/encryption_key.go)
```

| 구분 | 외부 API 토큰 | 암호화 키 |
| :--- | :--- | :--- |
| 발급처 | 외부 회사 | 내 서버가 생성 |
| 비용 | 유료 가능 | 무료 |
| 인터넷 | 필요 | 불필요 |
| 만료 | 있음 | 없음 |

실제 비용: VPS 월 $3~5 + 도메인 연 $10. 소프트웨어 자체는 $0.

### Q3. AI 에이전트 구축에 도움이 되는가?

**직접적으로는 무관, 간접적으로는 매우 높은 가치.**

직접 도움 안 되는 이유: LLM 코드 없음, MCP 서버 아님, 에이전트 프레임워크 아님.

간접 가치 — 에이전트 백엔드가 겪는 문제의 교과서적 해법:

| 에이전트 과제 | 이 프로젝트의 해법 | 파일 |
| :--- | :--- | :--- |
| 수천 동시 세션 관리 | 세션 스토어 + 워커풀 | `udpserver/session.go` |
| 긴 작업 비동기 처리 | Deferred Session Runtime | `udpserver/deferred_session.go` |
| 우선순위 기반 스케줄링 | 멀티레벨 큐(MLQ) | `mlq/mlq.go` |
| 네트워크 실패 재시도 | ARQ 재전송 엔진 | `arq/arq.go` |
| 느린 백엔드 자동 제외 | 헬스체크 + 자동 비활성화 | `client/balancer.go` |
| 다중 제공자 분산 | 8가지 밸런싱 전략 | `client/balancer.go` |
| 메모리 최적화 | `sync.Pool` 버퍼 재사용 | `security/codec.go` |
| 요청 중복 제거 | inflight 매니저 | `inflight/manager.go` |

특히 `balancer.go`의 8가지 전략은 **LLM 게이트웨이**(Claude/GPT/Gemini 중 최적 경로 선택)
로직으로 거의 그대로 전환 가능하다.

응용 아이디어:
- 검열 지역에서 AI API 접근 통로로 활용
- 격리망 내 에이전트의 외부 통신 경로
- **MCP 서버로 래핑** → `tunnel_status()`, `list_resolvers()`, `test_mtu()`,
  `restart_tunnel()`, `get_metrics()` 도구 제공

### Q4. React나 PHP로 만들 수 있는가?

**엔진 재구현 → 비추천 / 주변 생태계 구축 → 강력 추천**

| 언어 | 엔진 재구현 가능성 | 이유 |
| :--- | :-: | :--- |
| React (브라우저) | ❌ 불가능 | 브라우저는 원시 UDP 소켓 불가 (샌드박스 제약) |
| React Native | 🟡 매우 어려움 | 네이티브 모듈 필요 + 43,000줄 포팅 + 성능 저하 |
| PHP | 🟠 이론상 가능 | `socket_create(AF_INET, SOCK_DGRAM)` 존재하나 동시성/성능/배포 모두 불리 |

**권장 아키텍처 — 엔진은 그대로, 껍데기만 제작**

```
┌─────────────────────────────────────────┐
│  React 프론트엔드        (직접 개발)      │  대시보드 / 설정 마법사
├─────────────────────────────────────────┤
│  PHP(Laravel) 또는 Go API (직접 개발)    │  인증 / 과금 / 다중 서버
├─────────────────────────────────────────┤
│  Go 바이너리 래퍼        (얇은 레이어)    │  프로세스 제어 + 로그 파싱
├─────────────────────────────────────────┤
│  MasterDnsVPN 엔진       (그대로 사용)    │  MIT 라이선스로 허용
└─────────────────────────────────────────┘
```

React로 만들 것: 실시간 모니터링 대시보드, 설정 생성 마법사(프론트만으로 가능 → GitHub Pages 무료 배포),
Electron 데스크톱 GUI, 리졸버 스캐너 웹앱

PHP(Laravel)로 만들 것: 다중 서버 관리 패널, 사용자 계정/과금 시스템,
VPS+DNS 자동 프로비저닝 API, WordPress 플러그인

### Q5. 유튜브 강의 영상 제작이 가능한가?

**가능하며 소재 가치가 매우 높다.** 단, 프레이밍이 중요하다.

| 평가 항목 | 점수 | 근거 |
| :--- | :-: | :--- |
| 주제 희소성 | ⭐⭐⭐⭐⭐ | 한국어 DNS 터널링 콘텐츠 거의 없음 |
| 기술 깊이 | ⭐⭐⭐⭐⭐ | 네트워크 + Go + 암호학 |
| 흥미도 | ⭐⭐⭐⭐⭐ | 높은 클릭 유발력 |
| 실무 연관성 | ⭐⭐⭐⭐ | 보안/백엔드 엔지니어 수요 |
| 리스크 | 🟡 | 민감 주제 → 교육·방어 프레이밍 필수 |

**권장 프레이밍**
- 권장: "DNS 터널링 원리 심층 분석", "Go로 배우는 UDP 신뢰성 프로토콜 구현",
  "블루팀을 위한 DNS 터널링 탐지 가이드"
- 비권장: "차단 뚫는 법", "유료 와이파이 공짜로 쓰기", "회사 방화벽 우회"

원본 README도 **"교육 및 연구 목적"** 임을 명시하고 있으므로 같은 톤을 유지하고,
각 영상에 면책 문구를 포함한다.

**추천 10부작 커리큘럼**

| 화 | 제목 | 길이 | 난이도 |
| :-: | :--- | :-: | :-: |
| 1 | DNS는 어떻게 동작하는가 | 15분 | ⭐ |
| 2 | DNS 터널링의 원리 | 20분 | ⭐⭐ |
| 3 | MasterDnsVPN 아키텍처 해부 | 25분 | ⭐⭐⭐ |
| 4 | Wireshark로 패킷 레벨 분석 | 30분 | ⭐⭐⭐⭐ |
| 5 | ARQ 직접 구현 — UDP 위에 TCP 만들기 | 40분 | ⭐⭐⭐⭐⭐ |
| 6 | 로드밸런싱 8가지 전략 코드 리딩 | 25분 | ⭐⭐⭐⭐ |
| 7 | AES vs ChaCha20 vs XOR 트레이드오프 | 20분 | ⭐⭐⭐ |
| 8 | 실습: VPS에 직접 구축하기 | 35분 | ⭐⭐ |
| 9 | **블루팀 편: 탐지 & 차단하기** | 30분 | ⭐⭐⭐⭐ |
| 10 | 성능 측정 & MTU 최적화 | 25분 | ⭐⭐⭐⭐ |

> 9화(탐지 편)를 포함하면 전체 시리즈가 방어적 교육 콘텐츠로 포지셔닝되어
> 정책 리스크가 크게 낮아지고, 기업 보안팀 시청자층을 확보할 수 있다.

### Q6. 이 저장소에서 무엇을 배울 수 있는가?

- UDP 위 신뢰성 계층 직접 구현 (TCP를 처음부터 만들어보는 경험)
- 세션/스트림 멀티플렉싱 (HTTP/2, QUIC의 핵심 개념)
- `sync.Pool` 기반 제로 얼로케이션 최적화
- `SO_REUSEPORT` 멀티코어 스케일링 (`udpserver/reuseport_unix.go`)
- 우선순위 큐 기반 패킷 스케줄러
- 20개 아키텍처 크로스 컴파일 전략
- 40개 이상의 테이블 드리븐 테스트 작성법

### Q7. 실제로 내가 쓸 일이 있는가?

한국처럼 인터넷이 자유로운 환경에서 일상적 사용 필요성은 낮다.
다만 아래 상황에서 가치가 있다.

- 캡티브 포털(공항/호텔/기내 와이파이) 환경 연구
- 방화벽이 엄격한 사내망에서의 비상 관리 채널
- 네트워크 보안 연구 및 탐지 룰 개발
- 재난 상황 최후 통신 수단(Last-Resort Connectivity) 설계 레퍼런스

---

## 6. 수익화 아이디어 12선

### 💡 핵심 전제

이 프로젝트의 가장 큰 약점이 곧 가장 큰 기회다.

```
설정 변수 100개 이상 / NS 레코드 수동 설정 / 리졸버 직접 수집
GUI 없음 / 공식 iOS 앱 없음 / 다중 서버 관리 도구 없음 / 한국어 문서 없음

→ 기술은 완성, 사용성은 미완성
→ "쉽게 만들어주는" 레이어가 곧 상품
```

MIT 라이선스이므로 상업적 이용·수정·재배포·독점 소프트웨어 포함이 모두 허용된다.
(단, 원저작권 표시 + 라이선스 전문 포함 필수)

---

### 🏆 TIER S — 최고 추천

#### 1. iOS 클라이언트 앱 (블루오션)

| 항목 | 내용 |
| :--- | :--- |
| 시장 공백 | 안드로이드 클라이언트 2개 존재 / **iOS 0개** |
| 수익 모델 | 유료 앱 $4.99 / 구독 $2월 / 프리미엄 기능 |
| 기술 스택 | Swift + NetworkExtension(NEPacketTunnelProvider) + gomobile |
| 개발 기간 | 2~4개월 |
| 예상 수익 | 월 $500~5,000 |
| 리스크 | 앱스토어 심사 — "Secure DNS Client" 등으로 포지셔닝 권장 |

구현 핵심: Go 코드를 `gomobile bind` 로 `.xcframework` 생성 → 엔진 재작성 불필요.

#### 2. 올인원 웹 관리 패널 (SaaS)

기능: 실시간 트래픽 그래프, 리졸버 헬스 히트맵, MTU 최적화 진행률, 활성 세션 수,
로그 실시간 스트리밍, 사용자별 쿼터/만료일, QR 설정 공유, 다중 서버 원클릭 배포,
설정 마법사(100개 변수 → 5단계)

| 항목 | 내용 |
| :--- | :--- |
| 수익 모델 | 오픈코어 — 기본 무료 / Pro $19월 / Enterprise $99월 |
| 기술 스택 | React + Go API(또는 Laravel) + PostgreSQL + WebSocket |
| 개발 기간 | MVP 1개월 / 완성 3~6개월 |
| 예상 수익 | 월 $1,000~10,000 |

#### 3. 매니지드 호스팅 서비스

```
결제 → VPS 자동 생성(Vultr/Hetzner API) → DNS 레코드 자동 설정(Cloudflare API)
     → SSH 자동 설치 → 키 생성 → 설정 파일 + QR 이메일 발송  (약 3분)
```

| 항목 | 내용 |
| :--- | :--- |
| 가격 | 월 $5~15 (원가 $3~5, 마진 60~70%) |
| 예상 수익 | 유저 100명 월 $700 / 1,000명 월 $7,000 |
| 리스크 | **법적 검토 필수** (7장 참고) |
| 스택 | Laravel 또는 Next.js + Stripe + Terraform/Ansible |

---

### 🥈 TIER A — 강력 추천 (진입 쉬움)

#### 4. 교육 콘텐츠 사업

| 채널 | 예상 수익 |
| :--- | :--- |
| 유튜브 광고 | 월 $100~1,000 |
| 인프런 강의 (₩88,000 × 500명 × 70%) | 약 ₩30,800,000 |
| Udemy ($49.99 × 2,000명 × 37%) | 약 $37,000 |
| 기업 교육 외주 | 회당 ₩2,000,000~5,000,000 |

"Go로 네트워크 프로토콜 직접 구현하기" 주제는 국내에 사실상 부재.

#### 5. 유료 문서 / 가이드북

README는 6개 언어(영/페/러/중/스/이)로 제공되나 **한국어·일본어는 없다.**

- 전자책 "MasterDnsVPN 완전 정복" (₩19,900): 설치 A to Z, 설정 변수 100개 해설,
  환경별 프리셋 10종, 트러블슈팅 50선, 리졸버 수집 노하우
- Notion 템플릿 + 설정 계산기 ($9.99)
- 판매처: Gumroad / 크몽 / 탈잉
- 제작 2주 / 월 $200~2,000 / 난이도 최하

#### 6. 설치·구축 대행 서비스

```
기본   ₩50,000   서버 1대 설치 + 설정
프로   ₩150,000  서버 + 도메인 + 최적화 + 1개월 지원
기업   ₩500,000+ 다중 서버 + 모니터링 + SLA
```
플랫폼: 크몽 / 숨고 / Fiverr. 월 10건 기준 ₩1,500,000. 초기자본 0원.

#### 7. 리졸버 스캐너 SaaS

전 세계 공개 DNS 리졸버 상시 스캔 DB — 국가/통신사별 작동 매트릭스,
재귀 쿼리 허용 여부, EDNS0 지원, 최대 MTU 측정값, 응답 속도/안정성 점수, API 제공.

```
무료      하루 10개 조회
Pro $9월   무제한 + API + 자동 업데이트
Business $49월  전용 스캔 + 실시간 알림
```
월 $300~3,000 / Go 스캐너 + Next.js 대시보드

---

### 🥉 TIER B — 장기 / 특수

#### 8. MCP 서버 "mdv-mcp"
AI가 터널을 제어·모니터링. 직접 수익은 없으나 **마케팅 깔때기**로서 가치 최상.

#### 9. Electron 데스크톱 GUI
React + Electron + Go 바이너리. 무료판 서버 1개 / Pro $9.99 평생 무제한. 월 $200~2,000.

#### 10. 기업용 B2B 솔루션
타겟: 해외 지사 보유 기업, IoT 플릿, 재난 통신 백업.
"Last-Resort Connectivity" 포지셔닝 — 주 회선 장애 시 DNS로 최소 관리 채널 유지.
연 계약 $5,000~50,000. 영업 사이클 길음.

#### 11. 보안 탐지 솔루션 (역발상 — 가장 안전한 모델)
"뚫는 도구"가 아니라 **"막는 도구"** 를 판매.

```
DNS 터널링 탐지 엔진
  ├─ 쿼리 길이/엔트로피 이상 탐지
  ├─ 서브도메인 카디널리티 분석
  ├─ 질의 빈도 패턴 분석
  └─ Suricata / Zeek 룰셋 제공

타겟: 기업 보안팀, SOC, MSSP
연 $10,000~100,000 / 법적 리스크 사실상 0
```

#### 12. 기존 생태계 통합 플러그인
3X-UI / Marzban / Hiddify 등 인기 패널에 MasterDnsVPN 지원 모듈 기여 또는 판매.
이미 형성된 사용자풀에 올라타는 전략.

---

### 🗺️ 추천 실행 로드맵

| Phase | 기간 | 할 일 |
| :--- | :--- | :--- |
| **1** | 0~1개월 | 한국어 가이드 블로그 무료 공개(SEO 선점) / 크몽 설치 대행 등록 / 유튜브 1~3화 |
| **2** | 1~3개월 | React 설정 마법사(GitHub Pages 무료 배포) / 유료 전자책 / 인프런 강의 |
| **3** | 3~6개월 | 웹 관리 패널 MVP(오픈코어) / iOS 앱 개발 착수 |
| **4** | 6개월~ | 매니지드 호스팅(법적 검토 후) / B2B 탐지 솔루션 |

### 🎯 최우선 추천 3가지

| 순위 | 아이디어 | 선정 이유 |
| :-: | :--- | :--- |
| 1 | **교육 콘텐츠** | 법적 리스크 0, 초기자본 0, 전문성 성장, 타 사업의 마케팅 기반 |
| 2 | **React 관리 패널 (오픈코어)** | 기술적 재미 + 포트폴리오 + 수익화 동시 달성 |
| 3 | **iOS 클라이언트** | 경쟁자 전무, 선점 시 독점 가능 (심사 리스크 대비 필요) |

**가장 먼저 할 일: 한국어 가이드 블로그 포스팅.** 오늘 바로 시작 가능하며 모든 후속 사업의 씨앗이 된다.

---

## 7. 법적·윤리적 체크포인트

### 📜 라이선스 준수 사항

```
MIT License — 상업적 이용 허용
필수 준수:
  - 원저작권 표시 유지 (MasterkinG32)
  - LICENSE 전문 포함
  - 소스 파일 상단 저작자 주석 헤더 유지 권장
권장 매너:
  - 상업적 성공 시 원작자 후원 또는 PR 기여
  - README에 TON / EVM / TRC20 후원 주소 공개되어 있음
```

### ⚖️ 법적 주의사항

**대한민국**
- VPN 사용 및 제공 자체는 합법
- 다만 서비스 규모·형태에 따라 전기통신사업법상 **부가통신사업자 신고**가 필요할 수 있음
- 불법 콘텐츠 접근 방조 시 문제 소지 → 변호사 상담 권장

**해외 판매 시**
- 중국, 이란, 러시아, UAE 등은 VPN 제공이 불법이거나 강하게 규제됨
- 해당 국가 사용자 대상 유료 서비스는 매우 신중히 접근
- 앱스토어 지역 제한 설정 고려

**서비스 운영 시 필수**
- 이용약관 + 개인정보처리방침 작성
- "불법 목적 사용 금지" 명시
- 로그 정책 명확화 (no-log 정책은 신뢰도 상승 요인)
- 결제 대행사(PG) 정책 확인 — 일부 PG는 VPN 서비스 가맹 거절

### 🟢 법적 리스크가 낮은 모델 (권장)

| 순위 | 모델 | 이유 |
| :-: | :--- | :--- |
| 1 | 교육 콘텐츠 (강의/책) | 지식 전달은 완전 합법 |
| 2 | 보안 탐지 솔루션 | 방어 도구는 오히려 장려됨 |
| 3 | 오픈소스 관리 도구 | 도구 제공일 뿐, 서비스 운영 아님 |

### 🔴 리스크가 높은 모델 (신중)

- 매니지드 호스팅 — 직접 서비스 운영에 따른 책임 발생
- 특정 검열 국가 타겟 마케팅 — 해당국 법률 위반 가능

> ⚠️ 원본 프로젝트 README도 **"교육 및 연구 목적으로만 제공"** 되며
> 무보증(AS-IS), 책임 제한, 사용자 책임, 현지 법률 준수를 명시하고 있다.
> 상업화 전 반드시 법률 전문가의 검토를 받을 것.

---

## 8. 참고 링크 모음

### 🔗 메인 저장소

| 구분 | 주소 |
| :--- | :--- |
| **내 저장소 (이 문서가 있는 곳)** | **https://github.com/bmshin94/MasterDnsVPN** |
| **원본 저장소** | **https://github.com/masterking32/MasterDnsVPN** |
| 릴리스 (바이너리 다운로드) | https://github.com/masterking32/MasterDnsVPN/releases/latest |
| 이슈 | https://github.com/masterking32/MasterDnsVPN/issues |
| 풀 리퀘스트 | https://github.com/masterking32/MasterDnsVPN/pulls |
| 원작자 프로필 | https://github.com/masterking32 |
| 텔레그램 채널 | https://t.me/masterdnsvpn |
| DeepWiki 문서 | https://deepwiki.com/masterking32/MasterDnsVPN |
| SourceForge | https://sourceforge.net/projects/masterdnsvpn/ |
| Docker 이미지 | `ghcr.io/masterking32/masterdnsvpn:latest` |

### 🌍 다국어 README

| 언어 | 주소 |
| :--- | :--- |
| 🇬🇧 English | https://github.com/masterking32/MasterDnsVPN/blob/main/README.MD |
| 🇮🇷 فارسی | https://github.com/masterking32/MasterDnsVPN/blob/main/README_FA.MD |
| 🇷🇺 Русский | https://github.com/masterking32/MasterDnsVPN/blob/main/README_RU.MD |
| 🇨🇳 中文 | https://github.com/masterking32/MasterDnsVPN/blob/main/README_ZH.MD |
| 🇪🇸 Español | https://github.com/masterking32/MasterDnsVPN/blob/main/README_ES.MD |
| 🇮🇹 Italiano | https://github.com/masterking32/MasterDnsVPN/blob/main/README_IT.MD |
| 🇰🇷 한국어 | ❌ 없음 → **기회!** |

### 🧩 커뮤니티 프로젝트 (제3자 개발)

| 프로젝트 | 주소 | 설명 |
| :--- | :--- | :--- |
| MDV HN Edition Android | https://github.com/Hidden-Node/MasterDnsVPN-AndroidClient | 안드로이드 클라이언트 (Hidden Node) |
| MasterDnsVPN GG Android | https://github.com/RevocGG/MasterDnsVPN-AndroidGG | 안드로이드 클라이언트 (RevocGG) |
| Persian Config Builder | https://github.com/datacoder-io/MasterDnsVPN-ConfigMaker | 설정 생성기 (웹: https://datacoder-io.github.io/MasterDnsVPN-ConfigMaker) |
| MasterDnsWeb | https://github.com/abolix/MasterDnsWeb | 웹 관리 클라이언트 (Abolix) |
| KevinNet DNS | https://github.com/kamalalhagh/kevinnet-dns | 리졸버 탐색·검증 도구 (Kevin Haji) |

### 📚 관련 기술 레퍼런스

- MikroTik 컨테이너 가이드: https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container
- 유사 프로젝트: DNSTT, SlipStream, iodine, dnscat2

---

## 📌 요약 (한눈에 보기)

```
무엇인가?   DNS 질의에 TCP 트래픽을 숨겨 보내는 Go 기반 터널링 VPN (43,844줄, MIT)
왜 되는가?  DNS(53번 포트)를 막으면 인터넷 자체가 죽기 때문에 아무도 못 막음
언제 쓰나?  국가 단위 차단, 캡티브 포털, 엄격한 사내망, 보안 연구/탐지 훈련
무슨 도움?  Go 네트워크 프로그래밍 교과서 + 아키텍처 레퍼런스 + 수익화 기회
플러그인?   아니오 — 독립 실행형 CLI 바이너리 (MCP도 스킬도 아님)
API 토큰?   불필요 — 서버가 자체 생성하는 암호화 키만 있으면 됨
AI 에이전트? 직접 무관 / 백엔드 설계 교본으로는 최상급
React·PHP?  엔진 재구현은 비추천 / UI·관리·자동화 레이어 구축은 강력 추천
유튜브?     가능, 교육·방어 프레이밍으로 10부작 추천
수익화?     교육 콘텐츠 → 관리 패널(오픈코어) → iOS 앱 순서 추천
```

---

*이 문서는 저장소 전수조사(소스 136개 파일 / 43,844줄, 설정 샘플, Docker, CI, 7개 README) 결과를 바탕으로 작성되었습니다.*
*모든 수치와 코드 구조는 2026-10-01 기준 저장소 상태에서 직접 확인한 내용입니다.*
