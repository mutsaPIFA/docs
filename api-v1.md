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

> ⚑ 스타일 DNA는 **화면상 옷장에서 옷을 골라 시작**하지만 소속은 스타일링이다 — 옷장은 입력(재료)일 뿐이고, 하는 일(취향 분석·추천)과 출력(DNA·추천 결과)이 스타일링 도메인이기 때문. 그룹 기준은 "어느 화면에서 시작하나"가 아니라 "무슨 일을 하나"다.

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
type Color    = '블랙' | '화이트' | '네이비' | '그레이' | '베이지' | '브라운' | '카멜' | '그린' | '핑크' | '블루' | '스카이블루' | '레드' | '옐로우' | '카키' | '퍼플' | '오렌지' | '기타'
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

### 1-6. 게스트 발급

> `QR 진입` · `인증 불필요` · `시연`

```
POST /auth/guest
```

요청 본문 없음. 호출 시 서버가 게스트 계정을 생성하고 즉시 로그인 토큰을 발급한다 — 웹 루트(`/`)에 토큰 없이 진입하면 프론트가 자동 호출한다(QR 방문자는 입력 없이 옷장 진입). 일반 로그인은 `/login` 직접 접근으로 유지.

- 게스트 이메일은 `guest-{랜덤}@guest.mcmmuse.local`로 통일 — 시연 후 도메인 기준 일괄 정리 가능
- 비밀번호는 내부 생성이며 노출하지 않는다 (게스트 재접속은 발급된 토큰/refresh로만 유지)
- 옷장 시드: 서버 설정 `app.demo.guest-template-email`에 지정된 계정의 옷장을 복사해 시작. **기본은 비움(빈 옷장)** — 부스 실물 스캔 체험 유도 (팀 확정 2026-08-20)

**Response `201 Created`** — 회원가입(§1-1)과 동일한 `RegisterResponse` + refresh 쿠키
```json
{ "userId": 42, "accessToken": "...", "tokenType": "Bearer" }
```

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

> 로그인한 사용자 정보. profile 탭에서 닉네임·아바타·스타일 DNA 카드 표시에 사용.

**Response `200 OK`**
```json
{
  "userId": 1,
  "email": "muse@example.com",
  "nickname": "뮤즈",
  "avatarUrl": "https://cdn.mcmmuse.app/avatars/ab12.png",
  "styleDna": {
    "summary": "네이비·클래식 중심의 미니멀 무드",
    "dominantColors": ["네이비", "그레이"],
    "dominantMoods": ["클래식", "미니멀"],
    "keywords": ["차분한", "포멀", "베이직"],
    "updatedAt": "2026-08-18T11:20:00Z"
  }
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `userId` | `number` | 사용자 id |
| `email` | `string` | 가입 이메일 |
| `nickname` | `string` | 표시 이름 |
| `avatarUrl` | `string?` | 프로필 이미지 절대 URL. 미설정 시 `null` — 프론트는 기본 마스코트 표시 |
| `styleDna` | `object?` | **가장 최근 스타일 DNA 분석 결과** (4-1 응답 + `updatedAt`). 분석 이력이 없으면 `null` — 프로필 카드 미표시 |

> `styleDna`는 `POST /style-dna`(4-1)가 성공할 때마다 서버가 자동으로 최신 값으로 갱신한다. 별도 저장 호출 없음.

---

# 2. MCM 제품 카탈로그

### 2-1. 제품 검색 / 목록

> `home 상품보기` · `인증 필요` · `MVP`

```
GET /mcm-products?query=&category=&page=&size=
```

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `query` | `string` | 아니오 | 이름 검색 |
| `category` | `Category` | 아니오 | 카테고리 필터 |
| `page` | `int` | 아니오 | **1부터.** 주면 페이지 봉투 응답, 없으면 전체 배열(하위호환) |
| `size` | `int` | 아니오 | 페이지 크기 — 기본 8, 1~50으로 보정 |

> ⚑ 샵 목록은 `page`/`size`로 8개씩 페이지 조회한다(2026-08-20 확정 — 카탈로그 589건이라 전체 수신은 과함). 정렬은 서버가 소유: 필터 없는 전체 목록은 시리즈 뭉침 방지용 **고정 해시 셔플**(결정적 — 페이지 넘겨도 순서 일관), 검색·카테고리 필터 시 id 오름차순. 범위를 벗어난 `page`는 마지막 페이지로 보정. 단, CLOTHES 탭(복수 카테고리 합성)과 세부 태그 필터는 클라이언트 필터이므로 그 경우 프론트는 `page` 없이 전체를 받아 클라에서 페이징한다.

**Response `200 OK` (page 지정 시)** — 페이지 봉투
```json
{ "items": [], "page": 1, "size": 8, "totalItems": 589, "totalPages": 74 }
```
`items`는 아래 `McmProduct[]`와 동일 형태.

**Response `200 OK` (page 미지정)** — `McmProduct[]`
```json
[
  { "id": 103, "name": "Tracy 숄더백", "category": "가방", "color": "카멜", "material": "가죽",
    "price": 890000, "imageUrl": "https://...", "cutoutUrl": "https://...",
    "productUrl": "https://...",
    "description": "우아한 로고 락 클로저가 돋보이는 Tracy(트레이시) 숄더백은 ...",
    "size": "S",
    "imageUrls": ["https://...", "https://...", "https://..."] }
]
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `imageUrl` | `string` | 상품 대표 이미지 절대 URL (목록 카드용 — `imageUrls`의 첫 장과 동일) |
| `cutoutUrl` | `string?` | **누끼 이미지** 절대 URL. 화면 16 콜라주 렌더에 사용. 적재 시 생성하며 실패 시 `null` |
| `productUrl` | `string` | MCM 공식몰 상품 페이지. 화면 6 "구매하기"가 이 URL로 이동 |
| `description` | `string?` | 상세 설명 문단 — 화면 7-a 본문 |
| `size` | `string?` | 사이즈 표기 — **`\|` 구분 목록일 수 있음** (예: "미니", "S", 신발은 "35 IT / 여성 \| 36 IT / 여성 \| ...") |
| `imageUrls` | `string[]` | **상세 캐러셀 이미지 전체**(상품당 5~8장) — 화면 7-a 상단 캐러셀. 빈 배열 가능 |

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
>
> ⏱ **AI 처리로 10~40초 걸린다** — 로딩 상태 필수, HTTP 클라이언트 타임아웃은 60초 이상 권장.

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

> 스캔 응답 + 사용자가 수정한 태그로 내 옷장에 저장한다. `source=OWN`(내 옷) 또는 `MCM`(보유 중인 MCM). 태그는 AI 값을 **수정해서 보낼 수 있고**, `name`으로 **명칭도 처음부터 정할 수 있다**(§3-6과 같은 의미론 — 생략·빈 값이면 태그 조합 표시).

**Request Body**
```json
{
  "name": "출근용 네이비 셔츠",
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
| `name` | `string` | 아니오 | 사용자 지정 명칭 — 생략·빈 값이면 `null`(태그 조합 표시) |
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
  "name": null,
  "category": "상의", "color": "네이비", "material": "면", "mood": "클래식",
  "imageUrl": "https://cdn.mcmmuse.app/closet/12.jpg",
  "cutoutUrl": "https://cdn.mcmmuse.app/closet/12-cutout.png",
  "source": "OWN", "mcmProductId": null,
  "createdAt": "2026-08-13T10:00:00Z"
}
```

> `name`(`string?`) — **사용자 지정 명칭**(3-6에서 수정). `null`이면 프론트가 태그 조합("네이비 면 상의")으로 표시명을 만든다.

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

### 3-6. 옷장 아이템 수정 (명칭·태그)

> `화면 9 나의옷장(아이템 정보)` · `인증 필요` · `MVP`

```
PATCH /closet-items/{id}
```

> 보낸 필드만 수정한다(부분 수정). 사진 교체는 지원하지 않는다 — 재스캔으로.

**Request Body** (모든 필드 선택)
```json
{ "name": "출근용 셔츠", "category": "상의", "color": "네이비", "material": "면", "mood": "클래식" }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `name` | `string?` | 아니오 | 최대 30자. 빈 문자열 `""` 전송 시 명칭 제거(태그 조합 표시로 복귀), 필드 생략 시 유지 |
| `category` | `Category` | 아니오 | 어휘 내 값 |
| `color` | `Color` | 아니오 | 어휘 내 값 |
| `material` | `Material` | 아니오 | 어휘 내 값 |
| `mood` | `ItemMood` | 아니오 | 어휘 내 값 |

**Response `200 OK`** — `ClosetItem` (3-2 응답과 동일 구조)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 검증 실패 | `400` | 명칭은 30자 이하여야 합니다 |
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

> 선택한 옷장 아이템들로 취향 요약을 만든다. 응답은 매번 새로 생성하되, **성공 시 최신 결과가 사용자 프로필에 저장된다**(1-5 `styleDna` — 프로필 카드용).

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

> 무드 기준 코디 후보 **최대 3개**. 각 코디는 **MCM 제품을 정확히 1개 포함**한다(원칙 — MCM 없는 코디는 이 서비스의 코디가 아님). `seedMcmProductId`를 주면 그 제품을 고정(제품상세 큐레이팅). **미저장(transient).** 후보마다 **AI가 flat-lay 화보 1장을 생성해 `imageUrl`로 담아 준다.**
>
> ⏱ **화보 생성 때문에 응답이 20~40초 걸린다**(후보 화보 병렬 생성). "코디를 생성하고 있어요" 로딩 연출 필수, HTTP 타임아웃 90초 권장.
>
> **후보 수는 옷장 재고에 비례한다** — 상·하의 짝 재고(`min(상의+아우터 수, 하의 수)`, 원피스는 양쪽 겸용)가 다양성의 상한이므로 서버가 이렇게 정한다:
> 기본 **1개** · 짝 재고 3 이상 → **최대 2개** · 짝 재고 5 이상 + MCM 재료 2종 이상 → **최대 3개**.
> 추가로 **어떤 두 후보도 같은 아이템·제품을 2개 이상 공유하지 않는다**(중복 제약 — 서버가 코드로 강제). 화보 생성 실패분 제외까지 겹치면 그만큼 줄어든 개수가 온다(1~3개) — 프론트는 받은 개수만큼 렌더하면 되고 별도 분기는 불필요. 단, 옷장에 MCM이 하나도 없으면 아래 `409`.
> 프론트는 생성 진입 전에 **상의·하의·MCM 보유를 사전 안내**한다(화면 15 — "옷장에 상의가 있어야 해요" 등 단계별 문구).

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
    "concept": "Soft Classic",
    "imageUrl": "https://cdn.mcmmuse.app/outfits/3f2a.jpg",
    "closetItems": [
      { "id": 1, "cutoutUrl": "https://...", "category": "상의", "color": "화이트", "material": "면" },
      { "id": 5, "cutoutUrl": "https://...", "category": "하의", "color": "블랙", "material": "울" }
    ],
    "mcmProduct": { "id": 103, "imageUrl": "https://...", "cutoutUrl": "https://...", "name": "Tracy 숄더백" },
    "reason": "클래식 블라우스 + 비세토스로 포인트"
  }
]
```

> `concept` — 후보의 코디 컨셉명(**영어 2~3단어**, AI 작명 — 예: Refined Minimal). 후보 카드 제목으로 사용. AI 실패로 룰베이스 폴백이면 `null` — 미표시.
>
> `imageUrl` — 후보 조합을 AI가 한 장의 flat-lay 화보로 연출한 이미지. **재생성 이미지라 디테일이 원본과 다를 수 있다**(연출컷 — 원본 확인은 `closetItems`·`mcmProduct` 데이터와 상세 화면이 담당).
>
> **생성에 실패한 후보는 응답에서 제외된다** — 반환된 후보에는 `imageUrl`이 항상 있고, 그만큼 후보 수가 줄 수 있다(LOOK 번호는 배열 순서 그대로 매기면 됨). **전 후보 생성 실패**(사실상 AI 서비스 장애)면 아래 `503` — 프론트는 "다시 시도" 화면을 보여준다.
>
> ※ 후보 화보 생성은 **팀 시연 평가 후 유지/폐기를 결정하는 실험 기능**이다. 폐기되면 재료(`closetItems[].cutoutUrl`·`mcmProduct.cutoutUrl` 누끼) 콜라주 렌더로 전환한다 — 재료 필드를 유지하는 이유. `category`는 그 경우의 배치 힌트.
>
> `closetItems[].color`·`material` — 표시용 태그(§3-1과 동일 어휘). 프론트가 "화이트 면 상의"처럼 아이템 표시명을 조합하는 데 쓴다.

| 에러 | HTTP | code | 메시지 |
|------|------|------|--------|
| 옷장에 MCM 없음 | `409` | `NO_MCM_IN_CLOSET` | 코디를 만들려면 옷장에 MCM 제품이 있어야 합니다. |
| 전 후보 화보 생성 실패 | `503` | `OUTFIT_GENERATION_FAILED` | 코디 생성에 실패했어요. 다시 시도해 주세요. |
| 무드 없음 | `404` | — | 무드를 찾을 수 없습니다 |

---

### 4-5. 룩 저장 (택1)

> `화면 16 코디조합추천` · `인증 필요` · `MVP`

```
POST /looks
```

> 코디 후보 중 하나를 골라 저장한다. 영속. **고른 후보의 `imageUrl`을 body에 같이 보내면 그 화보가 룩 이미지로 저장된다** — 재생성 없음, 고른 것과 저장된 것이 동일.

**Request Body**
```json
{
  "moodId": 1,
  "closetItemIds": [1, 5],
  "mcmProductId": 103,
  "imageUrl": "https://cdn.mcmmuse.app/outfits/3f2a.jpg",
  "concept": "Soft Classic",
  "note": "오늘 옷 센스 있다는 말 들어서 기분 좋았던 날, 꾸안꾸 룩",
  "reason": "클래식 블라우스 + 비세토스로 포인트",
  "wornDate": "2026-08-15"
}
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `moodId` | `number` | 예 | 1~6 |
| `closetItemIds` | `number[]` | 예 | 내 옷장 아이템 |
| `mcmProductId` | `number` | 예 | 코디에 포함된 MCM |
| `imageUrl` | `string` | 아니오 | **4-4 후보의 `imageUrl` 그대로** (§4-4 정책상 반환된 후보에는 항상 존재) |
| `concept` | `string` | 아니오 | **4-4 후보의 `concept` 그대로** (60자 이하) — 기록 화면 제목용으로 룩에 영구 저장 |
| `note` | `string` | 아니오 | **사용자 소감** ("이 코디 어땠어요?" — 화면 11) 1000자 이하. AI의 `reason`과 별개 |
| `reason` | `string` | 아니오 | 코디 이유 (AI 추천 이유) |
| `wornDate` | `string` | 아니오 | `yyyy-MM-dd`. **안 주면 오늘.** 화면 9(무드 선택)의 날짜 선택 값을 그대로 전달 — 과거·미래 날짜 기록 가능, **같은 날짜에 여러 룩 허용** |

**Response `201 Created`** — `Look`
```json
{
  "id": 7, "userId": 1, "wornDate": "2026-08-15",
  "moodId": 1, "occasionLabel": "저녁 약속 / DINNER DATE",
  "concept": "Soft Classic",
  "closetItemIds": [1, 5], "mcmProductId": 103,
  "note": "오늘 옷 센스 있다는 말 들어서 기분 좋았던 날, 꾸안꾸 룩",
  "reason": "클래식 블라우스 + 비세토스로 포인트",
  "generatedImageUrl": "https://cdn.mcmmuse.app/outfits/3f2a.jpg"
}
```

> `generatedImageUrl`은 body의 `imageUrl`이 **저장 즉시 그대로 채워진다** — 별도 비동기 생성·폴링 없음. §4-4에서 생성 실패 후보는 응답에서 제외되므로, 후보에서 골라 저장한 룩에는 화보가 항상 있다.

---

### 4-6. 룩 단건 조회

> `화면 16 코디조합추천` · `인증 필요` · `MVP`

```
GET /looks/{id}
```

> 저장된 룩 하나를 조회한다. (`generatedImageUrl`은 저장 시점에 확정된다 — 폴링 불필요.)
>
> 룩이 참조하는 옷장 아이템은 **이후 옷장에서 삭제됐더라도 룩 조회에는 계속 나온다**(과거 코디 기록 보존). 삭제는 옷장 목록에서만 사라지게 한다.

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

> **정렬: `wornDate DESC`, 같은 날짜 안에서는 최근 저장순.** 하루에 룩 여러 개 기록 가능 — 캘린더(화면 10-b)에서 날짜를 탭하면 그날의 룩 목록을 이 응답에서 필터해 보여주면 된다.

---

### 4-8. 룩 삭제 (기록 취소)

> `화면 16 코디조합추천` · `인증 필요` · `MVP`

```
DELETE /looks/{id}
```

> 저장한 룩을 취소한다(완전 삭제). 취소 후 같은 후보를 다시 기록할 수 있다. 후보 카드의 "기록됨" 상태 해제와 **기록 상세의 삭제 버튼** 양쪽에서 쓴다.

**Response `204 No Content`**

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 내 룩 아님 | `403` | 권한이 없습니다 |
| 없음 | `404` | 룩을 찾을 수 없습니다 |

---

### 4-9. 룩 수정 (소감·날짜)

> `화면 10 코디기록 · 16 코디조합추천` · `인증 필요` · `MVP`

```
PATCH /looks/{id}
```

> 보낸 필드만 수정한다(부분 수정). `concept`·`reason`·이미지·아이템 구성은 수정 불가(코디 자체를 바꾸려면 삭제 후 재기록).

**Request Body** (모든 필드 선택)
```json
{ "note": "생각보다 반응이 좋았던 날", "wornDate": "2026-08-20" }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `note` | `string?` | 아니오 | 최대 1,000자 (4-5와 동일). 빈 문자열 `""` 전송 시 소감 제거, 필드 생략 시 유지 |
| `wornDate` | `string` | 아니오 | `yyyy-MM-dd` — 기록 날짜 이동 |

**Response `200 OK`** — `Look` (4-5 응답과 동일 구조)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 검증 실패 | `400` | 소감은 1,000자 이하여야 합니다 |
| 내 룩 아님 | `403` | 권한이 없습니다 |
| 없음 | `404` | 룩을 찾을 수 없습니다 |

---

# 5. 프로필·찜

> 화면 17(profile 탭)과 home 하트를 실기능으로 만드는 그룹. 팀 확정 범위: 스타일 DNA 프로필 저장(1-5·4-1에 반영) + 닉네임 수정 + 프로필 이미지 + 찜.

### 5-1. 닉네임 수정

> `화면 17 프로필` · `인증 필요` · `MVP`

```
PATCH /me
```

**Request Body**
```json
{ "nickname": "뮤즈" }
```

| 필드 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `nickname` | `string` | 예 | 1~20자, 공백만 불가 |

**Response `200 OK`** — 1-5 응답과 동일 구조

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 검증 실패 | `400` | 닉네임은 1~20자여야 합니다 |

---

### 5-2. 프로필 이미지 업로드

> `화면 17 프로필` · `인증 필요` · `MVP`

```
POST /me/avatar
Content-Type: multipart/form-data
```

| 파트 | 타입 | 필수 | 제약 |
|------|------|------|------|
| `image` | `file` | 예 | jpg/png/webp, 최대 15MB (3-1과 동일) |

**Response `200 OK`**
```json
{ "avatarUrl": "https://cdn.mcmmuse.app/avatars/ab12.png" }
```

> 업로드 즉시 교체(이전 이미지는 참조만 끊는다). 이후 `GET /me`의 `avatarUrl`에 반영.

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 형식/용량 | `400` | 지원하지 않는 이미지입니다 |

---

### 5-3. 찜 목록

> `화면 home 상품보기 · 17 프로필` · `인증 필요` · `MVP`

```
GET /wishlist
```

**Response `200 OK`** — `McmProduct[]` (2-1 객체의 배열, **최근 찜 순**)

> 제품 상세(6)의 하트 상태는 프론트가 이 응답의 id 집합으로 매칭한다(2-1 응답은 무변경 — 비파괴). 찜 진입점은 제품 상세·찜 목록 화면 — 샵 카드에는 하트를 두지 않는다(팀 확정).

---

### 5-4. 찜 추가

> `화면 home 상품보기 · 6 제품상세` · `인증 필요` · `MVP`

```
POST /wishlist/{mcmProductId}
```

**Response `201 Created`** (본문 없음) — 이미 찜한 상태면 `200 OK` (멱등)

| 에러 | HTTP | 메시지 |
|------|------|--------|
| 제품 없음 | `404` | 제품을 찾을 수 없습니다 |

---

### 5-5. 찜 해제

> `화면 home 상품보기 · 6 제품상세 · 17 프로필` · `인증 필요` · `MVP`

```
DELETE /wishlist/{mcmProductId}
```

**Response `204 No Content`** — 찜하지 않은 상태여도 `204` (멱등)

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
| 옷장 | `PATCH` | `/closet-items/{id}` | 필요 | 200 | 9 | 아이템 수정(명칭·태그) |
| 스타일링 | `POST` | `/style-dna` | 필요 | 200 | 4-a | 스타일 DNA(미저장) |
| 스타일링 | `POST` | `/recommendations` | 필요 | 200 | 4-a · 5 | MCM 추천(미저장) |
| 스타일링 | `GET` | `/moods` | 필요 | 200 | 15 | 무드 6개 |
| 스타일링 | `POST` | `/outfits` | 필요 | 200 | 16 · 6 | 코디 후보 최대 3(미저장) |
| 스타일링 | `POST` | `/looks` | 필요 | 201 | 16 | 룩 저장(후보 화보 `imageUrl` 재사용) |
| 스타일링 | `GET` | `/looks/{id}` | 필요 | 200 | 16 | 룩 단건 |
| 스타일링 | `GET` | `/looks` | 필요 | 200 | 10 | 룩 목록(`month` 필터) — 후순위 |
| 스타일링 | `DELETE` | `/looks/{id}` | 필요 | **204** | 16 · 10 | 룩 삭제(기록 취소) |
| 스타일링 | `PATCH` | `/looks/{id}` | 필요 | 200 | 10 | 룩 수정(소감·날짜) |
| 프로필 | `PATCH` | `/me` | 필요 | 200 | 17 | 닉네임 수정 |
| 프로필 | `POST` | `/me/avatar` | 필요 | 200 | 17 | 프로필 이미지 업로드 |
| 찜 | `GET` | `/wishlist` | 필요 | 200 | home · 17 | 찜 목록(최근 찜 순) |
| 찜 | `POST` | `/wishlist/{mcmProductId}` | 필요 | 201 | home · 6 | 찜 추가(멱등) |
| 찜 | `DELETE` | `/wishlist/{mcmProductId}` | 필요 | **204** | home · 6 · 17 | 찜 해제(멱등) |

---

# 화면 → 호출 API (역방향 인덱스)

프론트가 화면부터 시작할 때 보는 표. **화면 하나가 여러 API를 쓰고, API 하나가 여러 화면에서 쓰인다(N:M).**

| 화면 | 호출 API | 비고 |
|------|---------|------|
| 0 스플래시 | — | 토큰 유효성만 확인(`GET /me` 또는 `POST /auth/refresh`) |
| 1 온보딩 | — | 정적 화면 |
| 4-b 로그인/회원가입 | `POST /auth/login` · `POST /auth/register` | |
| **home 상품보기** | `GET /mcm-products` · `GET /wishlist` · `POST·DELETE /wishlist/{id}` | 하트 상태는 찜 목록 id 집합으로 매칭 |
| 6 제품상세 | `GET /mcm-products/{id}` · `POST /closet-items` · `POST /outfits` | 담기 / 이 제품으로 큐레이팅 / **구매하기는 `productUrl` 외부 이동** |
| 9 나의옷장 (closet 탭) | `GET /closet-items` · `DELETE /closet-items/{id}` | ＋버튼 → 화면 2 또는 갤러리 |
| 2 옷장스캔 | `POST /scan` | 카메라 단일 촬영 |
| 3 스캔결과 | `POST /closet-items` | 태그 수정 후 등록. '재스캔'은 `POST /scan` 재호출 |
| 4-a 스타일DNA | `GET /closet-items` · `POST /style-dna` · `POST /recommendations` | 옷 선택 → 진단 → DNA + PERFECT MATCH 1픽 |
| 5 MCM제품추천 | (4-a의 `POST /recommendations` 응답 `more` 재사용) | 추가 호출 불필요 |
| 15 무드선택 | `GET /moods` | |
| 16 코디조합추천 | `POST /outfits` · `POST /looks` | 3후보(화보 생성, 20~40초 로딩) → 택1 저장(화보 재사용) |
| 17 프로필 (profile 탭) | `GET /me` · `PATCH /me` · `POST /me/avatar` · `GET /wishlist` · `POST /auth/logout` | `styleDna` 카드는 `GET /me` 응답으로 |
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
