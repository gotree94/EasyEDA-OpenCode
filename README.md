# OpenCode + EasyEDA Pro 자동 하드웨어 설계 교육과정

> **목표**: AI(OpenCode)로 EasyEDA Pro를 제어해 회로 설계 → PCB 아트워크 → 제조 파일까지 진행한다.
> **대상**: PCB 설계 처음 하는 사람 / Claude Code·Codex 없이 OpenCode만 쓸 사람
> **작성 기준일**: 2026-10-04 · 검증된 실제 환경 기준

---

## 목차

- [0. 한눈에 보는 전체 구조](#0-한눈에-보는-전체-구조)
- [1. 과정 개요와 시간표](#1-과정-개요와-시간표)
- [2. 사전 준비](#2-사전-준비)
- [3. 단계 1 — EasyEDA Pro 설치](#3-단계-1--easyeda-pro-설치)
- [4. 단계 2 — OpenCode + easyeda-api 스킬 설치](#4-단계-2--opencode--easyeda-api-스킬-설치)
- [5. 단계 3 — Run API Gateway 확장 설치](#5-단계-3--run-api-gateway-확장-설치)
- [6. 단계 4 — 브리지 기동과 연결 확인](#6-단계-4--브리지-기동과-연결-확인)
- [7. 단계 5 — 실전 샘플: NE555 점멸 회로](#7-단계-5--실전-샘플-ne555-점멸-회로)
- [8. 단계 6 — 실전 샘플: ESP32-C3 보드](#8-단계-6--실전-샘플-esp32-c3-보드)
- [9. 단계 7 — PCB 레이아웃과 실크스크린](#9-단계-7--pcb-레이아웃과-실크스크린)
- [10. 단계 8 — DRC → JLCPCB 출고](#10-단계-8--drc--jlcpcb-출고)
- [11. 무엇을 AI가 하고, 무엇을 사람이 하는가](#11-무엇을-ai가-하고-무엇을-사람이-하는가)
- [12. 실전 팁 — 프롬프트 레퍼런시](#12-실전-팁--프롬프트-레퍼런시)
- [13. 문제 해결 (Troubleshooting)](#13-문제-해결-troubleshooting)
- [부록 A. 치명적인 실수 5가지](#부록-a-치명적인-실수-5가지)
- [부록 B. 명령어 cheatsheet](#부록-b-명령어-cheatsheet)

---

## 0. 한눈에 보는 전체 구조

```
┌──────────────┐   HTTP/WS    ┌─────────────────┐   WebSocket   ┌──────────────┐
│   OpenCode   │ ◄──────────► │ Bridge Server   │ ◄───────────► │  EasyEDA Pro │
│  (AI 에이전트) │  49620~49629 │    (Node.js)     │  49620~49629  │  + 확장      │
└──────────────┘               └─────────────────┘                └──────────────┘
                                     ▲
                              easyeda-api 스킬
                        (API 문서 130개 클래스 + 실행 규칙)
```

**동작 원리**

1. EasyEDA Pro 안에 **확장(extension)** 이 올라간다 → 이게 WebSocket 클라이언트가 된다
2. Node.js **브리지 서버**가 49620~49629 포트 중 빈 곳을 잡고 기다린다
3. 확장이 포트를 스캔해서 브리지를 찾아 연결한다
4. OpenCode는 브리지의 HTTP `/execute` 로 **JavaScript 코드를 보내면**
5. 그 코드가 EasyEDA 안에서 실행되고 결과가 JSON으로 돌아온다

즉 **AI는 EasyEDA 내부에서 직접 JavaScript를 실행**합니다. 화면 클릭이 아니라 API 호출입니다.

### 왜 OpenCode인가

`eext-run-api-gateway` 공식 README에서 지원하는 AI 도구 목록 첫 번째가 **OpenCode**입니다.
FAQ 전체가 OpenCode 기준 튜토리얼로 작성되어 있고, 설치 커맨드도 `~/.config/opencode/skills/` 경로를 기본으로 안내합니다. 즉 **가장 공식적인 경로**입니다.

---

## 1. 과정 개요와 시간표

| 단계 | 내용 | 예상 시간 | 자동/수동 |
|---|---|---|---|
| 1 | EasyEDA Pro 설치 | 10분 | 수동 |
| 2 | OpenCode + easyeda-api 스킬 설치 | 10분 | 자동 |
| 3 | Run API Gateway 확장 설치 | 5분 | 수동 |
| 4 | 브리지 기동 + 연결 확인 | 5분 | 반자동 |
| 5 | 실전 샘플: NE555 회로 | 20분 | AI |
| 6 | 실전 샘플: ESP32-C3 보드 | 60분 | AI |
| 7 | PCB 레이아웃 + 실크스크린 | 40분 | AI + 수동 |
| 8 | DRC → JLCPCB Gerber | 30분 | AI |
| | **합계** | **약 3시간** | |

> 실제.schema 생성에 걸리는 시간은 회로 복잡도에 따라 6~10분 수준입니다. (영상 벤치마크)

---

## 2. 사전 준비

### 하드웨어 / OS

- Windows 10 / 11 (macOS·Linux에서도 작동하나 Windows가 검증됨)
- 인터넷 연결 (설치·문서 조회용)
- EasyEDA 계정 (무료, 클라우드 프로젝트 보관용)

### 필수 소프트웨어 버전

| 항목 | 최소 | 권장 | 확인 |
|---|---|---|---|
| Node.js | 18.x | **22 LTS 이상** | `node -v` |
| OpenCode | 최신 | 1.18+ | `opencode --version` |

설치 여부 확인:

```powershell
node -v
npm -v
opencode --version
```

> **중요**: Node.js 22 이상이 OpenCode 툴체인 호환성에 가장 안전합니다. 20.17.0 미만이면 일부 SDK가 거부합니다.

---

## 3. 단계 1 — EasyEDA Pro 설치

### 3.1 다운로드

EasyEDA는 **브라우저 버전**과 **데스크톱 클라이언트(Pro)** 두 가지입니다.
API 확장은 데스크톱 클라이언트에서만 동작합니다.

- 다운로드: https://pro.easyeda.com/editor
- 데스크톱 앱을 설치하고 로그인

### 3.2 로그인

EasyEDA 계정으로 로그인합니다. 프로젝트가 클라우드에 저장되므로 로그인이 필수입니다.

### 3.3 확인

로그인 후 에디터가 열리고 좌측에 `프로젝트` / `Schematic` / `PCB` 메뉴가 보이면 정상입니다.

---

## 4. 단계 2 — OpenCode + easyeda-api 스킬 설치

### 4.1 왜 스킬이 필요한가

스킬은 다음 두 역할을 합니다.

1. **API 문서 제공** — 클래스 130개, enum 62개, 인터페이스 70개 전체 레퍼런스
   (웹에서 문서를 매번 크롤링하면 토큰 폭발, 로컬에서 읽으면 거의 무료)
2. **브리지 서버 기동 + 호출 규칙** — 브리지를 띄우고, API를 어떻게 호출할지 정의

스킬은 공식 저장소입니다: https://github.com/easyeda/easyeda-api-skill

### 4.2 설치 (권장 — ClawHub)

PowerShell에서 실행:

```powershell
npx clawhub@latest install easyeda-api --workdir "$HOME/.config\opencode" --dir skills
```

cmd 사용 시:

```bat
npx clawhub@latest install easyeda-api --workdir "%USERPROFILE%\.config\opencode" --dir skills
```

### 4.3 설치 (수동 — 네트워크 제한 시)

ClawHub가 안 되면 공식 zip을 직접 받습니다.

- 다운로드: https://image.lceda.cn/files/easyeda-api.zip
- 압축 해제 위치: `%USERPROFILE%\.config\opencode\skills\easyeda-api\`

**중요 — 중첩 디렉터리 금지:**

```
✅ 올바름:  ~/.config/opencode/skills/easyeda-api/SKILL.md
❌ 틀림:    ~/.config/opencode/skills/easyeda-api/easyeda-api/SKILL.md
```

### 4.4 설치 확인

정상 설치 시 아래 구조가 됩니다.

```
~/.config/opencode/skills/easyeda-api/
├── SKILL.md              ← 핵심 지시 파일
├── package.json
├── scripts/
│   └── bridge-server.mjs ← 브리지 서버
├── references/
│   ├── _index.md         ← 클래스/_enum_ 전체 인덱스
│   ├── _quick-reference.md ← 전체 시그니처 빠른 조회
│   ├── classes/          ← 130개 클래스 문서
│   ├── enums/            ← 62개 enum
│   ├── interfaces/       ← 70개 인터페이스
│   └── types/
├── format/               ← 프로젝트/스키매틱/PCB 소스 포맷 명세
├── guide/                ← 개발 가이드
└── user-guide/           ← 사용자 가이드
```

PowerShell로 검증:

```powershell
Test-Path "$env:USERPROFILE\.config\opencode\skills\easyeda-api\SKILL.md"
(Get-ChildItem "$env:USERPROFILE\.config\opencode\skills\easyeda-api\references\classes").Count
```

두 번째 명령의 결과가 **130** 근처면 정상입니다.

### 4.5 의존성 설치

브리지는 `ws` 패키지를 씁니다.

```powershell
cd "$env:USERPROFILE\.config\opencode\skills\easyeda-api"
npm install
```

### 4.6 OpenCode 재시작

스킬을 인식시키려면 OpenCode를 재시작하거나 환경을 리스캔해야 합니다.
확인은 `/skills` 명령으로 합니다.

---

## 5. 단계 3 — Run API Gateway 확장 설치

### 5.1 확장 다운로드

- 확장 마켓: https://jlc-ext.com/item/oshwhub/run-api-gateway
- 또는: https://jlcext.com/item/oshwhub-official/run-api-gateway

GitHub: https://github.com/easyeda/eext-run-api-gateway (Apache-2.0)

### 5.2 설치

EasyEDA Pro에서:

```
메뉴 → 설정(Settings) → 확장(Extensions) → 확장 관리자(Extension Manager)
```

`.eext` 파일을 드래그앤드롭하거나 파일 선택으로 설치합니다.

### 5.3 권한 활성화 (필수, 두 개 다)

확장 관리자에서 `Run API Gateway`를 찾아 아래 두 옵션을 켭니다.

- [x] **외부 인터페이스 허용** (Allow External Interaction)
- [x] **상단 메뉴에 표시** (Show in Top Menu)

> **외부 인터페이스 허용을 안 켜면** 브리지는 연결되지만 API 호출이 전부 거부됩니다.
> 이 옵션이 없으면 "연결은 됐는데 결과가 null" 같은 막힌 증상이 나타납니다.

### 5.4 확인

EasyEDA Pro 상단 메뉴에 **`API Gateway`** 가 보이면 성공입니다. 클릭하면 4개 항목이 나옵니다.

- `Reconnect` — 수동 재연결
- `Stop Connection` — 연결 끊기
- `Toggle Auto-Connect Status` — 자동 연결 토글
- `About...`

---

## 6. 단계 4 — 브리지 기동과 연결 확인

### 6.1 브리지 서버 기동

브리지 서버는 **백그라운드**로 돌려야 합니다. 포그라운드로 돌리면 AI가 응답을 기다리며 멈춥니다.

```powershell
Start-Process -FilePath "node" `
  -ArgumentList "scripts/bridge-server.mjs" `
  -WorkingDirectory "$env:USERPROFILE\.config\opencode\skills\easyeda-api" `
  -WindowStyle Hidden
```

### 6.2 상태 확인

```powershell
foreach ($p in 49620..49629) {
  try {
    $r = Invoke-RestMethod "http://127.0.0.1:$p/health" -TimeoutSec 2
    if ($r.service -eq 'easyeda-bridge') {
      "PORT=$p"; $r | ConvertTo-Json -Depth 4
    }
  } catch {}
}
```

정상 출력:

```json
{
  "service": "easyeda-bridge",
  "status": "ok",
  "edaConnected": false,
  "edaWindowCount": 0,
  "activeWindowId": null
}
```

- `service: "easyeda-bridge"` → 브리지 정상
- `edaConnected: false` → EasyEDA 확장 아직 미연결. **EasyEDA Pro를 실행하고 기다리면 자동으로 붙습니다.**

### 6.3 연결된 윈도우 확인

```powershell
Invoke-RestMethod "http://127.0.0.1:49620/eda-windows" | ConvertTo-Json -Depth 4
```

```json
{
  "windows": [{ "windowId": "abc-123", "connected": true, "active": true }],
  "activeWindowId": "abc-123",
  "count": 1
}
```

- `0개` → EasyEDA Pro에서 확장 로드 확인 → `API Gateway → Reconnect`
- `1개` → 자동 선택됨. 바로 사용 가능
- `2개 이상` → AI에게 어느 윈도우를 쓸지 물어보거나 선택

### 6.4 브리지 종료

작업이 끝나면 리소스를 풀어줍니다.

```powershell
Get-NetTCPConnection -LocalPort 49620 -State Listen |
  ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

### 6.5 OpenCode에서 연결 확인

브리지 띄워놓고 EasyEDA Pro 실행한 다음, OpenCode에서:

```
easyeda-api 스킬을 사용해서 현재 열려 있는 EasyEDA Pro 윈도우에 연결하고 상태를 알려줘.
```

또는 짧게:

```
EasyEDA, start!
```

---

## 7. 단계 5 — 실전 샘플: NE555 점멸 회로

복잡한 보드 전에 **최소 시스템**으로 전체 파이프라인을 검증합니다.
이 회로는 외장 부품 4~5개뿐이라 실패 지점을 isolate하기 좋습니다.

### 7.1 회로 사양

| Ref | 값 | 역할 |
|---|---|---|
| U1 | NE555P | 타이머 IC |
| R1 | 10kΩ | VCC → DISCHARGE (6번) |
| R2 | 47kΩ | DISCHARGE(6) ↔ THRESHOLD(7) |
| C1 | 10µF | THRESHOLD(7) ↔ GND — 점멸 주기 결정 |
| R3 | 330Ω | OUT(3) ↔ LED |
| LED1 | 빨강 | 표시등 |
| C2 | 100nF | 디커플링 |
| P1 | 9V 배터리 | 전원 |
| SW1 | 전원 스위치 | 전원 인가 |

- 점멸 주기 ≈ 1.1 × (R1 + 2×R2) × C1 ≈ 1.1 × (10k + 94k) × 10µF ≈ **1.15초**

### 7.2 실행 프롬프트

```
easyeda-api 스킬을 사용해서 EasyEDA Pro에서 NE555 점멸 회로 스키매틱을 만들어줘.

사양:
- U1: NE555P (DIP-8), lib_Device.search로 검색해서 정확한 부품을 골라줘
- R1 10kΩ: VCC → U1 6번(DISCHARGE)
- R2 47kΩ: U1 6번 → U1 7번(THRESHOLD)
- C1 10µF: U1 7번 → GND
- R3 330Ω: U1 3번(OUT) → LED1
- LED1: U1 3번 → GND (R3 경유)
- C2 100nF: VCC → GND
- P1: 9V 배터리를 VCC / GND로 연결

작업 순서:
1. 새 프로젝트 생성하고 Schematic 생성
2. 프로젝트 반드시 openProject로 연 뒤 진행해
3. 부품 배치 (격자에 정렬, 회로가 읽히게)
4. 전원 플래그(VCC/GND) 추가
5. 배선 완료
6. 마지막에 검토해서 오류 보고해줘

주의사항:
- 스키매틱 좌표는 1 unit = 0.01inch = 10mil 단위야
- 네트워크 전원 기호(GND/VCC)는 netFlag를 사용해서 넣어줘
- 라벨은 간결하게 (J1, C1 같은)
```

### 7.3 AI가 하는 일 (내부적으로)

OpenCode는 아래를 순서대로 실행합니다.

```javascript
// 1) 프로젝트 생성
await eda.dmt_Project.createProject("NE555 Blinker", "NE555 Blinker");

// 2) 반드시 열어야 함 — 안 열면 이후 모든 API가 null
await eda.dmt_Project.openProject(projectUuid);

// 3) Schematic 생성
await eda.dmt_Schematic.createSchematic();

// 4) 부품 검색
const devs = await eda.lib_Device.search("NE555");

// 5) 배치 (좌표는 0.01inch 단위)
await eda.sch_PrimitiveComponent.create(
  { libraryUuid: dev.libraryUuid, uuid: dev.uuid },
  500, 300, "", 0, false, true, true
);
//                     ↑x  ↑y      ↑rot ↑mir ↑bom ↑pcb

// 6) 배선 — [x1,y1,x2,y2, x2,y2,x3,y3, ...]
await eda.sch_PrimitiveWire.create([500,300,600,300], "OUT");

// 7) 전원 플래그
await eda.sch_PrimitiveComponent.createNetFlag("Ground", "GND", 600, 400);
```

### 7.4 검증 체크리스트

- [ ] R1·R2 저항값이 표시되는가
- [ ] 전원 기호(GND)가 실제로 붙어 있는가 — 안 붙으면 "floating net"
- [ ] U1 핀 번호가 회로와 맞나 (6/7/3번)
- [ ] ai가 리포트한 검증 결과에 "unconnected pin" 경고가 없는가
- [ ] 배치가 실제 배선 가능한 간격인가 (회로도만 넣는 경우)

### 7.5 실패 모드와 대응

| 증상 | 원인 | 해결 |
|---|---|---|
| 모든 create가 null 반환 | 프로젝트 미오픈 | `openProject()` 먼저 호출 |
| 부품이 멀리 흩어짐 | 좌표 단위 오해 | 스키매틱은 10mil 단위 |
| 핀이 연결 안 됨 | 배선 좌표가 핀 위치와 불일치 | 핀 좌표 조회 후 정확히 맞추기 |
| API가 permission denied | 외부 인터페이스 권한 | 확장 관리자에서 재활성화 |

---

## 8. 단계 6 — 실전 샘플: ESP32-C3 보드

### 8.1 왜 C3인가

ESP32-C3는 리퍼런스 설계가 공개되어 있고(Espressif HW docs), 부품이 복잡해서 **AI가 실제로 검증된 회로를 재현할 수 있는지** 확인하기 좋은 대상입니다. 영상에서 사용한 보드입니다.

### 8.2 필수 회로 블록

```
USB-C (5V) ──┬── Schottky diode ──┬── 3V3 (AMS1117-3.3)
              │                    │
             R1 10k               C1 10µF
              │                    │
              └── EN (pull-up)   C2 100nF ( decoupling)

USB-C D+/D- ── 22Ω ── ESP32-C3 GPIO18/19  (USB Serial/JTAG)
BOOT(GPIO9) ── 10k pull-up ── GND 버튼
RST (GPIO8)  ── 10k pull-up ── GND 버튼
안테나 영역  ── GND keep-out (공정 영역)
```

### 8.3 실행 프롬프트

```
easyeda-api 스킬을 사용해서 ESP32-C3 Development Board 회로를 스키매틱으로 구성해줘.

Espressif 공식 reference design을 기준으로:
https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3-devkitc-02.html

블록:
1. USB-C 커넥터 (5V, GND, D+, D-)
2. Schottky diode를 거쳐 5V → 3V3
3. AMS1117-3.3 LDO (5V in, 3V3 out)
4. 입력 캡 10µF + 출력 캐패시터 10µF + 디커플링 100nF
5. R1 10k: EN → 3V3 (pull-up)
6. R2/R3 10k: GPIO8(RST), GPIO9(BOOT) pull-up + GND 버튼 2개
7. ESP32-C3-MINI-1 모듈
8. USB D+/D-: 22Ω 직렬저항 → GPIO20/GPIO21
9. 부트스트랩 저항 (EN은 GPIO9)

이후:
- 전원 플래그와 네트 라벨 정리
- ESP32 각 핀에 뭐가 붙었는지 표로 정리해서 보고
- 레퍼런스와 비교 검토
```

### 8.4 실전 소요 시간

- 회로 구성: **6~10분** (영상 벤치마크)
- 검토 리포트: 2~3분
- 손으로 다듬기: 10~20분

### 8.5 검토 요청 프롬프트

스키매틱 완성 후 반드시 검토를 한 번 거칩니다.

```
완성된 스키매틱을 검토해서:
1. ESP32-C3 각 핀에 어떤 신호가 연결됐는지 표로 정리
2. GND 연결이 빠진 곳 없는지
3. 디커플링 캡이 각 전원 핀에 붙어 있는지
4. 부트스트랩 저항과 EN pull-up 값이 맞는지
5. 공식 레퍼런스와 다른 부분 목록

문제 있으면 표로 보고해줘. 절대 임의로 수정하지 말고 보고만 해.
```

> **팁**: AI에게 수정을 맡기기 전에 반드시 리포트를 먼저 받는 게 안전합니다.
> AI가 스스로 판단해서 고치면 "왜 이렇게 고쳤는지" 추적이 안 됩니다.

---

## 9. 단계 7 — PCB 레이아웃과 실크스크린

### 9.1 PCB 생성

스키매틱이 완성되면:

```
easyeda-api 스킬로 PCB를 새로 생성하고, 현재 스키매틱의 변경사항을
pcb_Document.importChanges()로 가져와줘. 그 다음 전체 배치를 보고해줘.
```

### 9.2 부품 배치

```
PCB 배치를 진행해서:
- USB-C 커넥터는 보드 왼쪽 가장자리에 (케이블 나가는 방향 고려)
- ESP32-C3 모듈은 안테나 영역을 보드 가장자리로 두고 배치
- LDO, 디커플링 캡은 전원 핀 가까이
- 결정정렬이랑 부품 간격을 8mm 이상으로
- 실리콘크린에 남는 공간도 고려

배치 끝나면 전체 배치 요약표 만들어줘.
```

배치 확인용 API:

```javascript
// 현재 PCB의 모든 부품
const comps = await eda.pcb_PrimitiveComponent.getAll();
comps.map(c => ({ id: c.getState_PrimitiveId(), x: c.getState_X(), y: c.getState_Y() }));

// 특정 부품의 핀/패드 확인
await eda.pcb_PrimitiveComponent.getAllPinsByPrimitiveId(primitiveId);
```

### 9.3 부품 이동 (수정 패턴)

PCB/SCH의 기존 primitive를 수정할 때는 **async 패턴**을 반드시 써야 합니다.

```javascript
// 1) 조회
const prim = await eda.pcb_PrimitiveComponent.get([primitiveId]);
// 2) async 모드 전환
const asyncPrim = prim.toAsync();
// 3) 값 설정
asyncPrim.setState_X(newX);
asyncPrim.setState_Y(newY);
asyncPrim.setState_Rotation(90);
// 4) 확정
asyncPrim.done();
```

`done()`을 안 부르면 **변경이 화면에 반영되지 않습니다.** 이게 실수 중 가장 흔한 케이스입니다.

### 9.4 실크스크린

```
각 부품 옆에 레퍼런스 디자인레이터(R1, C1, U1 등)를 실크스크린 레이어에
부품 바깥쪽으로 배치해줘. 실크스크린이 부품 패드와 겹치면 안 돼.
겹치는 것들 찾으면 보고해줘.

실크스크린 라인도 전원 기호(GND/VCC) 근처에 넣어줘.
```

### 9.5 자동 배선 (BETA)

```
pcb_Document.autoRouting() 으로 autorouting 해줘.
그 다음 rats线(unrouted connection) 개수를 보고해줘.
```

> **주의**: EasyEDA 공식 문서에 "현재 자동 배선 효과는 일반적이며 수동 조정이 필요하다"고 명시돼 있습니다.
> 결과물을 반드시 DRC로 검증하세요.

배선 결과 확인:

```javascript
// 미배선 상태 확인
const status = await eda.pcb_Document.getCalculatingRatlineStatus();

// 배선 삭제 후 재시도
await eda.pcb_Document.clearRouting('all');
```

외부 배선 경로(선급)도 지원합니다.

```
1. DSN 내보내기 → 2. 외부 autorouter(Specctra DSN 등) → 3. SES 파일 → 4. 재임포트
API: pcb_Document.importAutoRouteSesFile()
```

### 9.6 보드 외곽선 — 여기서부터는 사람이 필요

영상 제작자도 **이 부분은 직접 작업했다고 명시**했습니다.

`pcb_PrimitiveLine`으로 `EPCB_LayerId.BOARD_OUTLINE`(값 11) 레이어에 선을 그려 사각형을 만들 수는 있습니다.

```javascript
await eda.pcb_PrimitiveLine.create(
  "",                        // net (외곽선은 빈 문자열)
  EPCB_LayerId.BOARD_OUTLINE,
  0, 0,                      // 시작점
  2000, 0,                   // 끝점
  10,                        // lineWidth
  false
);
```

다만 실무 판단이 필요한 부분이 남습니다.

- 보드 크기 결정 (안테나 keep-out, 케이블 여유, USB-C 하우징 규격)
- 라운드 코너 R 값
- 마운팅 홀 위치
- 다른 앵커 보드(Standard edge cuts) 규격 맞춤

> **결론**: 직사각형 외곽선은 스크립트로 뽑을 수 있지만, **실제품 규격 결정은 사람이 합니다.**

### 9.7 패널라이징 — AI가 가장 어려운 영역

API는 존재합니다 (`eda.dmt_Panel`, 패널 소스 포맷 문서도 포함).
하지만 다중 보드 간 간격, 돌출(tab) 구조, V-cut, 확장점(mark) 계산은 실제 생산 규격과 맞아야 하고, **영상 제작자도 수동 작업**했다고 밝힌 영역입니다.

→ **첫 프로토타입은 패널라이징 없이 단품으로 먼저制造**하세요.
JLCPCB에 단품 SMT를 주문해도 수량 최소치는 매우 낮습니다.

---

## 10. 단계 8 — DRC → JLCPCB 출고

### 10.1 DRC 실행

```javascript
// 간단 검사 (통과 시 true)
const passed = await eda.pcb_Drc.check(true, true, false);

// 상세 검사 (오류 목록 반환)
const errors = await eda.pcb_Drc.check(true, true, true);
```

프롬프트:

```
pcb_Drc.check()로 DRC 돌리고, 발견된 오류를 심각도별로 분류해서 표로 정리해줘.
아직 고치지 말고 보고만 해줘.
```

> **알려진 이슈**: EasyEDA v2.2.47.x에서 `overwriteCurrentRuleConfiguration()`이 교착 상태가 되고
> `saveRuleConfiguration()`이 동작하지 않는 버그가 서드파티 레포에 보고돼 있습니다.
> 규칙 일괄 설정이 안 먹으면 이 버그일 가능성이 높으니, PCB 화면에서 직접 수동 설정하세요.

### 10.2 네트워크 클레스 / 차등페어

```javascript
// 전원 네트워크 클레스 생성
await eda.pcb_Drc.createNetClass("POWER", ["3V3", "5V", "VBUS"], color);

// 차등페어 생성
await eda.pcb_Drc.createDifferentialPair("USB", "D+", "D-");

// 등길이 그룹
await eda.pcb_Drc.createEqualLengthNetGroup("SDIO", ["CMD", "CLK", "D0", "D1"], color);
```

차등페어는 USB처럼 반드시 등길이로 맞춰야 합니다.

### 10.3 Gerber / BOM / CPL 추출

EasyEDA Pro 상단 메뉴:

```
파일(File) → 내보내기(Export) → Gerber / BOM / 좌표파일(CPL)
```

- **Gerber** — 제조용
- **BOM** — 부품 목록 (LCSC 품번 포함이 JLCPCB SMT에 필요)
- **CPL (Pick & Place)** — 실장 좌표

> **JLCPCB SMT 주문의 핵심**: BOM에 **LCSC Part Number (C开头 8자리)** 가 들어 있어야 합니다.
> EasyEDA에서 부품 검색 시 LCSC 라이브러리를 선택하면 자동으로 붙습니다. 라이브러리를 임의로 만들면 실장이 안 됩니다.

### 10.4 주문 전 최종 체크

- [ ] DRC 오류 0개
- [ ] 보드 외곽선 완전 폐쇄 (안 닫으면 Gerber 에러 + 코퍼 푸 안 나옴)
- [ ] 외곽선이 불필요한 라인 중복이 없는지
- [ ] 모든 부품에 BOM에 올바른 LCSC 번호가 있음
- [ ] 실크스크린이 패드/마스크오프셋과 안 겹침
- [ ] 전원극성 핀1 표시
- [ ] 좌면 부품 방향 확인
- [ ] 안테나 keep-out 영역에 부품/지면铜 없음

---

## 11. 무엇을 AI가 하고, 무엇을 사람이 하는가

영상 내용을 검증한 결과입니다.

### AI가 잘하는 것

| 작업 | 평가 |
|---|---|
| 회로 블록 생성 (스키매틱) | **우수** — 공식 레퍼런스 기반으로 빠르게 |
| 부품 배치 | **우수** — 영상에서도 "생각보다 잘 된다" |
| 배선 (autorouting) | **양호** — BETA, 수동 조정 필요 |
| 회로 검토 / 비교 리포트 | **우수** — 표로 정리해주는 게 큰 도움 |
| DRC 실행 및 분류 | **우수** — 사람이 하나씩 누르는 것보다 빠름 |
| BOM / LCSC 코드 매칭 | **우수** |
| 실크스크린 텍스트 배치 | **양호** |
| 문서 소스 포맷 분석 | **우수** — `format/` 문서가 제공됨 |

### 사람이 해야 하는 것

| 작업 | 이유 |
|---|---|
| **보드 외곽선 규격 결정** | 케이블 하우징, 마운팅 규격 같은 물리적 제약 |
| **패널라이징** | 생산 규격 + 공정 지식 필요. 영상도 수동 작업 |
| ** autorouting 결과 다듬기** | BETA 한계, 고속·아날로그는 수동 필수 |
| **전원 무결성 검토** | 리플레스 전류, 디커플링 위치, 발진 여부 |
| **고속 신호 검토** | 등길이, 임피던스, 시그널 무결성 |
| **안테나 keep-out** | 무선 성능 — 측정 필요 |
| **부품 선택 최종 판단** | 라이프사이클, 재고, 대체품 |
| **펌웨어 올리고 동작 테스트** | 회로가 "그려졌다" ≠ "작동한다" |

### 결정적인 한계

**회로가 그려지는 것과 회로가 작동하는 것은 다릅니다.**

영상 제작자도 이 부분을 정확히 인정합니다 — "PCB는 문제없이 작동했고, 미리 준비한 펌웨어를 올려 테스트했다".
즉 **AI 설계 → 수동 펌웨어 검증**이 필수 사이클입니다.

---

## 12. 실전 팁 — 프롬프트 레퍼런시

### 작업 분할 원칙

한 번에 "전체 보드 만들어줘"라고 하지 마세요. 이렇게 나누는 게 성공률입니다.

```
1단계: 프로젝트 + 문서 생성만
2단계: 블록 단위로 회로 구성 (전원 → MCU → 클럭 → 커넥터)
3단계: 검토 (수정 금지, 보고만)
4단계: 지적된 것만 수정
5단계: PCB 생성 + 배치
6단계: 배선
7단계: DRC (보고만)
8단계: 지적된 것만 수정
```

### 좋은 프롬프트 패턴

```
easyeda-api 스킬을 사용해줘.

목표: [구체적 회로명]

사양:
- [Ref Desig]: [부품명], [값]
- [Ref Desig]: [부품명], [값]
[연결 관계 명시]

참조: [공식 문서 링크]

제약:
- 좌표 단위 주의해 (스키매틱 10mil / PCB 1mil)
- 좌측 정렬, 간격 일정하게
- 전원 플래그 반드시 붙여

완료 후:
- 검토해서 표로 보고 (수정은 하지 말고)
```

### 피해야 할 프롬프트

| 지시 | 이유 |
|---|---|
| "최적으로 만들어줘" | 기준이 없어 AI가 추측 |
| "레퍼런스 무시하고 네가 설계해" | 검증 불가, 위험 |
| "최적화해서 배선해줘" | BETA autorouting에 맡기면 DRC 깨짐 |
| "고쳐줘" (리포트 없이) | 이유 추적 불가 |

### 세션 관리

브리지가 백그라운드에서 계속 리소스를 먹습니다. 주제가 완전히 바뀌면 정리하세요.

```
easyeda-api 스킬 정리하고 브리지 서버 내려줘.
```

---

## 13. 문제 해결 (Troubleshooting)

### 브리지 연결 실패

**증상**: `edaConnected: false`, `edaWindowCount: 0`

점검 순서:

1. EasyEDA Pro가 실행 중인가
2. 상단 메뉴에 `API Gateway`가 보이는가
   - 안 보이면 **확장 자체가 로드 안 된 것** → 확장 관리자에서 삭제 후 재설치
3. `API Gateway → Reconnect` 클릭
4. 브리지 프로세스 살아있는지 확인

```powershell
Get-NetTCPConnection -LocalPort 49620 -State Listen
```

**흔한 원인: 포트 충돌.** 49620~49629를 다른 프로그램이 점유합니다.
자주 범인: IDE Live Share, 디버그 프록시, 다른 로컬 서비스.

```powershell
Get-NetTCPConnection -LocalPort (49620..49629) -State Listen
netstat -ano | findstr :4962
```

점유 프로세스를 종료하고 EasyEDA에서 `Reconnect`.

**"Handshake failed: unexpected service"** → 127.0.0.1:4962x에 다른 프로그램이 있는 경우입니다.
브리지인지 확인하려면 반드시 `service === "easyeda-bridge"` 를 검증해야 합니다.

### 최종 복구 수단 (대부분 이걸로 해결됨)

이 순서로 전부 껐다가 다시 켜면 해결됩니다.

1. OpenCode 종료
2. EasyEDA Pro 종료
3. EasyEDA Pro 다시 실행 → 확장 로드 확인
4. 사용자 홈 디렉터리에서 `opencode` 재실행
5. 연결 명령 재실행

### 프록시 환경

```powershell
$env:HTTP_PROXY  = "http://proxy:port"
$env:HTTPS_PROXY = "http://proxy:port"
```

일부 사내 네트워크는 `ws://127.0.0.1` 핸드셰이크를 차단합니다.

### API가 계속 null 반환

체크리스트:

- [ ] 프로젝트가 **열려** 있는가 (`openProject` 호출했는가)
- [ ] **올바른 문서 타입**이 활성 상태인가
      - `PCB_*` API → PCB 문서 활성 상태여야 함
      - `SCH_*` API → Schematic 페이지 활성 상태여야 함
- [ ] 외부 인터페이스 권한이 켜져 있는가
- [ ] 계정/라이선스 등급에 제한이 있는가

확인 코드:

```javascript
const proj = await eda.dmt_Project.getCurrentProjectInfo();
const doc  = await eda.dmt_SelectControl.getCurrentDocumentInfo();
return { proj, doc };
```

### 시간이 오래 걸릴 때

브리지는 기본 타임아웃 30초입니다. 복잡한 작업은 쪼개세요.

- 스키매틱을 페이지 단위로 분리
- 부품 20개씩 배치
- 배선은 네트 그룹별로

---

## 부록 A. 치명적인 실수 5가지

### A-1. await 누락 (가장 흔함)

```javascript
// ❌ Promise 객체가 반환됨 — 실제 데이터가 아님
const project = eda.dmt_Project.getCurrentProjectInfo();

// ✅ 반드시 await
const project = await eda.dmt_Project.getCurrentProjectInfo();
```

EasyEDA API는 **거의 전부** Promise를 반환합니다. 반환 타입에 `Promise<...>`가 있으면 await 필수.
확인 안 하면 조용히 잘못된 값이 오고 원인을 찾기 매우 어렵습니다.

### A-2. 좌표 단위 오해 (비용이 가장 큼)

| 도메인 | 1 unit = | 환산 |
|---|---|---|
| **PCB** | 1 mil (0.0254mm) | 1mm ≈ 39.37 units |
| **Schematic** | 0.01 inch = 10 mil (0.254mm) | 1mm ≈ 3.937 units |

**10배 차이입니다.** 잘못 쓰면 부품이 의도한 위치에서 10배 멀리 배치됩니다.

### A-3. 숫자 대신 enum 사용

```javascript
// ❌ 의미 불명, 잘못된 레이어로 조용히 배치될 수 있음
await eda.pcb_PrimitiveLine.create("GND", 1, 0, 0, 1000, 0, 10, false);

// ✅ 명확하고 안전
await eda.pcb_PrimitiveLine.create("GND", EPCB_LayerId.TOP, 0, 0, 1000, 0, 10, false);
```

레퍼런스: `references/enums/EPCB_LayerId.md`

레이어 enum 값 참고:

| 레이어 | 값 |
|---|---|
| `TOP` | 1 |
| `BOTTOM` | 2 |
| `TOP_SILKSCREEN` | 3 |
| `BOTTOM_SILKSCREEN` | 4 |
| `TOP_SOLDER_MASK` | 5 |
| `BOTTOM_SOLDER_MASK` | 6 |
| `TOP_PASTE_MASK` | 7 |
| `BOTTOM_PASTE_MASK` | 8 |
| `TOP_ASSEMBLY` | 9 |
| `BOTTOM_ASSEMBLY` | 10 |
| `BOARD_OUTLINE` | 11 |
| `MULTI` | 12 |
| `DOCUMENT` | 13 |
| `MECHANICAL` | 14 |
| `INNER_1` | 15 |
| `HOLE` | 47 |
| `RATLINE` | 57 |

### A-4. 프로젝트 미오픈

```javascript
await eda.dmt_Project.createProject("MyBoard");  // 생성만
await eda.dmt_Project.openProject(uuid);          // ❗이게 없으면 이후 전부 실패
```

스킬 문서가 이렇게 경고합니다:
> "프로젝트를 생성했다면 반드시 열기 전에 `openProject()`를 호출해야 합니다. 프로젝트를 열기 전에는 어떤 문서도 조작할 수 없습니다."

### A-5. `done()` 누락

```javascript
const prim = await eda.pcb_PrimitiveComponent.get([id]);
const a = prim.toAsync();
a.setState_X(1000);
a.setState_Y(2000);
a.done();   // ❗ 이게 없으면 화면에 반영 안 됨
```

---

## 부록 B. 명령어 cheatsheet

### Windows PowerShell — 환경

```powershell
node -v
npm -v
opencode --version

# 스킬 설치
npx clawhub@latest install easyeda-api --workdir "$HOME/.config\opencode" --dir skills

# 스킬 의존성
cd "$env:USERPROFILE\.config\opencode\skills\easyeda-api"; npm install
```

### Windows PowerShell — 브리지

```powershell
# 기동 (백그라운드)
Start-Process -FilePath "node" `
  -ArgumentList "scripts/bridge-server.mjs" `
  -WorkingDirectory "$env:USERPROFILE\.config\opencode\skills\easyeda-api" `
  -WindowStyle Hidden

# 상태
Invoke-RestMethod "http://127.0.0.1:49620/health" | ConvertTo-Json -Depth 4

# 윈도우 목록
Invoke-RestMethod "http://127.0.0.1:49620/eda-windows" | ConvertTo-Json -Depth 4

# 윈도우 선택
Invoke-RestMethod -Method POST "http://127.0.0.1:49620/eda-windows/select" `
  -ContentType "application/json" -Body '{"windowId":"abc-123"}'

# 코드 실행
Invoke-RestMethod -Method POST "http://127.0.0.1:49620/execute" `
  -ContentType "application/json" `
  -Body '{"code":"return await eda.dmt_Project.getCurrentProjectInfo();"}'

# 종료
Get-NetTCPConnection -LocalPort 49620 -State Listen |
  ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

### 자주 쓰는 API 빠른 참조

| 하고 싶은 것 | API |
|---|---|
| 현재 프로젝트 | `eda.dmt_Project.getCurrentProjectInfo()` |
| 현재 문서 | `eda.dmt_SelectControl.getCurrentDocumentInfo()` |
| 프로젝트 생성 | `eda.dmt_Project.createProject(name)` |
| 프로젝트 열기 | `eda.dmt_Project.openProject(uuid)` |
| Schematic 생성 | `eda.dmt_Schematic.createSchematic()` |
| Schematic 페이지 생성 | `eda.dmt_Schematic.createSchematicPage(schUuid)` |
| PCB 생성 | `eda.dmt_Pcb.createPcb()` |
| 문서 열기 | `eda.dmt_EditorControl.openDocument(uuid)` |
| 부품 검색 | `eda.lib_Device.search("ESP32")` |
| LCSC 번호로 검색 | `eda.lib_Device.getByLcscIds("C194584")` |
| 스키매틱 부품 배치 | `eda.sch_PrimitiveComponent.create(...)` |
| 스키매틱 배선 | `eda.sch_PrimitiveWire.create([x1,y1,x2,y2,...], net)` |
| GND 플래그 | `eda.sch_PrimitiveComponent.createNetFlag('Ground','GND',x,y)` |
| PCB 부품 | `eda.pcb_PrimitiveComponent.getAll()` |
| PCB 라인 | `eda.pcb_PrimitiveLine.create(...)` |
| PCB에 스키메틱 반영 | `eda.pcb_Document.importChanges(uuid)` |
| 자동 배치 | `eda.pcb_Document.autoLayout()` |
| 자동 배선 (BETA) | `eda.pcb_Document.autoRouting()` |
| 배선 삭제 | `eda.pcb_Document.clearRouting('all')` |
| DRC | `eda.pcb_Drc.check(true,true,true)` |
| 네트워크 클레스 | `eda.pcb_Drc.createNetClass(...)` |
| 차등페어 | `eda.pcb_Drc.createDifferentialPair(...)` |
| 문서 저장 | `eda.pcb_Document.save()` |
| 토스트 메시지 | `eda.sys_Message.showToastMessage("완료")` |

### OpenCode 프롬프트 단축키

```
EasyEDA, start!
easyeda-api 스킬로 연결 상태 확인해줘
easyeda-api 스킬 정리하고 브리지 내려줘
```

---

## 참고 링크

| 용도 | 링크 |
|---|---|
| EasyEDA Pro 에디터 | https://pro.easyeda.com/editor |
| 공식 API 문서 | https://prodocs.easyeda.com/en/api/guide |
| PCB_Document 레퍼런스 | https://prodocs.easyeda.com/en/api/reference/pro-api.pcb_document.html |
| Run API Gateway 확장 | https://jlc-ext.com/item/oshwhub/run-api-gateway |
| 확장 GitHub | https://github.com/easyeda/eext-run-api-gateway |
| easyeda-api 스킬 GitHub | https://github.com/easyeda/easyeda-api-skill |
| 공식 FAQ (자세한 튜토리얼) | https://github.com/easyeda/eext-run-api-gateway/blob/main/FAQ.en.md |
| 확장 개발 SDK | https://github.com/easyeda/pro-api-sdk |
| JLCPCB | https://www.jlcpcb.com/ |
| LCSC | https://www.lcsc.com/ |
| OSHWLAB (공개 프로젝트) | https://oshwlab.com/ |

---

## 최종 요약

**무엇이 진짜 가능한가**

- 스키매틱 회로 구성: 가능하고 빠름 (6~10분)
- 회로 검토 / 리포트: 가능하고 유용
- PCB 부품 배치: 가능하고 품질 좋음
- 자동 배선: 가능하나 BETA, 수동 조정 필요
- DRC / BOM / Gerber: 가능
- JLCPCB SMT + 대량 생산: 가능

**무엇이 안 되는가**

- 보드 외곽선 규격 결정 (스크립트 가능, 판단은 사람이)
- 패널라이징 (실무 지식 필요)
- 고속 / 아날로그 / RF 최종 검증
- "만들어진 회로가 실제로 작동한다"는 보장은 없음

**결론**: AI가 설계·정리 작업을 크게 가속하고, 사람은 물리적 제약 판단과 실제 동작 검증에 집중하는
**협업 구조가 현재 현실적인 최적 방식**입니다. 첫 보드는 단순한 회로(NE555, LED, MCU 최소 시스템)로
전체 파이프라인을 먼저 검증하는 걸 권합니다.