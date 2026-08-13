# MCM MUSE — API 정의서 v1

> 프론트·AI·백엔드 공통 API 계약. 이 문서가 **엔드포인트의 최종 기준**입니다.
> 결정 배경·데이터모델 근거는 [`api-contract-v1.md`](api-contract-v1.md), 큰 그림은 [`architecture.md`](architecture.md).

## 개요

| 항목 | 내용 |
|------|------|
| Base URL | `/api/v1` |
| 인증 | `Authorization: Bearer {accessToken}` (`/auth/**`, `/images/**` 제외) |
| Content-Type | `application/json` (`POST /scan`만 `multipart/form-data`) |
| 성공 응답 | **래퍼 없이 데이터 직접 반환** (`response.status`로 성공 구분) |
| 에러 응답 | `{ status, message, code? }` (아래 공통 에러) |
| 날짜 형식 | ISO 8601 UTC (`2026-08-13T10:00:00Z`), 날짜만 필요한 필드는 `yyyy-MM-dd` |
| 이미지 | **절대 URL** 반환 (프론트 `<img src>` 그대로). 데모=서버 로컬, 배포=S3/CloudFront |
| 페이지네이션 | **v1 없음** — 목록은 전체 반환 |

---

## 문서 읽는 법

각 엔드포인트 제목 아래에 **배지**가 붙는다.

> `화면 9 나의옷장` · `인증 필요` · `MVP`

| 배지 | 뜻 |
|------|-----|
| `화면 N …` | 이 API를 호출하는 화면. 프론트는 여기만 보고 "이 화면에서 뭘 부르지"를 판단하면 된다 |
| `인증 필요` / `인증 불필요` | `Authorization: Bearer` 헤더 필요 여부 |
| `MVP` / `후순위` | 2026-08-15 데모 범위인지 |

**"화면 → 호출 API" 역방향 인덱스는 문서 맨 아래에 있다.** 화면부터 시작할 땐 거기서 찾는 게 빠르다.

## 도메인 그룹

API는 4개 도메인 그룹으로 나뉜다. 문서의 섹션 순서가 이 그룹을 따른다.

| § | 도메인 그룹 | 담당 화면 |
|---|------------|----------|
| 1 | **인증 · 사용자** | 0 스플래시 · 1 온보딩 · 4-b 로그인/회원가입 · 17 프로필 |
| 2 | **MCM 제품 카탈로그** | home 상품보기 · 6 제품상세 |
| 3 | **옷장** | 9 나의옷장 · 2 옷장스캔 · 3 스캔결과 |
| 4 | **스타일링** | 4-a 스타일DNA · 5 MCM추천 · 15 무드선택 · 16 코디조합추천 |

## 계약 변경 절차

이 문서는 프론트·AI가 동시에 의존하므로 **혼자 고치지 않는다.**

1. `docs` 레포에서 `feature/*` 브랜치 → **PR (base `dev`)**
2. **영향받는 역할을 리뷰어로 지정** — 프론트가 쓰는 필드가 바뀌면 프론트, AI 응답 모양이 바뀌면 AI
3. 승인 후 머지, 변경 사실을 팀 채널에 공유

PR 본문에 **무엇이 어떤 화면에 영향을 주는지** 한 줄로 적을 것. (예: "`POST /looks` 이미지 비동기 전환 — 화면 16 폴링 필요")

## 흐름 개요

| 흐름 | 경로 | 화면 |
|------|------|------|
| **A. 스캔→편입** | `POST /scan` → (확인) → `POST /closet-items` | 2, 3, 9 |
| **B. 추천** | `POST /style-dna` → `POST /recommendations` → `GET /mcm-products/{id}` | 4-a, 5, 6 |
| **C. 큐레이터** | `GET /moods` → `POST /outfits` → `POST /looks` | 15, 16 |

---

## 공통 타입

옷장 아이템·MCM 제품의 라벨은 아래 controlled vocabulary로 정규화된다 (v1, 추후 확장 가능).

```ts
type Category = '상의' | '하의' | '아우터' | '원피스' | '신발' | '가방' | '악세서리'
type Color    = '블랙' | '화이트' | '네이비' | '그레이' | '베이지' | '브라운' | '카멜' | '그린' | '핑크' | '기타'
type Material = '면' | '니트' | '데님' | '가죽' | '실크' | '울' | '합성' | '기타'
type ItemMood = '미니멀' | '캐주얼' | '클래식' | '스트릿' | '페미닌' | '럭셔리'   // 옷장 아이템 무드 태그
type Source   = 'OWN' | 'MCM'   // 내 옷 / 보유·담은 MCM
```

> ⚠️ 용어 주의: 위 `ItemMood`(아이템 태그)와 큐레이터의 **`Mood`(무드 선택, §4-3)** 는 다르다. `Mood`는 `{id,label,labelEn,iconKey}` 고정 시드 6개.

---

## 공통 응답 / 에러

### 성공
래퍼 없이 데이터를 그대로 반환한다. 생성은 `201`, 삭제는 `204`(body 없음 — JSON 파싱 금지).

### 에러
모든 에러는 같은 형식.

```json
{ "status": 409, "message": "코디를 만들려면 옷장에 MCM 제품이 있어야 합니다.", "code": "NO_MCM_IN_CLOSET" }
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `status` | `number` | HTTP 상태 코드 |
| `message` | `string` | 사용자에게 보여줄 수 있는 한국어 메시지 |
| `code` | `string?` | 프론트가 분기해야 하는 비즈니스 에러만 부여 (아래) |

| HTTP | 의미 |
|------|------|
| `400` | 요청 값 검증 실패 |
| `401` | 인증 토큰 없음/만료 |
| `403` | 권한 없음 (본인 리소스 아님 등) |
| `404` | 리소스 없음 |
| `409` | 상태 충돌 / 비즈니스 선행조건 위반 |

**비즈니스 `code`**

| code | HTTP | 상황 |
|------|------|------|
| `NO_MCM_IN_CLOSET` | `409` | `POST /outfits` — 옷장에 MCM(`source=MCM`) 아이템이 없음 |

### 인증 구조

| 토큰 | 위치 | 유효기간(기본) | 용도 |
|------|------|------|------|
| Access Token | 응답 body → **메모리 저장 권장** | 30분 | `Authorization: Bearer` 헤더 |
| Refresh Token | **HttpOnly 쿠키**(자동 관리) | 14일 | Access 갱신 (`POST /auth/refresh`) |

- 쿠키를 쓰므로 프론트는 **`withCredentials: true`**(axios) / `credentials: 'include'`(fetch) 필수.
- Access 만료 시 `401` → `POST /auth/refresh`로 갱신 후 재시도, refresh도 만료면 로그인 이동.
- 공개 엔드포인트: `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `GET /images/**`. 그 외 전부 인증 필요.

```js
const api = axios.create({ baseURL: '/api/v1', withCredentials: true })
```

---

# 1. 인증 · 사용자

### 1-1. 회원가입

> `화면 4-b 로그인/회원가입` · `인증 불필요` · `MVP`

```
POST /auth/register
```

**Request Body**
```json
{ "email": "muse@example.com", "password": "password123", "nickname": "뮤즈" }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `email` | `string` | 예 | 이메일 형식, 중복 불가 |
| `password` | `string` | 예 | 최소 8자 |
| `nickname` | `string` | 예 | — |

**Response `201 Created`** (+ `Set-Cookie: refresh_token=...; HttpOnly`)
```json
{ "userId": 1, "accessToken": "eyJhbGciOiJIUzI1NiJ9...", "tokenType": "Bearer" }
```

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 이메일 중복 | `409` | 이미 가입된 이메일입니다 |
| 검증 실패 | `400` | 값이 올바르지 않습니다 |

---

### 1-2. 로그인

> `화면 4-b 로그인/회원가입` · `인증 불필요` · `MVP`

```
POST /auth/login
```

**Request Body**
```json
{ "email": "muse@example.com", "password": "password123" }
```

**Response `200 OK`** (+ `Set-Cookie: refresh_token=...; HttpOnly`)
```json
{ "accessToken": "eyJhbGciOiJIUzI1NiJ9...", "tokenType": "Bearer" }
```

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 자격 불일치 | `401` | 이메일 또는 비밀번호가 올바르지 않습니다 |

---

### 1-3. 토큰 갱신

> `전역 (401 발생 시)` · `쿠키 인증` · `MVP`

```
POST /auth/refresh
```

> Refresh Token 쿠키로 새 Access Token 발급. 쿠키 자동 전송(`withCredentials`). 새 refresh 쿠키도 갱신됨.

**Response `200 OK`**
```json
{ "accessToken": "eyJhbGciOiJIUzI1NiJ9...", "tokenType": "Bearer" }
```

| 에러 | HTTP | 메시지 |
|------|------|--------|
| refresh 없음/만료 | `401` | 다시 로그인해 주세요 |

---

### 1-4. 로그아웃

> `화면 17 프로필 (profile 탭)` · `쿠키 인증` · `MVP`

```
POST /auth/logout
```

> refresh 쿠키를 만료시킨다. 프론트는 메모리의 Access Token도 제거.

**Response `204 No Content`**

---

### 1-5. 내 정보

> `화면 17 프로필 (profile 탭)` · `인증 필요` · `MVP`

```
GET /me
```

> 로그인한 사용자 정보. profile 탭에서 닉네임 표시·로그아웃 진입에 사용.

**Response `200 OK`**
```json
{ "userId": 1, "email": "muse@example.com", "nickname": "뮤즈" }
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `userId` | `number` | 사용자 id |
| `email` | `string` | 가입 이메일 |
| `nickname` | `string` | 표시 이름 |

---

# 2. MCM 제품 카탈로그

### 2-1. 제품 검색 / 목록

> `home 상품보기` · `인증 필요` · `MVP`

```
GET /mcm-products?query=&category=
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `query` | `string` | 아니오 | 이름 검색 |
| `category` | `Category` | 아니오 | 카테고리 필터 |

> ⚑ **v1 프론트는 두 파라미터를 사용하지 않는다.** home 진입 시 전체를 한 번 받아 검색·탭 전환을 클라이언트에서 처리한다(시드 150~200개 규모). 파라미터는 서버에 구현해 두되 향후 데이터 증가·타 클라이언트 대비용.

**Response `200 OK`** — `McmProduct[]`
```json
[
  { "id": 103, "name": "Tracy 숄더백", "category": "가방", "color": "카멜", "material": "가죽",
    "price": 890000, "imageUrl": "https://...", "cutoutUrl": "https://...",
    "productUrl": "https://..." }
]
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `imageUrl` | `string` | 상품 원본 이미지 절대 URL |
| `cutoutUrl` | `string?` | **누끼 이미지** 절대 URL. 화면 16 콜라주 렌더에 사용. 적재 시 생성하며 실패 시 `null` |
| `productUrl` | `string` | MCM 공식몰 상품 페이지. 화면 6 "구매하기"가 이 URL로 이동 |

**home 카테고리 탭 ↔ `Category` 매핑** (프론트 클라이언트 필터)

| 탭 | 포함 `Category` |
|----|----------------|
| 전체 | 전부 |
| BAGS | 가방 |
| ACC | 악세서리 |
| CLOTHES | 상의, 하의, 아우터, 원피스, 신발 |

---

### 2-2. 제품 상세

> `화면 6 제품상세` · `인증 필요` · `MVP`

```
GET /mcm-products/{id}
```

**Response `200 OK`** — `McmProduct` (2-1 객체와 동일 구조)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 없음 | `404` | 제품을 찾을 수 없습니다 |

> 화면 6의 **"구매하기"는 `productUrl`로 MCM 공식몰 이동** (우리가 결제를 처리하지 않는다). 인앱 브라우저로 띄우는 것을 권장.

---

# 3. 옷장

### 3-1. 옷 스캔 (태깅 + 누끼)

> `화면 2 옷장스캔` · `인증 필요` · `MVP`

```
POST /scan
Content-Type: multipart/form-data
```

> 사진 1장을 올리면 AI 태깅 + 누끼(배경 제거) 결과를 돌려준다. **아직 저장 안 함(임시).** '재스캔' = 이 API 재호출. 프론트가 이 응답을 들고 있다가 **3-2**로 등록한다.

**Request** — form-data

| 파트 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `image` | file | 예 | jpg/png (iPhone HEIC는 변환 후 업로드) |

**Response `200 OK`**
```json
{
  "originalUrl": "https://cdn.mcmmuse.app/scan/ab12.jpg",
  "cutoutUrl":   "https://cdn.mcmmuse.app/scan/ab12-cutout.png",
  "tags": { "category": "상의", "color": "네이비", "material": "면", "mood": "클래식" }
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `originalUrl` | `string` | 업로드 원본 절대 URL |
| `cutoutUrl` | `string` | 누끼 이미지 절대 URL |
| `tags.category` | `Category` | AI 추정 (사용자가 3-2에서 수정 가능) |
| `tags.color` | `Color` | 〃 |
| `tags.material` | `Material` | 〃 |
| `tags.mood` | `ItemMood` | 〃 |

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 이미지 없음/형식 오류 | `400` | 이미지를 확인해 주세요 |
| AI 처리 실패 | `409` | 분석에 실패했어요. 다시 시도해 주세요 |

---

### 3-2. 옷장에 등록 (스캔 결과)

> `화면 3 스캔결과` · `인증 필요` · `MVP`

```
POST /closet-items
```

> 스캔 응답 + 사용자가 수정한 태그로 내 옷장에 저장한다. `source=OWN`(내 옷) 또는 `MCM`(보유 중인 MCM). 태그는 AI 값을 **수정해서 보낼 수 있다.**

**Request Body**
```json
{
  "source": "OWN",
  "category": "상의",
  "color": "네이비",
  "material": "면",
  "mood": "클래식",
  "imageUrl": "https://cdn.mcmmuse.app/scan/ab12.jpg",
  "cutoutUrl": "https://cdn.mcmmuse.app/scan/ab12-cutout.png"
}
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `source` | `Source` | 예 | `OWN` \| `MCM` |
| `category` | `Category` | 예 | — |
| `color` | `Color` | 예 | — |
| `material` | `Material` | 예 | — |
| `mood` | `ItemMood` | 예 | — |
| `imageUrl` | `string` | 예 | 3-1의 `originalUrl` |
| `cutoutUrl` | `string` | 아니오 | 3-1의 `cutoutUrl` |

**Response `201 Created`** — `ClosetItem`
```json
{
  "id": 12, "userId": 1,
  "category": "상의", "color": "네이비", "material": "면", "mood": "클래식",
  "imageUrl": "https://cdn.mcmmuse.app/closet/12.jpg",
  "cutoutUrl": "https://cdn.mcmmuse.app/closet/12-cutout.png",
  "source": "OWN", "mcmProductId": null,
  "createdAt": "2026-08-13T10:00:00Z"
}
```

---

### 3-3. 옷장에 담기 (카탈로그 MCM)

> `화면 6 제품상세` · `인증 필요` · `MVP`

```
POST /closet-items
```

> 앱에서 검색한 MCM 카탈로그 제품을 옷장에 담는다(`source=MCM`). 태그·이미지는 제품 정보에서 복사.
> **3-2와 같은 경로지만 body가 다르다** — `mcmProductId`만 오면 이 경로로 분기.

**Request Body**
```json
{ "mcmProductId": 103 }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `mcmProductId` | `number` | 예 | 존재하는 McmProduct id |

**Response `201 Created`** — `ClosetItem` (`source=MCM`, `mcmProductId=103`)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 제품 없음 | `404` | 제품을 찾을 수 없습니다 |

---

### 3-4. 옷장 목록 조회

> `화면 9 나의옷장` · `화면 4-a 스타일DNA(옷 선택)` · `인증 필요` · `MVP`

```
GET /closet-items?source=OWN|MCM
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `source` | `Source` | 아니오 | 안 주면 전체(OWN+MCM) |

**Response `200 OK`** — `ClosetItem[]` (3-2 응답 객체의 배열)

> **정렬: `createdAt DESC`** (최근 등록순). 화면 9 나의옷장 그리드가 이 순서를 그대로 사용한다.
> 카테고리 필터는 프론트가 클라이언트에서 처리 (v1 서버 파라미터 없음).

---

### 3-5. 옷장 아이템 삭제

> `화면 9 나의옷장` · `인증 필요` · `MVP`

```
DELETE /closet-items/{id}
```

**Response `204 No Content`**

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 내 아이템 아님 | `403` | 권한이 없습니다 |
| 없음 | `404` | 아이템을 찾을 수 없습니다 |

---

# 4. 스타일링

> 스타일 DNA·MCM 추천(4-1, 4-2)과 무드·코디·룩 큐레이터(4-3 ~ 4-7)를 한 그룹으로 묶는다. 둘 다 "옷장을 읽어 MCM을 제안한다"는 같은 일을 한다.

### 4-1. 스타일 DNA

> `화면 4-a 스타일DNA` · `인증 필요` · `MVP`

```
POST /style-dna
```

> 선택한 옷장 아이템들로 취향 요약을 만든다. **미저장(매번 생성).**

**Request Body**
```json
{ "closetItemIds": [1, 5, 12] }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `closetItemIds` | `number[]` | 예 | 1개 이상, 내 옷장 아이템 |

**Response `200 OK`**
```json
{
  "summary": "네이비·클래식 중심의 미니멀 무드",
  "dominantColors": ["네이비", "그레이"],
  "dominantMoods": ["클래식", "미니멀"],
  "keywords": ["차분한", "포멀", "베이직"]
}
```

> 화면 4-a는 DNA와 PERFECT MATCH 1픽을 함께 보여주므로 **4-1과 4-2를 연달아 호출**한다. 둘 다 LLM을 타므로 로딩 상태를 한 번만 걸고 병렬 호출해도 된다.

---

### 4-2. MCM 제품 추천

> `화면 4-a 스타일DNA` · `화면 5 MCM제품추천` · `인증 필요` · `MVP`

```
POST /recommendations
```

> 옷장 기반 MCM 추천. 베스트 1픽 + 더보기. **미저장(transient).** 응답 id는 백엔드가 DB로 재검증한 실재 제품만.

**Request Body**
```json
{ "closetItemIds": [1, 5, 12] }
```

**Response `200 OK`**
```json
{
  "bestPick": {
    "mcmProductId": 103,
    "reason": "네이비 룩에 포인트가 되는 카멜 비세토스",
    "pairsWithItemIds": [1, 5],
    "isExpansion": false,
    "product": { "id": 103, "name": "Tracy 숄더백", "imageUrl": "https://...", "price": 890000, "productUrl": "https://..." }
  },
  "more": [
    { "mcmProductId": 210, "reason": "...", "pairsWithItemIds": [12], "isExpansion": false,
      "product": { "id": 210, "name": "...", "imageUrl": "https://...", "price": 450000, "productUrl": "https://..." } }
  ]
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `bestPick` | `Recommendation` | 최우선 1픽 (화면 4-a의 PERFECT MATCH) |
| `more` | `Recommendation[]` | 더보기 목록 (화면 5) |
| `Recommendation.pairsWithItemIds` | `number[]` | 함께 매치되는 내 옷장 아이템 id |
| `Recommendation.isExpansion` | `boolean` | 취향 확장 추천 여부 (v1 항상 false) |
| `Recommendation.product` | `McmProduct` | 재검증된 제품 요약 |

---

### 4-3. 무드 목록

> `화면 15 무드선택` · `인증 필요` · `MVP`

```
GET /moods
```

> 고정 시드 6개.

**Response `200 OK`** — `Mood[]`
```json
[
  { "id": 1, "label": "저녁 약속", "labelEn": "DINNER DATE", "iconKey": "dinner" },
  { "id": 2, "label": "출근",     "labelEn": "OFFICE",      "iconKey": "office" },
  { "id": 3, "label": "출장",     "labelEn": "BUSINESS TRIP","iconKey": "trip" },
  { "id": 4, "label": "데일리",   "labelEn": "CASUAL",      "iconKey": "daily" },
  { "id": 5, "label": "주말 산책", "labelEn": "WEEKEND WALK", "iconKey": "walk" },
  { "id": 6, "label": "파티",     "labelEn": "EVENT",       "iconKey": "party" }
]
```

> ⚠️ 시드 6개는 Figma 무드 카드와 정렬 예정 — 확정 후 갱신.

---

### 4-4. 코디 후보 생성

> `화면 16 코디조합추천` · `화면 6 제품상세(큐레이팅)` · `인증 필요` · `MVP`

```
POST /outfits
```

> 무드 기준 코디 후보 **최대 3개**. 각 코디는 **MCM 제품을 정확히 1개 포함**한다(원칙 — MCM 없는 코디는 이 서비스의 코디가 아님). `seedMcmProductId`를 주면 그 제품을 고정(제품상세 큐레이팅). **미저장(transient).** 프론트가 `closetItems`·`mcmProduct`의 `cutoutUrl`로 콜라주 렌더.
>
> **재료가 부족하면 3개 미만이 올 수 있다**(1~3개). 프론트는 받은 개수만큼 렌더하면 되고 별도 분기는 불필요. 단, 옷장에 MCM이 하나도 없으면 아래 `409`.

**Request Body**
```json
{ "moodId": 1, "seedMcmProductId": 103 }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `moodId` | `number` | 예 | 1~6 |
| `seedMcmProductId` | `number` | 아니오 | 고정할 MCM 제품 id |

**Response `200 OK`** — `Outfit[]` (최대 3)
```json
[
  {
    "moodId": 1,
    "occasionLabel": "저녁 약속 / DINNER DATE",
    "closetItems": [
      { "id": 1, "cutoutUrl": "https://...", "category": "상의" },
      { "id": 5, "cutoutUrl": "https://...", "category": "하의" }
    ],
    "mcmProduct": { "id": 103, "imageUrl": "https://...", "cutoutUrl": "https://...", "name": "Tracy 숄더백" },
    "reason": "클래식 블라우스 + 비세토스로 포인트"
  }
]
```

> 콜라주는 `closetItems[].cutoutUrl` 과 `mcmProduct.cutoutUrl` 을 같이 쓴다. 둘 다 누끼라 배경이 일관된다.

| 에러 | HTTP | code | 메시지 |
|------|------|------|--------|
| 옷장에 MCM 없음 | `409` | `NO_MCM_IN_CLOSET` | 코디를 만들려면 옷장에 MCM 제품이 있어야 합니다. |
| 무드 없음 | `404` | — | 무드를 찾을 수 없습니다 |

---

### 4-5. 룩 저장 (택1)

> `화면 16 코디조합추천` · `인증 필요` · `MVP`

```
POST /looks
```

> 코디 후보 중 하나를 골라 저장한다. **여기서만** 코디 이미지(B, Nano Banana) 1장 생성. 영속.

**Request Body**
```json
{
  "moodId": 1,
  "closetItemIds": [1, 5],
  "mcmProductId": 103,
  "reason": "클래식 블라우스 + 비세토스로 포인트",
  "wornDate": "2026-08-15"
}
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `moodId` | `number` | 예 | 1~6 |
| `closetItemIds` | `number[]` | 예 | 내 옷장 아이템 |
| `mcmProductId` | `number` | 예 | 코디에 포함된 MCM |
| `reason` | `string` | 아니오 | 코디 이유 |
| `wornDate` | `string` | 아니오 | `yyyy-MM-dd`. **안 주면 서버가 오늘 날짜로 채움** (화면 16에는 날짜 선택 UI가 없음) |

**Response `201 Created`** — `Look`
```json
{
  "id": 7, "userId": 1, "wornDate": "2026-08-15",
  "moodId": 1, "occasionLabel": "저녁 약속 / DINNER DATE",
  "closetItemIds": [1, 5], "mcmProductId": 103,
  "reason": "클래식 블라우스 + 비세토스로 포인트",
  "generatedImageUrl": null
}
```

> **이미지 생성은 비동기다.** `POST /looks`는 룩을 저장하고 **즉시 `201`을 반환**하며, 이 시점의 `generatedImageUrl`은 보통 `null`이다. 이미지는 백그라운드에서 생성된다.
>
> **프론트 처리:** `201`을 받으면 저장 완료 화면을 먼저 띄우고, `generatedImageUrl`이 `null`이면 **`GET /looks/{id}`를 폴링**(2~3초 간격 권장)해서 값이 채워지면 이미지를 교체한다. 생성이 최종 실패해도 `null`로 남으므로 폴링은 적당한 횟수에서 중단하고 플레이스홀더를 유지한다.

---

### 4-6. 룩 단건 조회

> `화면 16 코디조합추천 (이미지 폴링)` · `인증 필요` · `MVP`

```
GET /looks/{id}
```

> 저장된 룩 하나를 조회한다. **이미지 생성 완료 여부 폴링에 사용.**

**Response `200 OK`** — `Look` (4-5 응답과 동일 구조)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 내 룩 아님 | `403` | 권한이 없습니다 |
| 없음 | `404` | 룩을 찾을 수 없습니다 |

---

### 4-7. 룩 목록 (코디 기록)

> `화면 10 코디기록(캘린더)` · `인증 필요` · `후순위` — 화면은 토요일 이후

```
GET /looks?month=2026-08
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `month` | `string` | 아니오 | `yyyy-MM` 필터. 안 주면 전체 |

**Response `200 OK`** — `Look[]` (4-5 객체의 배열)

---

# 엔드포인트 요약

| 그룹 | 메서드 | 경로 | 인증 | 응답 | 화면 | 설명 |
|----|--------|------|------|------|------|------|
| 인증 | `POST` | `/auth/register` | 불필요 | 201 | 4-b | 회원가입(+refresh 쿠키) |
| 인증 | `POST` | `/auth/login` | 불필요 | 200 | 4-b | 로그인(+refresh 쿠키) |
| 인증 | `POST` | `/auth/refresh` | 쿠키 | 200 | 전역 | Access 갱신 |
| 인증 | `POST` | `/auth/logout` | 쿠키 | **204** | 17 | 로그아웃 |
| 인증 | `GET` | `/me` | 필요 | 200 | 17 | 내 정보(profile 탭) |
| 카탈로그 | `GET` | `/mcm-products` | 필요 | 200 | home | 제품 검색/목록 |
| 카탈로그 | `GET` | `/mcm-products/{id}` | 필요 | 200 | 6 | 제품 상세 |
| 옷장 | `POST` | `/scan` | 필요 | 200 | 2 | 옷 스캔(태깅+누끼), 미저장 |
| 옷장 | `POST` | `/closet-items` | 필요 | 201 | 3 · 6 | 옷장 등록(스캔결과 or 카탈로그 MCM) |
| 옷장 | `GET` | `/closet-items` | 필요 | 200 | 9 · 4-a | 옷장 목록(`source` 필터) |
| 옷장 | `DELETE` | `/closet-items/{id}` | 필요 | **204** | 9 | 옷장 아이템 삭제 |
| 스타일링 | `POST` | `/style-dna` | 필요 | 200 | 4-a | 스타일 DNA(미저장) |
| 스타일링 | `POST` | `/recommendations` | 필요 | 200 | 4-a · 5 | MCM 추천(미저장) |
| 스타일링 | `GET` | `/moods` | 필요 | 200 | 15 | 무드 6개 |
| 스타일링 | `POST` | `/outfits` | 필요 | 200 | 16 · 6 | 코디 후보 최대 3(미저장) |
| 스타일링 | `POST` | `/looks` | 필요 | 201 | 16 | 룩 저장(+B 이미지 **비동기** 생성) |
| 스타일링 | `GET` | `/looks/{id}` | 필요 | 200 | 16 | 룩 단건(이미지 생성 폴링) |
| 스타일링 | `GET` | `/looks` | 필요 | 200 | 10 | 룩 목록(`month` 필터) — 후순위 |

---

# 화면 → 호출 API (역방향 인덱스)

프론트가 화면부터 시작할 때 보는 표. **화면 하나가 여러 API를 쓰고, API 하나가 여러 화면에서 쓰인다(N:M).**

| 화면 | 호출 API | 비고 |
|------|---------|------|
| 0 스플래시 | — | 토큰 유효성만 확인(`GET /me` 또는 `POST /auth/refresh`) |
| 1 온보딩 | — | 정적 화면 |
| 4-b 로그인/회원가입 | `POST /auth/login` · `POST /auth/register` | |
| **home 상품보기** | `GET /mcm-products` | 진입 시 1회, 검색·탭은 클라 필터 |
| 6 제품상세 | `GET /mcm-products/{id}` · `POST /closet-items` · `POST /outfits` | 담기 / 이 제품으로 큐레이팅 / **구매하기는 `productUrl` 외부 이동** |
| 9 나의옷장 (closet 탭) | `GET /closet-items` · `DELETE /closet-items/{id}` | ＋버튼 → 화면 2 또는 갤러리 |
| 2 옷장스캔 | `POST /scan` | 카메라 단일 촬영 |
| 3 스캔결과 | `POST /closet-items` | 태그 수정 후 등록. '재스캔'은 `POST /scan` 재호출 |
| 4-a 스타일DNA | `GET /closet-items` · `POST /style-dna` · `POST /recommendations` | 옷 선택 → 진단 → DNA + PERFECT MATCH 1픽 |
| 5 MCM제품추천 | (4-a의 `POST /recommendations` 응답 `more` 재사용) | 추가 호출 불필요 |
| 15 무드선택 | `GET /moods` | |
| 16 코디조합추천 | `POST /outfits` · `POST /looks` · `GET /looks/{id}` | 3후보 → 택1 저장 → 이미지 폴링 |
| 17 프로필 (profile 탭) | `GET /me` · `POST /auth/logout` | |
| 10 코디기록(캘린더) | `GET /looks?month=` | **후순위** — 토요일 이후 |

---

# 미확정 / 추후 (P1+)

- 캘린더 화면(10)·AR(7)·예약및결제(8)·아카이브(11~13): API 후순위.
- **♡찜(wishlist): v1 제외.** 화면에 아이콘을 노출하지 않는다. 도입 시 `POST/DELETE /wishlist/{productId}` + `GET /wishlist`.
- **코디 내 MCM 복수 포함**: v1은 정확히 1개. 배열화(`mcmProducts[]`)는 `/looks`의 `mcmProductId`까지 연쇄 변경이라 v2 검토.
- **결제**: v1은 `productUrl` 딥링크로 MCM 공식몰 이동(우리가 결제 처리하지 않음). 향후 제휴 시 커머스 API 위임 검토.
- **무드 시드 6개**: Figma 무드 카드와 정렬 확정 대기.
- 취향 확장 추천(`isExpansion=true`) 로직: v1은 항상 false.
- 데이터 증가 시 페이지네이션 도입 검토(v1 없음).
- Access/Refresh 유효기간·쿠키 속성(SameSite/Secure/Path)은 구현 시 확정.
