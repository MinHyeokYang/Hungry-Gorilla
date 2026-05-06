# SupportDesk — K-Beauty 지원사업 매칭 플랫폼

> K-뷰티 브랜드 대표가 기업 조건을 입력하면, KOTRA·중기부·지자체 지원사업을 자동 매칭하고
> 받을 수 있는 이유·못 받는 이유·신청 가이드를 한 번에 제공하는 웹 서비스

---

## Project Context

### Problem Statement
K-뷰티 중소 브랜드 대표는 어떤 정부 지원사업이 있는지, 자기 조건에 맞는지 알기 어렵다.
각 기관 사이트를 직접 돌아다니며 확인해야 하고, 조건 충족 여부도 스스로 판단해야 한다.
결국 지원 마감일을 놓치거나, 조건이 안 맞는 사업에 시간을 낭비하는 일이 반복된다.

### Core User Goal
> "내 브랜드 조건으로 지금 받을 수 있는 지원이 뭔지, 왜 안 되는지, 어떻게 신청하는지 한 번에 알고 싶다."

### Target User
- K-뷰티 브랜드 대표 / 창업자
- 수출 전담 인력 없이 해외 진출을 준비 중
- 지원사업 정보를 구글링이나 지인 소개로 얻고 있음
- 미국·영국 시장 진출을 목표로 하는 인디 브랜드

---

## Current State (v0.1 — Prototype)

### What's Built
- `kbeauty-support.html` — 단일 파일 프로토타입 (HTML + CSS + Vanilla JS)
- 조건 입력 플로우 (기업 정보 / 제품 정보 / 해외 진출 / 보유 인증)
- 하드필터 + 소프트 스코어링 2단계 매칭 엔진
- 4단계 결과 상태 (최적매칭 / 신청가능 / 조건미충족 / 해당없음)
- 클릭 시 펼쳐지는 상세 카드 (조건 체크 / AI 이유 / 신청 절차)
- 필터 탭 (전체 / 상태별)
- Mock DB 6개 지원사업 하드코딩

### What's NOT Built Yet
- 실제 지원사업 DB (크롤링 자동 수집)
- Claude API 연동 (AI 매칭 이유 실제 생성)
- 마감일 D-day 실시간 계산 (현재 정적 날짜)
- 회원가입 / 북마크 / 알림
- 관리자 페이지 (지원사업 DB 편집)

---

## Tech Stack

```
Frontend:   Vanilla HTML + CSS + JavaScript (no framework)
AI Engine:  Anthropic Claude API (예정 — claude-opus-4-5)
            - 매칭 이유 생성, 대안 추천 문구 생성
Fonts:      Google Fonts (Nanum Myeongjo, Pretendard, DM Mono)
Storage:    없음 (MVP는 상태 유지 없음)
Hosting:    Static file — Vercel / Netlify / GitHub Pages
```

---

## File Structure

```
/
├── kbeauty-support.html    # MAIN FILE. 전체 앱이 이 파일 안에 있음
├── README.md               # This file
└── (planned)
    ├── /api
    │   ├── match.js        # Claude API 프록시 — 매칭 이유 생성
    │   └── crawl.js        # 지원사업 크롤러 (KOTRA, 중기부, 지자체)
    ├── /db
    │   └── supports.json   # 지원사업 DB (크롤링 결과 저장)
    └── /admin
        └── index.html      # 지원사업 DB 관리 페이지
```

---

## Architecture

### Data Flow

```
User Input (조건 입력)
  │
  ├─ revenue          (연 매출 규모)
  ├─ employees        (직원 수)
  ├─ business_years   (업력)
  ├─ region           (소재지)
  ├─ sku_count        (SKU 수)
  ├─ moq              (MOQ)
  ├─ category         (제품 카테고리)
  ├─ target_countries (타겟 국가 — 복수 선택)
  ├─ channels         (판매 채널 — 복수 선택)
  ├─ has_export_experience (수출 경험 여부)
  └─ certifications   (보유 인증 — 복수 선택)
  │
  ▼
collectInput() → userInput 객체
  │
  ▼
matchAll(userInput)
  ├─ hardFilter()     → pass | fail (이진 판단)
  └─ scoreMatch()     → 0~100점 (가중치 스코어링)
  │
  ▼
status 판정
  ├─ best_match  (score ≥ 80)
  ├─ eligible    (hardFilter pass, score 60~79)
  ├─ partial     (hardFilter pass, score < 60)
  └─ ineligible  (hardFilter fail)
  │
  ▼
renderResults() → 카드형 결과 UI
  │
  └─ (예정) Claude API → AI 매칭 이유 실시간 생성
```

---

## Support DB Schema

지원사업 1건당 구조. `SUPPORTS` 배열에 하드코딩되어 있음. 추후 `/db/supports.json`으로 분리 예정.

```javascript
{
  // 기본 정보
  id: "kotra-export-voucher",          // 고유 ID (kebab-case)
  name: "수출바우처 사업 (일반형)",      // 지원사업명
  agency: "KOTRA",                     // 기관명
  agency_type: "national",             // "national" | "sme" | "local"
  region: null,                        // 지자체면 "서울" | "경기" 등, 국가기관은 null
  support_type: "voucher",             // "voucher" | "subsidy" | "loan" | "rd" | "consulting" | "exhibition"
  description: "string",              // 1~2줄 설명
  max_amount: 50000000,               // 최대 지원금액 (원)
  support_ratio: 70,                  // 지원 비율 (%)
  url: "https://...",                 // 공식 신청 링크
  deadline: "2025-11-30",            // 신청 마감일 (YYYY-MM-DD)
  is_recurring: true,                 // 매년 반복 여부
  recruit_start: "2025-09-01",       // 모집 시작일
  period_note: "연 1회, 매년 9~11월", // 자유 텍스트

  // 자격 조건 (매칭 엔진 핵심)
  conditions: {
    max_revenue: 5000000000,          // 연매출 상한 (null = 제한없음)
    min_revenue: null,
    max_employees: 50,
    min_employees: null,
    min_business_years: 1,
    target_countries: ["all"],        // ["all"] or ["us","uk","jp","cn","sea"]
    channels: ["all"],
    requires_export_experience: false,
    required_certs: [],               // 필수 인증 — 없으면 hardFilter에서 탈락
    required_regions: null,           // null = 전국 | ["서울","경기"] = 해당 지역만
  },

  // 신청 가이드
  apply_steps: [
    { step: 1, title: "string", desc: "string" },
  ],
  tags: ["마케팅", "인증비용"],

  // AI 매칭 이유 (현재 하드코딩 — 추후 Claude API로 동적 생성)
  reasons: {
    best_match: "string",
    eligible: "string",
    partial: "string",
    ineligible: "string",
  }
}
```

---

## Matching Engine

### Step 1 — Hard Filter (이진 판단, 코드 기반)

`hardFilter(support, user)` 함수. 아래 조건 중 하나라도 실패하면 `ineligible`.

```
max_revenue       → user.revenue > max_revenue 이면 탈락
min_revenue       → user.revenue < min_revenue 이면 탈락
max_employees     → user.employees > max_employees 이면 탈락
min_business_years → user.business_years < min_business_years 이면 탈락
required_regions  → user.region이 목록에 없으면 탈락
requires_export_experience → 수출경험 필수인데 없으면 탈락
required_certs    → 필수 인증이 user.certifications에 없으면 탈락 + missing 반환
```

### Step 2 — Soft Scoring (가중치 점수, 0~100점)

`scoreMatch(support, user)` 함수.

| 변수 | 가중치 | 계산 방식 |
|------|--------|-----------|
| 매출 적합도 | 30% | max_revenue 대비 현재 매출 비율로 역산 |
| 국가 매칭 | 25% | target_countries와 user.target_countries 교집합 비율 |
| 업력 충족 | 15% | min_business_years 이상이면 15점, 미달이면 5점 |
| 인증 보유 | 15% | GMP·ISO·FDA 중 하나라도 보유 시 15점 |
| 수출 경험 | 15% | 있으면 15점, 없으면 8점 |

### Step 3 — Status 판정

```
hardFilter == false           → ineligible
score >= 80                   → best_match
score >= 60                   → eligible
score < 60                    → partial
```

---

## Key Functions

| 함수 | 설명 |
|------|------|
| `collectInput()` | 폼에서 userInput 객체 수집. select 값을 실제 숫자로 변환 |
| `hardFilter(support, user)` | 이진 탈락 판단. true 또는 { pass:false, missing:[] } 반환 |
| `scoreMatch(support, user)` | 0~100점 스코어 반환 |
| `getStatus(support, user)` | hardFilter + scoreMatch 조합 → status + score 반환 |
| `matchAll(user)` | SUPPORTS 전체에 getStatus 적용, status 순·score 역순 정렬 |
| `renderTopbar(results, user)` | 상단 요약 (신청가능/조건미충족/해당없음 카운트) |
| `renderFilterTabs(results)` | 필터 탭 렌더링. `currentFilter` 상태와 연동 |
| `renderList(results, user)` | 지원사업 카드 목록 렌더링. `currentFilter`로 필터링 |
| `toggleCard(id)` | 카드 펼치기/닫기. 한 번에 하나만 펼쳐짐 |
| `setFilter(key)` | currentFilter 변경 → filterTabs + List 재렌더링 |
| `startMatching()` | 조건 수집 → 로딩 → showResults() |
| `showResults()` | matchAll() 실행 → results 섹션 렌더링 |
| `resetApp()` | 결과 화면 → 입력 폼으로 리셋 |

---

## State Variables

```javascript
let userInput = {};           // collectInput() 결과. matchAll()에 전달
let matchResults = [];        // matchAll() 결과 배열. 필터링 기준
let currentFilter = 'all';   // 'all' | 'best_match' | 'eligible' | 'partial' | 'ineligible'
let hasExportExp = false;     // 수출 경험 토글 상태
```

---

## Design System

### Color Tokens (CSS Variables)

```css
--bg: #0E0E0F           /* 페이지 배경 — 다크 */
--surface: #161618      /* 카드 배경 */
--surface2: #1E1E21     /* 입력 필드 배경 */
--gold: #C9A84C         /* 주요 강조색 — 최적매칭, 버튼 */
--gold-light: #E8C97A
--green: #4CAF7D        /* 신청가능 */
--yellow: #E8A020       /* 조건미충족, 마감임박 */
--red: #E05C5C          /* 실패 조건, 긴급 마감 */
--blue: #5B8DEF         /* 국가기관 뱃지 */
```

### Status → Visual Mapping

| Status | 카드 왼쪽 바 | 점 색상 | 뱃지 배경 |
|--------|------------|---------|---------|
| best_match | gold + glow | gold | gold-bg |
| eligible | green | green | green-bg |
| partial | yellow | yellow | yellow-bg |
| ineligible | text-faint, opacity 0.65 | text-faint | surface3 |

### Typography

- 제목·숫자: `Nanum Myeongjo` (serif, 한국 감성)
- 본문·UI: `Pretendard` (sans-serif, 가독성)
- 코드·뱃지·모노: `DM Mono`

---

## Mock Data (현재 6개)

| ID | 기관 | 유형 | 최대 지원금 | 특이 조건 |
|----|------|------|------------|---------|
| kotra-export-voucher | KOTRA | 바우처 | 5,000만원 | 매출 50억 이하 |
| kotra-kbeauty-export | KOTRA | 전시회 | 1,000만원 | 미국·영국·EU 타겟 |
| sme-startup-export | 중소벤처기업부 | 컨설팅 | 2,000만원 | 매출 10억 이하, 수출경험 무관 |
| sme-rd-support | 중소벤처기업부 | R&D | 1억원 | GMP 인증 필수 |
| seoul-startup-global | 서울시 | 보조금 | 3,000만원 | 서울 소재 + 매출 5억 이하 |
| kita-cert-support | KITA | 바우처 | 1,500만원 | 미국·영국 타겟 전용 |

---

## Next Steps for Codex

### P0 — 실사용 가능한 상태 만들기

**1. 지원사업 DB 확장**
- `SUPPORTS` 배열에 지원사업 10→30개로 확대
- KOTRA, 중기부, 지자체(서울·경기·부산) 실제 지원사업 추가
- 각 지원사업마다 `conditions`, `apply_steps`, `reasons` 필드 완성

**2. `/db/supports.json` 분리**
- `SUPPORTS` 배열을 별도 JSON 파일로 분리
- `fetch('/db/supports.json')`으로 로드하도록 변경
- 이후 크롤러가 이 파일을 업데이트하는 구조

**3. 마감일 D-day 실시간 계산**
- 현재 `dDay()` 함수는 이미 구현됨
- 지원사업 `deadline` 필드를 실제 2025년 마감일로 업데이트

### P1 — Claude API 연동

**4. 매칭 이유 동적 생성**
- 현재 `reasons` 필드는 하드코딩된 문자열
- Claude API 연결 후 userInput + support 조건을 프롬프트에 넣어 동적 생성
- 시스템 프롬프트: "너는 K-뷰티 정부지원사업 전문가야. 아래 기업 조건과 지원사업 조건을 분석해서 매칭 이유를 2~3문장으로 설명해줘."
- API 프록시: `POST /api/match` → `{ supportId, userInput }` → Claude 호출 → 이유 반환

**5. 대안 지원 추천 고도화**
- ineligible 카드의 `alt-suggestion`을 하드코딩이 아닌 AI 추천으로
- "현재 조건에 안 맞지만 가장 가까운 지원사업 2개를 추천해줘" 프롬프트

### P2 — 크롤러 구축

**6. 지원사업 자동 수집**
- 크롤링 대상: exportvoucher.com, kotra.or.kr, mss.go.kr, smtech.go.kr
- 수집 필드: 지원명, 기관, 마감일, 지원금액, 자격조건, 신청링크
- 스케줄: 매주 1회 실행 → `/db/supports.json` 업데이트
- 기술 스택 후보: Puppeteer (Node.js) 또는 Python Playwright

**7. 관리자 페이지**
- `/admin/index.html` — 지원사업 목록 조회·수정·추가·비활성화
- `is_active` 필드로 마감된 지원 숨김 처리

### P3 — 사용자 경험 개선

**8. 북마크 / 관심 지원 저장**
- localStorage에 북마크한 지원사업 ID 저장
- 결과 화면에 "북마크" 버튼 추가

**9. 마감 임박 알림**
- 이메일 입력 → 관심 지원사업 마감 7일 전 알림
- 백엔드: Resend API로 이메일 발송

**10. SKUCheck 연동**
- SKUCheck에서 분석 완료 후 "지원사업 매칭받기" 버튼으로 SupportDesk 연결
- SKUCheck 결과(타겟국가, 인증 현황)를 SupportDesk 입력값으로 자동 채우기

---

## Deployment

### Vercel (권장)
```bash
# 빌드 없음. HTML 파일 직접 배포
vercel deploy --prod
# 또는 vercel.com 대시보드에서 파일 업로드
```

### GitHub Pages
```bash
git init && git add . && git commit -m "init: SupportDesk prototype"
git remote add origin <repo-url>
git push -u origin main
# Settings → Pages → main branch / root
```

---

## Known Issues

| 이슈 | 영향 | 해결 방법 |
|------|------|---------|
| DB 6개밖에 없음 | 매칭 결과 빈약 | P0: 지원사업 30개로 확장 |
| AI 이유가 하드코딩 | 조건 변화 반영 안 됨 | P1: Claude API 연동 |
| 마감일이 정적 | 지난 지원도 표시 가능 | P0: is_active 필드로 필터링 |
| 크롤링 없음 | 신규 지원사업 반영 불가 | P2: 크롤러 구축 |
| 모바일 레이아웃 일부 깨짐 | UX 저하 | P1: 반응형 개선 |

---

## Disclaimer

본 서비스가 제공하는 지원사업 매칭 결과는 참고용 1차 정보입니다.
실제 자격 조건 및 신청 가능 여부는 각 기관의 공식 공고를 반드시 확인하세요.
지원사업 정보는 변경될 수 있으며, 최신 정보는 각 기관 공식 사이트를 기준으로 합니다.

---

*v0.1 prototype · Last updated: 2026.05*
