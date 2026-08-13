# MCM MUSE — 시스템 아키텍처 개요

- 작성일: 2026-08-13 · 상태: v1 (팀 합의용 초안)
- 대상 독자: 프론트·AI·백엔드·디자인 **전원**
- 짝 문서: [`api-contract-v1.md`](api-contract-v1.md) (엔드포인트·데이터모델) · [`research-vision-llm.md`](research-vision-llm.md) (기술 선택)

> 이 문서는 "큰 그림"을 잡는 용도입니다. 특정 역할 관점이 아니라 **시스템 전체가 어떻게 맞물리는지**를 공유하고, 각 역할이 자기 파트를 어디에 끼우는지 알 수 있게 합니다.

---

## 1. 제품 한 줄

> **내 옷장에서 시작해 → MCM을 더하고 → 다시 옷장으로 스타일링하는 한 바퀴.**

옷장 스캔으로 내 옷을 디지털화하고, 스타일 DNA를 뽑아 MCM 제품을 추천하고, 무드에 맞춰 코디를 큐레이팅해 다시 옷장에 쌓는 **리텐션 루프**가 핵심.

## 2. 목표 (2026-08-15, 토요일)

뼈대만 세우는 게 아니라 **아래 흐름이 실제로 동작**하는 것이 목표. (목업은 병렬 개발을 막지 않기 위한 임시 수단일 뿐, 산출물이 아님)

- **A. 스캔→편입:** 옷 촬영 → 태깅+누끼 → 내 옷장에 실제 저장
- **B. 추천:** 옷장에서 선택 → 스타일 DNA → MCM 제품 추천 → 제품 상세
- **C. 큐레이터:** 무드 선택 → MCM 포함 코디 후보 → 택1 저장(코디 이미지 생성)

## 3. 구성도

```
┌──────────────┐   HTTPS /api/v1    ┌───────────────────────────┐
│  Client       │ ─────────────────▶ │  Backend (Spring Boot)     │
│  React 웹앱    │ ◀───────────────── │  · 공개 API · 인증(JWT)      │
│  (→ 모바일 앱) │      JSON           │  · 비즈니스 로직             │
└──────────────┘                     │  · AI 응답 id DB 재검증      │
                                      └───┬───────────┬────────────┘
                             내부 HTTP      │           │  JPA
                                   ┌────────▼──┐    ┌───▼────────┐   ┌──────────────┐
                                   │ AI 서비스  │    │ PostgreSQL │   │ Object Storage│
                                   │ (FastAPI) │    │            │   │ (이미지 URL)   │
                                   │ rembg+    │    └────────────┘   │ 데모=로컬/CDN  │
                                   │ Gemini    │                     └──────────────┘
                                   └───────────┘
```

## 4. 컴포넌트 역할 (역할 = 레포)

| 컴포넌트 | 레포 | 책임 | 스택 |
| --- | --- | --- | --- |
| **Client** | `frontend` | 화면·UX, `/api/v1` 소비. 코디는 누끼 콜라주로 렌더 | React + Vite + TS, 모바일 우선 반응형 |
| **Backend** | `backend` | 공개 `/api/v1`, 인증, DB 영속, AI 오케스트레이션·**id 재검증**, 이미지 저장 추상화 | Java 21 · Spring Boot 3.x · PostgreSQL |
| **AI 서비스** | `ai` | 태깅·누끼·추천·코디·이미지 생성. 내부 API만 | Python FastAPI · rembg · Gemini |
| **공유 문서** | `docs` | 계약·아키텍처·리서치 | Markdown |

**경계 원칙**
- 공개 계약(`/api/v1`)은 **backend가 소유**. Client·AI는 이 계약/내부 계약의 JSON 모양에만 의존.
- **AI 응답은 신뢰 대상이 아님** — backend가 반환된 `id`를 DB로 재조회해 **실재하는 것만** 응답에 실음(환각 차단).
- 이미지는 **URL/key로만** 오간다. 저장 구현(로컬↔S3)은 `StorageService` 뒤로 숨겨 무통증 교체.

## 5. 클라이언트: 웹앱 우선 → 모바일 앱

`backend`가 순수 REST API라 클라이언트는 **교체·확장 가능한 소비자**. 그래서 웹으로 시작해 나중에 앱으로 확장 가능.

| 경로 | 설명 | 특징 |
| --- | --- | --- |
| PWA | 웹앱에 manifest+SW → 홈화면 설치 | 최저비용, 앱스토어 X |
| **Capacitor** | 같은 React 코드를 네이티브 셸로 래핑 | **코드 1벌**, 카메라·파일 네이티브, 웹→앱 최단경로 |
| React Native | 네이티브 UI 재작성 | 네이티브감 최고, 로직/타입만 공유 |

**지금 지킬 것(경로는 나중에 확정해도 됨):** API-first · 모바일 우선 반응형 · 카메라/파일은 표준 웹 API. 이 3개만 지키면 어느 경로든 열림.

## 6. 데이터 흐름 (세 갈래)

```
A 스캔→편입 :  [카메라] → POST /scan (backend→AI: 태깅+누끼) → 확인 → POST /closet-items(실제 저장)
B 추천      :  옷장 선택 → POST /style-dna → POST /recommendations(후보 id → backend가 DB 재검증) → 제품상세
C 큐레이터   :  GET /moods → POST /outfits(MCM 포함 코디 최대 3, transient) → 택1 → POST /looks(저장 + 코디 이미지 1장 생성)
```

- **큐레이팅=반드시 MCM 포함 룩** → 옷장 아이템의 `source(OWN/MCM)` 구분이 큐레이터의 핵심 입력.
- 저장 정책: 추천·DNA·코디후보는 **transient(미저장)**, 저장한 룩(`Look`)만 영속.

## 7. AI 통합 방식

- backend는 AI 기능을 **포트(인터페이스)**로 두고, 구현을 **HTTP(ai 서비스 호출)** 또는 **목업**으로 스위치.
- 이유: **Gemini 키·모델 확정(AI 담당) 전에도** 프론트·백엔드가 계약 모양대로 병렬 개발 가능. 키 확정 즉시 HTTP로 전환.
- AI 서비스 내부 API는 공개 `/api/v1`과 **별개**이며 backend만 호출한다(내부망).
  **정의는 `ai` 레포 [`docs/internal-api.md`](https://github.com/mutsaPIFA/ai/blob/main/docs/internal-api.md)가 단일 기준** — 여기에 목록을 중복해 적지 않는다.
- 룩 이미지 생성은 수십 초가 걸릴 수 있어 **비동기**로 처리한다. `POST /looks`는 즉시 응답하고 이미지는 나중에 채워진다(공개 계약 `api-v1.md` §4-5).

## 8. 배포 / 로컬 실행

- **로컬·데모:** 루트 `docker-compose`로 `postgres + backend + ai` 한 번에. Client는 dev 서버(또는 정적 빌드).
- **최종 배포:** 각 서비스 = 컨테이너. compose가 "합쳐서 배포"의 기본 단위.
- 시크릿(Gemini 키 등)은 `.env`로 주입, 레포에 커밋 금지.

## 9. 확장·스케일 (파킹랏)

- 이미지: 로컬 → **S3/R2 + CDN**, 업로드는 presigned + 큐(SQS)+워커 비동기.
- 가상 피팅 비용↑ 시: 오픈소스 VTON(CatVTON 등) self-host(GPU) 검토. (`research-vision-llm.md`)
- 취향 확장 큐레이터·아카이브 마켓·AR·결제는 MVP 이후 P1/P2.

## 10. 레포 · 문서 배치

```
mutsaPIFA/
├─ docs       팀 공용 문서(계약·아키텍처·리서치)  ← 여기
├─ backend    Spring Boot · 공개 API · DB · 루트 docker-compose · 백엔드 구현 상세 docs
├─ frontend   React 웹앱 · 프론트/디자인 docs(Figma 컨벤션)
└─ ai         FastAPI · rembg + Gemini · AI 구현 상세 docs
```

> 팀 공용 문서만 `docs`. 역할 전용 문서는 각 역할 레포의 `docs/`.

## 11. 열린 항목

- Gemini API 키·모델명·쿼터 (AI 담당) — 미해결 시 목업 포트로 계속 진행.
- MCM 제품 데이터 확보 — 공식몰이 Cloudflare 챌린지로 **서버 크롤링 차단**(브라우저는 정상, 이미지 CDN은 접근 가능). **브라우저 세션 수확으로 시드 150~200개 + 주최측 공식 피드 요청 병행.** 소스는 `ProductSource` 인터페이스 뒤로 숨겨 교체 가능하게. 상세: `api-contract-v1.md` §7.
- ~~화면 용어 정리: "나의옷장 ↔ 디지털옷장" 역할 중복~~ → **해결: 14 디지털옷장 삭제, 9 나의옷장이 closet 탭.**
- 클라이언트 모바일 전환 경로(PWA/Capacitor/RN) 최종 선택 시점.
