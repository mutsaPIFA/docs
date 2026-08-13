# 브랜치 · PR · 커밋 컨벤션

> 모든 레포(`backend`/`frontend`/`ai`/`docs`)에 동일 적용. 레포가 역할별로 분리돼 있으니 팀 prefix 없이 단순하게 간다.

## 브랜치 구조 (3단)

```
main    최종 데모용 · 안정 상태
  ↑ merge commit (데모 릴리스 시)
dev     개발 통합용 · 모든 작업이 모이는 곳
  ↑ Squash and merge (PR)
feature/기능이름   각자 작업 브랜치
```

| 브랜치 | 용도 | 누가 | 머지 방식 | 보호 |
|---|---|---|---|---|
| `main` | 최종 데모용, 안정 상태 | owner(선재)/합의 | `dev → main` merge commit | PR 필수, force push 금지 |
| `dev` | 개발 통합 | 팀원 PR로 | `feature/* → dev` **Squash and merge** | PR 필수 |
| `feature/*` | 각자 작업 | 작성자 본인 | — | 없음 |

- **모든 PR의 base는 `dev`.** (GitHub 기본값이 main이면 바꿀 것)
- `main`은 개발 중 직접 건드리지 않는다. **데모 직전(또는 주기적으로) `dev → main` PR**로 승격.

## 브랜치 이름

```
feature/기능이름      # 새 기능 (기본)
fix/버그이름          # 버그 수정
refactor/설명         # 리팩터
docs/설명             # 문서
chore/설명            # 설정·잡일
test/설명             # 테스트
```

- 설명은 **영어 소문자 + 하이픈**. 예: `feature/closet-scan`, `fix/login-cookie`, `docs/api-v1`.
- 이슈 번호 있으면 붙여도 됨: `feature/42-closet-scan`.

## 일상 작업 흐름

```bash
# 1. 최신 dev에서 브랜치 따기
git checkout dev
git pull origin dev
git checkout -b feature/closet-scan

# 2. 작업 중 dev가 업데이트되면 rebase (merge 말고)
git fetch origin
git rebase origin/dev

# 3. 작업 완료 → push → PR
git push origin feature/closet-scan
# GitHub에서 PR 생성: feature/closet-scan → dev (base가 dev인지 확인!)
```

- 리뷰 승인 후 **Squash and merge**로 dev에 통합 (히스토리 깔끔하게).
- 머지되면 그 feature 브랜치는 삭제.

## 커밋 메시지

```
{타입}: {한국어 설명}
```

**타입** — `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`

```
feat: 옷장 스캔 태깅 API 추가
fix: 로그인 쿠키 path 누락 수정
docs: API 정의서 v1 작성
refactor: 추천 서비스 id 재검증 분리
chore: docker-compose postgres 설정
```

**하지 말 것**
```
로그인 버그 수정        # 타입 없음 X
feat: 여러 기능 추가     # 너무 뭉뚱그림 X
feat: Added scan API    # 영어면 동사원형으로
```

- **하나의 커밋 = 하나의 논리적 변경.** "A도 B도"는 커밋 둘로 나눈다.

## FAQ

**Q. 레포가 여러 개인데 브랜치 전략은 레포마다 따로?**
그렇다. 각 레포가 독립적으로 `main`/`dev`/`feature/*`를 가진다. 여러 레포에 걸친 기능은 각 레포에 같은 이름의 feature 브랜치를 파고 각각 PR.

**Q. main에 급한 수정이 필요하면?**
owner 판단으로 `fix/*`를 main에서 따서 main에 PR → 이후 `main → dev` 역머지로 dev 동기화. 예외 상황이라 규칙보다 판단.

**Q. dev가 깨지면?**
깨트린 PR 작성자가 즉시 fix PR로 복구. dev는 항상 동작 상태 유지.
