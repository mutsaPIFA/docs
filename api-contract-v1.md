# MCM MUSE — API 계약서 & 결정 사항 v1

- 작성일: 2026-08-13
- 상태: v1 인터페이스 합의 (모양은 고정, 속은 추후 수정 가능)

> 이 문서 목적: 프론트/AI/백엔드가 **이 JSON 모양대로 목업 붙여 병렬로** 달리기 위한 계약.
> 못 박는 건 "모양(엔드포인트+요청/응답)", 바꾸는 건 "속(구현·로직)".

---

## 1. MVP 범위 (토요일 = 2026-08-15 목표)

**포함 화면:** 0 스플래시 · 1 온보딩 · 4-b 로그인/회원가입 · **9 나의옷장(closet 탭)** · 2 옷장스캔 · 3 스캔결과(단일) · **home 상품보기(MCM 카탈로그)** · 6 제품상세 · 4-a 스타일DNA · 5 MCM추천 · 15 무드선택 · 16 코디조합추천(3후보) · profile 탭
**보류:** 7 AR · 8 예약및결제 · 10 코디기록(캘린더 화면) · 11~13 아카이브/마켓 계열 화면
**삭제:** 14 디지털옷장(9와 중복) · 4-b 로그인요청

> **하단 탭(MVP): `home(상품보기) · closet · style · profile`.** home은 피그마에 화면이 없어 우리가 정의했다 — 검색바 + 카테고리 탭(전체/BAGS/ACC/CLOTHES) + 상품 그리드. 검색·탭 필터는 프론트 클라이언트 처리.

**세 갈래 흐름**
- **A. 스캔→편입:** `2 스캔 → 3 결과(원본+누끼 확인, 재스캔/등록) → 옷장 저장`
- **B. 추천:** `내 옷장에서 여러 옷 선택 → 4-a 스타일DNA → 5 추천(베스트1픽+더보기) → 6 제품상세`
- **C. 큐레이터:** `무드 선택(15) → 코디 최대 3개 생성(16) → 택1 저장(Look + 이미지 비동기 생성)`

---

## 2. 핵심 원칙

1. **큐레이팅 = 반드시 MCM을 포함한 룩.** 그래서 옷장의 `source(OWN/MCM)` 구분이 큐레이터의 핵심 입력. (MCM 없는 코디는 이 서비스의 코디가 아님)
2. **LLM 응답 id는 DB로 재검증** 후 사용 (환각 차단). 실재 데이터만 프롬프트에 넣음.
3. **편입은 실제 DB 반영**(리텐션 루프의 경첩). 결제·정품인증은 목업.
4. **결제 추적 불가** → MCM 편입은 사용자 명시 행동(촬영/검색/구매후등록)으로 처리. (구매 유도 아님)
5. **이미지는 `StorageService` 인터페이스로 추상화** → 데모=로컬, 프로덕션=S3+CDN. 계약(URL 필드)은 불변.
6. **코디 이미지 생성(B)은 '저장'한 룩 1개에만** → 후보 3개 전부 생성 안 함(비용·지연 1/N).

---

## 3. 데이터 모델

```
User        { id, email, nickname }
ClosetItem  { id, userId, category, color, material, mood,
              imageUrl, cutoutUrl, source(OWN|MCM), mcmProductId(nullable), createdAt }
McmProduct  { id, name, category, color, material, price,
              imageUrl, cutoutUrl(nullable), productUrl }
Recommendation (transient, 미저장)
            { id?, mcmProductId, reason, pairsWithItemIds[], isExpansion, product{...} }
Mood        { id, label, labelEn, iconKey }              // 고정 시드 6개
Outfit      (코디 후보, transient·미저장)
            { moodId, occasionLabel, closetItems[{id,cutoutUrl,category}],
              mcmProduct{id,imageUrl,cutoutUrl,name}, reason }   // MCM은 정확히 1개
Look        (저장된 룩, 영속)
            { id, userId, wornDate, moodId, occasionLabel,
              closetItemIds[], mcmProductId, reason, generatedImageUrl(nullable) }
```

**controlled vocabulary (옷장·MCM 크롤링 라벨링 공통, v1 — 추후 수정 가능)**
- `category` = {상의, 하의, 아우터, 원피스, 신발, 가방, 악세서리}
- `color` = {블랙, 화이트, 네이비, 그레이, 베이지, 브라운, 카멜, 그린, 핑크, 기타}
- `material` = {면, 니트, 데님, 가죽, 실크, 울, 합성, 기타}
- `mood` = {미니멀, 캐주얼, 클래식, 스트릿, 페미닌, 럭셔리}

**무드 시드 6개 (GET /moods)**
| id | label | labelEn | iconKey |
|----|-------|---------|---------|
| 1 | 저녁 약속 | DINNER DATE | dinner |
| 2 | 출근 | OFFICE | office |
| 3 | 출장 | BUSINESS TRIP | trip |
| 4 | 데일리 | CASUAL | daily |
| 5 | 주말 산책 | WEEKEND WALK | walk |
| 6 | 파티 | EVENT | party |

---

## 4. API 계약

> **엔드포인트 정의는 [`api-v1.md`](api-v1.md)가 단일 기준이다.**
> 요청/응답 스키마·필드·에러·상태코드는 전부 그 문서에 있고, 이 문서는 **왜 그렇게 정했는지**(결정 로그·데이터 모델·화면 맵)만 다룬다.
> 예전엔 여기에도 엔드포인트 요약을 중복해서 적어 두었으나, 두 문서가 어긋나 프론트가 낡은 쪽을 보는 사고가 실제로 생겨 삭제했다. **엔드포인트를 이 문서에 다시 적지 말 것.**

핵심 규약만 요약(상세는 `api-v1.md`):

- prefix `/api/v1` · 인증 `Authorization: Bearer <accessToken>` (`/auth/**` 제외) · 페이지네이션 없음(v1)
- 성공 응답 래퍼 없음(데이터 직접) · 에러 `{status, message, code?}`
- 인증 = access token(body) + refresh token(HttpOnly 쿠키)
- 이미지는 절대 URL · 날짜 ISO 8601
- `POST /outfits`는 옷장에 MCM이 없으면 **409 `NO_MCM_IN_CLOSET`**
- 스캔은 프론트가 결과를 보관했다가 `/closet-items`에 태그 포함 전체 전송(태그 수정 가능)

---

## 5. 이미지 저장 아키텍처 (스케일 대비)

- 이미지는 **오브젝트 스토리지**(S3 / Cloudflare R2 / NCP Object Storage). DB엔 **URL/key만**.
- 서빙은 **CDN** 앞단. 데모는 앱 서버 로컬 저장 + `/images/{id}` 정적 서빙.
- 업로드: 데모=서버 경유(비전+rembg 처리 필요). 스케일=presigned 직접 업로드 + 큐(SQS)+워커 비동기.
- 등록 안 된 임시 스캔 이미지 = S3 lifecycle TTL 자동 삭제.
- **지금 할 것: `StorageService` 인터페이스만 추상화** → Local↔S3 무통증 교체.

---

## 6. 결정 로그

| 항목 | 결정 |
|---|---|
| Q1 유저/인증 | 회원가입/로그인 구현(실제 userId). 토요 프로토도 이 위에서 동작 |
| Q2 스캔 저장 시점 | 스캔=임시결과(원본+누끼 확인), '등록' 버튼으로 DB 저장. 한 벌씩 |
| Q3 DNA/추천 콜 | 2콜 (style-dna → recommendations). 추천은 베스트1픽+더보기 |
| Q4 코디 이미지 | 백엔드(근엽) 담당. 계약은 A(조합 데이터). B(생성 이미지)는 저장 시점에만, 필드 추가로 확장 |
| Q5 코디 개수 | 무드 1개 → 최대 3개 후보 → 택1. 옷 적으면 3개 이하 |
| Q6 코디 MCM 소스 | 옷장의 MCM(source=MCM). 제품상세 큐레이팅은 seedMcmProductId |
| Q7 태그 enum | 위 vocabulary로 확정(추후 수정 가능) |
| Q8 이미지 저장 | 로컬→S3+CDN, StorageService 추상화 |
| Q9 저장 | 추천·DNA 미저장(매번 생성). 저장한 룩(Look)만 영속 |
| N1 촬영-MCM 매칭 | 카탈로그 매칭 안 함(source=MCM, mcmProductId=null) |
| N2 코디에 MCM 0개 | **409 + `code:NO_MCM_IN_CLOSET`** (빈 배열 아님 — 배열엔 code를 실을 곳이 없음) |
| N3 MCM 검색 | 서버에 query(이름)+category 파라미터 제공하되, **v1 프론트는 전체 받아 클라 필터** |
| ⑥ isExpansion | MVP 단순 매칭, 필드만 두고 항상 false(로직은 P1) |

**추가 결정 (2026-08-13, 피그마 수정본 대조 후)**

| 항목 | 결정 |
|---|---|
| D1 인증 화면 | 데모용 계정 자동로그인 안 함. **프론트가 0 스플래시·1 온보딩·4-b 로그인/회원가입을 실제 구현.** `/auth/**` 계약은 변경 없음 |
| D2 룩 이미지 생성 | **비동기.** `POST /looks`는 즉시 201(+`generatedImageUrl=null`), 프론트가 **`GET /looks/{id}` 폴링**으로 교체. 동기 대기는 타임아웃 위험이라 배제 |
| D3 프로필 | **`GET /me` 추가** — profile 탭이 하단탭 4개 중 하나인데 닉네임 띄울 API가 없었음 |
| D4 MCM 누끼 | **`McmProduct.cutoutUrl` + `Outfit.mcmProduct.cutoutUrl` 추가.** 없으면 화면 16 콜라주에서 MCM만 흰 배경으로 튐. 시드 적재 시 rembg로 생성 |
| D5 코디 내 MCM 수 | **정확히 1개**(스키마 단수 유지). "1개 이상"은 *옷장에 MCM이 최소 1개 있어야 큐레이팅 가능*이라는 전제조건(=N2)과 혼동된 표현이었음 |
| D6 구매하기 | 목업 모달 아님 — **`productUrl`로 MCM 공식몰 이동.** 필드가 이미 있어 추가 개발 0이고 실제로 동작함 |
| D7 ♡찜 | **v1 제외**, 아이콘 미노출. '옷장에 담기'가 유사 역할을 이미 함 |
| D8 home 화면 | 피그마에 없어 우리가 정의 — 검색바 + 탭(전체/BAGS/ACC/CLOTHES) + 그리드. **UI 탭은 4개지만 내부 `Category`는 7분류 유지**(스타일링 엔진이 상의/하의/아우터를 구분해야 함) |
| D9 wornDate | 필수 → **옵션, 미전송 시 서버가 오늘 날짜.** 화면 16에 날짜 선택 UI가 없음 |
| D10 옷장 정렬 | `GET /closet-items`는 **`createdAt DESC`** 고정 |
| D11 문서 경계 | 엔드포인트 정의는 `api-v1.md` 단일 기준. 이 문서 §4의 중복 코드블록은 삭제(두 문서가 어긋나 사고 발생) |
| D12 BC 분할 | 백엔드를 **`auth` / `catalog` / `closet` / `styling` + `shared`** 로 분할(tech-blog DDD-Lite 컨벤션 준용). `home`은 화면명이라 도메인명 `catalog`로, 추천(DNA·MCM추천)은 closet이 아니라 `styling`으로, `profile`은 엔티티가 없어 당분간 `auth`에 흡수 |
| D13 화면 라벨 위치 | **D11 일부 조정** — 화면 배지를 `api-v1.md` 각 엔드포인트에 달고, 문서 끝에 "화면 → 호출 API" 역인덱스를 둔다. 대신 이 문서 §8 화면맵에서는 API 대응을 뺀다. **프론트가 읽는 파일에 라벨을 몰아 단일 소스를 유지**하려는 것 |
| D14 문서 구성 | `api-v1.md` 섹션 순서 = BC 순서(auth→catalog→closet→styling). 섹션 번호만 봐도 담당 BC를 알 수 있게 함 |

---

## 7. 열린 항목 / 파킹랏 (리셋 후 이어서)

- ⚠️ **LLM provider·API 키 확정** — 비전 태깅(Gemini/GPT-4o), 추천 LLM, 코디 이미지 생성(Nano Banana). 누가 키 준비? (AI 담당 시은과 협의)
- **코디 이미지 생성(B) 방법 스파이크** — Nano Banana 멀티이미지 합성 vs 실물 MCM 누끼 픽셀 오버레이. "저장 시점 1장만 생성"으로 비용 절감.
- **상품 수집** — MCM 공식몰(`kr.mcmworldwide.com`)은 **Cloudflare 관리형 챌린지로 서버 HTTP 크롤링이 차단**됨(브라우저 접속은 정상). 반면 이미지 CDN(`images.mcmworldwide.com/i/mcmworldwide/{SKU}_01`)은 서버에서 접근 가능. → **브라우저 세션으로 시드 카탈로그(150~200개) 수확 + 주최측에 공식 피드 요청 병행.** 카테고리별 최소 수량을 강제해 코디 조합이 성립하게 함(가방만 있으면 화면 16이 무의미). 서버 상시 크롤러는 파킹랏.
- **입력 이미지** — jpg/png (+iPhone HEIC 변환 주의), 최대 크기 제한.
- **후순위/P1·P2** — 캘린더 화면(10) · 취향 확장 큐레이터(P1) · 아카이브 거래(P2) · AR(7) · 예약및결제(8)
- **결제 로드맵** — v1은 `productUrl` 딥링크(우리가 결제 처리 안 함) → 제휴 성사 시 커머스 API 위임 검토. 아카이브 C2C(11~13)만 에스크로 PG가 별도로 필요.
- ❓ **화면 8 "예약및결제" 기획 의도 확인 필요** — 화면 7이 AR인 걸 보면 *오프라인 매장 방문 예약 + 예약금*일 가능성. 순수 이커머스 결제와 설계가 다름(매장 재고·타임슬롯).
- **팀 오너십** — LLM 프롬프트·추천 로직=AI(시은), API·id검증·DB·저장=백엔드(근엽·선재)

---

## 8. 화면 맵 (피그마 "디자인 수정" 새 번호)

| # | 화면 | 상태 | # | 화면 | 상태 |
|---|---|---|---|---|---|
| 0 | 스플래시 | MVP | 9 | 나의옷장 (= closet 탭) | MVP |
| 1 | 온보딩 | MVP | 10 | 코디기록(캘린더) | 보류 |
| 2 | 옷장스캔 | MVP | 11 | 아카이브마켓 | 보류 |
| 3 | 옷장스캔결과(단일) | MVP | 12 | 아카이브 상세 | 보류 |
| 4-a | 스타일DNA | MVP | 13 | 아카이브 출품 | 보류 |
| 4-b | 로그인/회원가입 | MVP | ~~14~~ | ~~디지털옷장~~ | **삭제(9와 중복)** |
| 5 | MCM제품추천 | MVP | 15 | 무드선택 | MVP |
| 6 | 제품상세 | MVP | 16 | 코디조합추천(3후보) | MVP |
| 7 | AR로 입어보기 | 보류 | 17 | 프로필 (= profile 탭) | MVP |
| 8 | 예약및결제 | 보류 | — | **home 상품보기** | **MVP(피그마 없음)** |

> ⚑ **이 표는 화면 목록과 MVP 여부만 관리한다.** 화면 ↔ API 대응은 **[`api-v1.md`](api-v1.md)** 가 단일 소스다 — 각 엔드포인트에 화면 배지가 붙어 있고, 문서 끝에 "화면 → 호출 API" 역방향 인덱스가 있다. 프론트가 읽는 파일에 라벨을 몰아둔 것이므로 **여기에 API를 다시 적지 말 것.**
>
> 참고로 화면↔API는 **N:M**이다. `GET /closet-items` 하나가 화면 9·3·4-a에서 쓰이고, 화면 16 하나가 `/outfits`+`/looks`+`/looks/{id}`를 쓴다. 그래서 엔드포인트 이름·구조는 리소스 기준으로 두고 화면 번호를 경로에 반영하지 않는다.
>
> **화면 17 프로필**은 하단탭 4개 중 하나라 MVP로 승격됐다(`GET /me` + 로그아웃).
> **home 상품보기**는 피그마에 프레임이 없어 번호가 없다. 정의는 §1 참조 — 디자이너에게 이 정의를 넘겨 프레임을 만들어야 한다.
