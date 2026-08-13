# MCM MUSE — 비전/LLM/이미지 기술 리서치 (누끼·자동분류·코디·가상피팅)

- 작성일: 2026-08-13
- 목적: 코디 이미지/가상피팅/옷 자동분류에 **어떤 API·오픈소스**를 쓸지 결정
- 트리거: 참고 앱 "Dress AI" 분석 + 웹 리서치

---

## 1. 참고 앱 "Dress AI" 분석 (사용자 캡처)

| 기능 | 구현 | 비고 |
|---|---|---|
| AI 처리 전부 | **Google Gemini** | 설정에 명시 "AI 처리(Google Gemini)". 사용자 키 입력 옵션, Pro 모델 토글(고품질 피팅) |
| 옷 등록 | 촬영 → **자동 카테고리 분류**(상의/하의/모자/원피스…) | Gemini 비전 태깅 |
| AI 스타일리스트(코디) | **누끼 이미지 "+" 조합 + 이유/팁 텍스트** | ⚠️ 생성 이미지 아님. 누끼 콜라주 |
| 무드/상황 입력 | **프리텍스트**("다음 주 화요일 입사 면접") + 프리셋 칩 | 우리 고정 6무드보다 유연 → 참고 |
| 입어보기(가상 피팅) | 아바타(전신샷)에 착장 **생성**, ~20초 → "오늘의 핏" → 룩북 저장 | Gemini 이미지(Nano Banana) |

**핵심 통찰:** 이 앱은 **코디 카드=누끼 콜라주(비생성, 쌈)** 와 **입어보기=이미지 생성(비쌈)** 을 분리한다. 우리가 앞서 정한 방향과 정확히 동일.

---

## 2. 우리 기능 → 필요 기술 분해

| 기능 | 난이도 | 기술 |
|---|---|---|
| ① 옷 태깅·자동분류 | 🟢 쉬움 | **Gemini 비전 1콜**(사진→category/color/material/mood JSON). 대안: SegFormer-clothes |
| ② 누끼(배경 제거) | 🟢 쉬움 | **rembg**(로컬·무료). 코디 콜라주·피팅 입력의 전제 |
| ③ 코디 조합(데일리룩) | 🟢 쉬움 | LLM이 조합 선택 + **누끼 콜라주**(프론트 배치). ← "누끼따서 데일리룩"은 **생성 아님** |
| ④ 입어보기(가상 피팅) | 🔴 어려움 | **Gemini Nano Banana**(API) 또는 오픈소스 VTON(GPU) |

> 오해 방지: **"누끼 따서 데일리룩 만들기"는 쉬운 쪽(③)**. 진짜 어려운 건 **입어보기(④, 생성)** 다.

---

## 3. 옵션 비교

### A. LLM/이미지 API — Google Gemini (권장)
- **Gemini 2.5 Flash Image (Nano Banana)**: GA. **~$0.039/이미지**(1290 output tokens/장). 공식적으로 **가상 피팅용 지원**(캐릭터 정체성·객체 충실도 유지). 멀티이미지 입력(사람+옷) → 착장 합성.
- **후속(2026)**: **Nano Banana 2 Lite (Gemini 3.1 Flash Image)**, Gemini Omni Flash 출시 — 더 저렴/신형 가능. 착수 시 최신 모델명 확인.
- **비전 태깅**: Gemini 2.5 Flash(텍스트/비전)로 사진→JSON 1콜. 무료 티어(레이트 리밋) 존재, 이미지 생성은 유료 가능 → 확인 필요.
- **장점**: GPU·서버 불필요, API 한 방, 참고 앱과 동일 검증된 경로, 데모 비용 미미(수백 장 = 몇 달러).
- **단점**: 외부 종속·유료, 레이트 리밋.

### B. 오픈소스 가상 피팅 (HuggingFace) — 장기/비용 대안
| 모델 | 특징 | 요구 |
|---|---|---|
| **IDM-VTON** | "2026 최고 오픈소스 VTON"으로 가장 많이 인용, SDXL 기반, 품질↑ | GPU 무거움 |
| **CatVTON** | VITON-HD SOTA(2024.11), 1024×768 **~35초, <8GB VRAM** | GPU(가벼움) |
| **Leffa** | try-on + 포즈 전이 통합, 디테일 왜곡↓ | GPU |
- **공통 단점**: **GPU 필수 + 호스팅 + ~35초 지연**. 해커톤엔 무겁다. 직접 학습 불필요(사전학습 가중치 사용).

### C. 옷 분할/파싱 (자동분류 대안)
- **`mattmdjaga/segformer_b2_clothes`**: ATR 18클래스, 착장 파싱(상의 87%·바지 90%·원피스 74%, 벨트 등 소물 약함). *사람이 입은* 사진 파싱용.
- **누끼**: **rembg**(U2Net/ISNet) — 단일 옷 flat 사진 배경 제거. **자동분류 대안으로는 SegFormer보다 Gemini 비전 태깅이 단순**(한 벌 flat 사진엔 파싱보다 분류가 맞음).

---

## 4. 권고 

**Gemini 올인 + rembg.** 참고 앱과 동일 경로가 가장 빠르고 안전.
1. **옷 태깅·자동분류** → Gemini 비전 1콜 (controlled vocab enum으로 정규화)
2. **누끼** → rembg (로컬)
3. **코디 조합** → LLM 조합 선택 + 누끼 콜라주(프론트) — **생성 없음, 저비용**
4. **입어보기(가상 피팅)** → Gemini Nano Banana. **'저장/입어보기' 눌렀을 때 1장만 생성**(비용·지연 최소)

**MCM 특화 주의:** 럭셔리 제품(로고·비세토스)은 생성 시 왜곡 위험 → 착장 생성 결과에 **실물 MCM 누끼를 픽셀 오버레이**하는 하이브리드를 스파이크로 검토.

## 5. 장기/비용 (파킹랏)
- 사용자 수만 명·생성 비용 커지면 → 오픈소스 VTON(**CatVTON** 우선: 가벼움/빠름) self-host + GPU 인스턴스 고려.
- Gemini 이미지 생성 결과엔 **SynthID 워터마크** 포함(정책 확인).

## 6. 착수 전 확인
- Gemini API 키·무료 티어·이미지 생성 쿼터 (AI 담당 시은과)
- Nano Banana 최신 모델명(2.5 vs Nano Banana 2 Lite) 및 단가
- 무드 입력을 **프리텍스트+프리셋 칩**으로 할지(참고 앱 방식) → `POST /outfits`에 `situation:string` 추가 검토

---

## 출처
- [Gemini 2.5 Flash Image (Nano Banana) — Google Developers Blog](https://developers.googleblog.com/introducing-gemini-2-5-flash-image/)
- [Nano Banana 2 Lite / Gemini Omni Flash — Google Cloud Blog](https://cloud.google.com/blog/products/ai-machine-learning/nano-banana-2-lite-and-gemini-omni-flash-available)
- [Nano Banana 가격 — OpenRouter](https://openrouter.ai/google/gemini-2.5-flash-image)
- [Top 4 오픈소스 VTON 비교 — FASHN](https://fashn.ai/blog/comparing-the-top-4-open-source-virtual-try-on-viton-models)
- [오픈소스 VTON 비교 (2026) — Fashiolabs](https://fashiolabs.com/blog/open-source-virtual-try-on-compared)
- [Awesome-Try-On-Models — GitHub](https://github.com/Zheng-Chong/Awesome-Try-On-Models)
- [segformer_b2_clothes — HuggingFace](https://huggingface.co/mattmdjaga/segformer_b2_clothes)
