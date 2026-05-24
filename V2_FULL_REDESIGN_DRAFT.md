# V2_FULL_REDESIGN_DRAFT.md

> **목적**: a4-page-flow-factory를 **새 폴더에서 처음부터** 다시 만들기 위한 V2 전체 재설계 초안.
> **상태**: Draft (구현 지시 아님, 설계 문서)
> **작성 시점**: 2026-05-24
> **선행 자료**: `docs/V1_CURRENT_STATE_REVIEW.md` (V1 현재 상태 검토) + 사용자 첨부 v1.2 잠금본
> **출력 제약**: 본 문서 1개만 작성. 코드/설치/커밋 없음.

---

## 0. 한 페이지 요약

| 항목 | 결정 |
|---|---|
| 형태 | 로컬 CLI (Mode A 소형 MVP) |
| 최종 산출물 | HTML/CSS → Playwright PNG (이미지 생성 AI 사용 안 함) |
| 1순위 워크플로우 | **반자동 프롬프트 출력** (사용자가 외부 LLM 웹에 붙여넣음, 비용 0) |
| LLM SDK 의존성 | **없음** (MVP). Phase 2부터 adapter 추가 |
| 도메인 핵심 자산 | 학습 자료실(content-learning-library) + `blocks.jsonl` 단일 진실 원본 |
| 프롬프트 레이어 | P1 EXTRACT / P2 CLASSIFY / P3 STYLE / P4 IMAGE_ELEM / P5 CHANNEL |
| MVP 게이트 | `instagram_portrait` 1개 profile로 6장 PNG 통과 + 품질 체크리스트 11/11 |
| V1 재사용 코드 | `cache.py` + `slicer.py` + `__main__.py` 패턴 (≈ 220 lines) |
| V1 폐기 | 5계층 파이프라인 / Anthropic SDK / 5 고정 템플릿 / 12 블록 타입 YAML 분류기 |
| 새 폴더 위치 (제안) | `a4-page-flow-factory-v2/` (V1 폴더와 분리) |

---

## 1. 사전 분석 (문서 작성 전 8개 질문)

### 1.1 V2 요구사항의 핵심 방향

| # | 방향 | 근거 (v1.2 §) |
|---|---|---|
| 1 | **반자동 프롬프트 우선** — CLI가 .md 프롬프트 파일을 만들고 사용자가 외부 LLM에 붙여넣음. 비용 0 | §1, §21, §22 |
| 2 | **HTML/CSS → Playwright PNG**가 최종 산출물. 이미지 생성 AI로 카드뉴스 생성 금지 | §1, §15, §29 |
| 3 | **학습 자료실(content-learning-library)이 단일 진실 원본**. blocks.jsonl 단일 파일 | §4, §6 |
| 4 | **OCR/Vision은 별도 파이프라인**. extract/ 폴더로 분리, 자동 적재 금지 | §20 |
| 5 | **자동게시·OAuth·DB·웹앱은 전부 Post-MVP**. placeholder만 유지 | §1, §24 |
| 6 | **수치 단정 금지** ("30% 향상" 같은 표현 안 씀) | §1 |
| 7 | **5개 output_profile** (a4_document / instagram_square / instagram_portrait / story_9_16 / threads_image) | §16 |
| 8 | MVP는 `instagram_portrait` 1개 profile부터 | §23 |

### 1.2 V1에서 가져갈 자산 (재사용 검토)

| 자산 | V1 라인 수 | V2 반영 형태 | 결정 |
|---|---|---|---|
| `cache.py` (SHA256 sharded JSON, Windows-safe) | 106 | `src/factoryv2/utils/cache.py`로 복사 | **유지** |
| `slicer.py` (PIL PNG 슬라이서) | 78 | `src/factoryv2/extract/slicer.py`로 복사 (위치 변경) | **유지+이동** |
| `__main__.py` Typer `@app.callback()` 패턴 (L1) | 36 | V2 entry에 즉시 적용 | **유지** |
| `init_cmd.py`의 `SEED_FILES` idempotent 패턴 | 266 | 패턴만 차용, 시드 콘텐츠는 v1.2 자료실로 교체 | **수정** |
| L1/L2/L3 학습 (Windows cp949 / PYTHONIOENCODING) | — | V2 1일차에 즉시 적용 | **유지** |
| `forbidden_phrases.txt` 20개 시드 | — | `brand/forbidden_words.md`로 이주 | **수정** |
| `business_profile.md` 템플릿 | — | `products/[slug]/meta.yml`로 분해 | **수정** |
| `style_notes.md` 템플릿 | — | `voice/[tone].md`로 분해 | **수정** |
| `asset_pack/manifest.yaml` 스키마 | — | `products/[slug]/source_images/` + manifest로 재배치 | **수정** |
| `pyproject.toml` ruff/mypy strict 설정 | — | anthropic 제거 후 그대로 | **수정** |
| `pyproject.toml` 의존성 `anthropic` | — | 제거 | **폐기** |
| `pyproject.toml` 의존성 `tenacity` | — | Phase 2까지 보류 | **폐기 (MVP)** |
| TDD RED → GREEN 분리 커밋 패턴 | — | V2도 동일 적용 | **유지** |
| `_make_png` 테스트 헬퍼 | — | conftest.py로 승격 | **수정** |
| `test_input_structure.py` parametrize 패턴 | — | V2 시드 구조 검증에 응용 | **유지** |

### 1.3 V1에서 버릴 구조

| 폐기 대상 | 폐기 이유 |
|---|---|
| 5계층 단방향 파이프라인 (Input→Analysis→Synthesis→Render→Validation) | V2는 "프롬프트 출력 → 외부 LLM → 결과 적재 → 렌더" 다이아몬드 흐름 |
| `src/factory/flow/abstractor.py` 가정 | flow_template 개념 자체가 사라짐. recipes.md가 대체 |
| `src/factory/synthesis/rewriter.py` 가정 | 외부 LLM이 처리. CLI는 프롬프트 빌더만 가짐 |
| `src/factory/ethics/{forbidden,similarity,watermark}.py` 가정 | v1.2 §1 "윤리/유사도/워터마크 하드게이트 제거" |
| `src/factory/manifest/{builder,validator}.py` 가정 | page_manifest.csv 18 컬럼 폐기 |
| 5개 고정 HTML 템플릿 (T1~T5) | output_profile 5개로 교체 (개념이 다름) |
| `config/block_types.yaml` 12 타입 고정 | block_type enum은 코드에 정의 (12→12 동일하지만 YAML+분류기 이중 구조 폐기) |
| Anthropic SDK 직접 호출 모듈 | 외부 LLM 웹 사용. SDK 불필요 (MVP) |
| `factory analyze --set --no-cache` 명령 시그니처 | 새 시그니처 `factory analyze [--target reference]` |
| `factory expand --pages 40` 명령 | 폐기. 대신 `target_pages=N` recipe 컬럼 |
| `factory cache clear/stats` 명령 | Post-MVP로 보류 |
| `PAGE_MANIFEST_SPEC.md` 286 lines | 폐기 |
| `REVIEW_CHECKLIST.md` 자동 점수 ≥80 산식 | 폐기 (v1.2는 PNG 품질 체크리스트로 교체) |
| `DESIGN_SPEC.md` 5계층 / 모듈 표 | 폐기 (본 문서로 교체) |
| `quality_score` 필드 가정 | v1.2 §25 명시적 금지 |
| 날짜형 ID | v1.2 §6 금지 |

### 1.4 V1 리뷰와 V2 요구사항이 충돌하는 부분 ⚠️

| # | V1 리뷰 권장 | V2 요구사항(v1.2 잠금본) | 외부 사용자 지시 | 본 문서 해결 |
|---|---|---|---|---|
| 1 | **새 폴더 V2 신규 생성** (Option B) | v1.2 §1 "현재 CLI 골격 **유지**", §32 "M2-5 GREEN" | "새 폴더에서 처음부터 만들 예정" | **사용자 지시 우선** — 새 폴더 `a4-page-flow-factory-v2/`. V1의 M2-5는 손대지 않음. v1.2의 "M2-5 GREEN"은 V1 폴더의 일이고, V2 폴더는 새로 시작 |
| 2 | "P1~P5 프롬프트 산출물" 검증 방법 잠금 권장 | v1.2는 P1~P5 코드명을 직접 안 씀 (`analyze` / 텍스트 / OCR/Vision / 이미지 / 채널 프롬프트로 흩어져 있음) | "P1~P5 프롬프트 산출물 검증 방식" 포함 요구 | **본 §7에서 5개 프롬프트를 P1~P5로 표준화 매핑** |
| 3 | `anthropic` 의존성 제거 | v1.2 §21 "API 자동 호출은 Phase 2"라며 adapter 폴더 구조 설계 | "anthropic 직접 의존성은 제거 후보로 본다" | **MVP는 제거**, Phase 2에서 adapter_anthropic.py 추가 시 다시 add. dependency-groups의 `ai` extras로 분리 |
| 4 | 5 고정 HTML 템플릿 폐기 | v1.2는 새 템플릿 설계를 명시 안 함 (output_profile만 5개) | "5개 고정 HTML 템플릿 폐기" | **MVP는 instagram_portrait 1개 템플릿**. 5개 강제하지 않음. profile당 1개씩 점진 추가 |
| 5 | 9개 문서 → 3개로 압축 | v1.2는 자체 문서 + 자료실 문서 추가 | "문서가 4개를 초과하면 위험 신호" | **3개로 잠금**: `SYSTEM_TECH.md` + `PLAN.md` + `CLAUDE.md`. 자료실 내부 문서(SKILL.md 등)는 자료지 메타 문서가 아님 — 별개 |
| 6 | `block_types.yaml` 폐기 | v1.2 §7은 block_type enum 12개 정의 | "12 타입 고정 폐기" | **YAML 파일은 폐기**, enum 12개는 `src/factoryv2/schema/enums.py`에 코드로 정의 (단일 정의) |

> **결론**: 사용자 외부 지시 > v1.2 잠금본 > V1 리뷰 (사용자가 잠금본 자체를 수정하는 새 지시를 줬으므로). 본 문서는 모든 충돌에서 **새 폴더 + P1~P5 표준화** 방향으로 통일.

### 1.5 V2에서 새로 정의해야 할 핵심 개념

| 신규 개념 | 정의 | 어디서 다루는가 |
|---|---|---|
| **반자동 프롬프트(Prompt Artifact)** | CLI가 출력하는 `.md` 파일. 사용자가 외부 LLM에 통째 붙여넣을 수 있는 완성 프롬프트 | §7, §8, §9 |
| **블록(blocks.jsonl)** | 재사용 가능한 카피 조각. 단일 JSONL 파일에 누적. 12개 block_type enum | §10, §11 |
| **레시피(recipe)** | 어떤 블록 시퀀스 + 어떤 channel + 어떤 tone으로 만들지 정의한 표 1행 | §10 |
| **output_profile** | 5개 캔버스 규격 (a4_document / instagram_square / instagram_portrait / story_9_16 / threads_image) | §13 |
| **추출 배치(extract batch)** | 6장 이상의 원본 이미지를 한 묶음으로 처리하는 단위 | §12 |
| **공급망 분리(Supply Separation)** | OCR/Vision → 사람 검수 → 적재. 자동 적재 금지 | §12 |
| **골든 샘플(golden sample)** | `examples/[slug]_[channel]_good_[NN].md` 검증된 결과 1~2개 | §11, §18 |
| **단일 진실 원본(SSOT)** | `blocks.jsonl`이 유일. 제품 폴더에 블록 중복 저장 금지 | §11 |
| **수치 단정 금지** | 카피·문서·UI 어디에도 "30% 향상" 류 사용 금지 | §15 |

### 1.6 P1~P5 프롬프트 산출물 검증 방식 (선검토)

전체 §9에서 상세히 다룸. 요약:

| 검증 축 | 방법 |
|---|---|
| **구조 검증** | 프롬프트 .md가 frontmatter + 필수 섹션 (역할/맥락/입력자료/원하는 출력형식/제약) 을 모두 포함하는지 |
| **재현성 검증** | 같은 입력 2회 실행 → 바이트 동일한 프롬프트 출력 (deterministic) |
| **참조 무결성** | 프롬프트가 인용하는 자료실 파일이 실제 존재하고, 인용 anchor가 유효한지 |
| **enum 정합성** | 프롬프트가 명시하는 tone/channel/block_type이 enum에 있는지 |
| **금지표현 미포함** | 프롬프트 본문에 forbidden_words 누출 없는지 |
| **골든 비교** | `tests/golden/prompts/P1_*.md` 고정 출력과 diff |
| **수동 외부 LLM 검증 (반복)** | 같은 프롬프트를 외부 LLM 3회 실행 → 결과 분산이 허용치 이내인지 (사람 평가) |

### 1.7 새 CLI가 실제로 출력해야 하는 파일

| 단계 | 출력 파일 | 형식 |
|---|---|---|
| `factory init` | `content-learning-library/` 전체 + `products/[slug]/` 시드 | 폴더+seed .md/.yml/.jsonl |
| `factory analyze` | `prompts/P1_extract_[batch].md` | 외부 LLM에 붙여넣을 프롬프트 |
| `factory extract import` | `products/[slug]/original_extracted.md` + `extracted.json` + `review.md` | 사람 검수용 |
| `factory generate` | `prompts/P3_style_[recipe].md` + `prompts/P5_channel_[recipe].md` | 외부 LLM 프롬프트 |
| `factory copy import` | `output/[recipe]/copy_master.json` | LLM 응답 적재본 |
| `factory sample` | `output/[recipe]/page_NNN.html` + `page_manifest.json` | 렌더 직전 manifest |
| `factory render` | `output/[recipe]/page_NNN.png` + `quality_report.json` | 최종 PNG |
| `factory doctor` | `stdout` (Playwright/font/encoding 진단) | CLI 출력 |

### 1.8 새 CLI에서 구현하지 말아야 할 기능

| 구현 금지 | 이유 |
|---|---|
| OAuth / Instagram·Threads·X API 클라이언트 | v1.2 §1, §25 |
| DB / Supabase / sqlite 사용 | v1.2 §1 — 파일 기반만 |
| 웹 UI / 대시보드 / Framer식 편집기 | v1.2 §25 |
| 이미지 생성 AI 직접 호출 (DALL·E / Imagen / Midjourney) | v1.2 §21, §29 — MVP 미사용 |
| 자동 발행 worker / Telegram 알림 / 캘린더 배치 | v1.2 §1, §24 — Post-MVP |
| 윤리·유사도·워터마크 하드게이트 | v1.2 §31 — 제거 결정됨 |
| 자동 점수 산식 (REVIEW_CHECKLIST ≥80 같은) | v1.2 §25 — `quality_score` 금지 |
| `factory expand --pages 40` 류의 대량 자동 생성 | v1.2 §25 |
| 진짜 파인튜닝 / 벡터 DB 도입 | v1.2 §24 (자료 1,000개 이전 금지) |
| LLM 응답 자동 파싱 후 blocks.jsonl 직접 적재 | v1.2 §20 — "사람 검수 후 한 줄씩" |
| 페이지 매니페스트 자동 점수 검증 (n-gram 유사도) | v1.2 §31 |
| `factory cache clear/stats` 명령 | Post-MVP. MVP에는 불필요 |

---

## 2. V2 제품 정의

| 항목 | 내용 |
|---|---|
| 프로젝트명 | `a4-page-flow-factory-v2` (V1 폴더와 물리적으로 분리) |
| 한 줄 정의 | 외부 LLM(웹)에 붙여넣을 **반자동 프롬프트**를 생성하고, 사람이 검수한 카피를 받아 **HTML/CSS → Playwright PNG**로 5개 채널 결과물을 만드는 로컬 CLI |
| 형태 | 로컬 CLI (Mode A 소형 MVP) |
| 사용자 | 1인 사업자 / 브랜드 운영자 (한국어) |
| 비용 모델 | MVP는 **무료** (외부 LLM은 사용자의 무료/유료 계정 사용) |
| 차별점 | (a) 이미지 생성 AI 없이 PNG 카드뉴스 (b) 학습 자료실 SSOT (c) 외부 LLM 비종속 |
| 최종 산출물 형태 | `.png` (5개 output_profile 중 선택) + `.html` (감사 가능) + `.json` (manifest) |

---

## 3. V2 핵심 목표 (Goals)

| ID | 목표 | 측정 방법 |
|---|---|---|
| G1 | 외부 LLM 사용 → 반자동 프롬프트 1개 출력 → 결과 적재 → PNG 1장까지 **15분 이내** | 사용자 실측 |
| G2 | 같은 입력 → 같은 프롬프트 .md 바이트 동일 (deterministic) | `pytest -q tests/test_prompts_deterministic.py` |
| G3 | `instagram_portrait` 1개 profile에서 6장 PNG 100% 통과 + 품질 체크리스트 11/11 | 자동 검증 |
| G4 | 외부 LLM 변경 (GPT↔Claude↔Gemini)에 CLI 변경 0줄 | 코드 의존성 grep |
| G5 | MVP 종료까지 LLM SDK 의존성 0개 (anthropic/openai/google-genai 모두 미설치) | `pyproject.toml` 검증 |

### 비목표 (Non-Goals)

- ⛔ 자동 발행 / OAuth / SNS API
- ⛔ DB / 로그인 / 결제 / 다중 사용자
- ⛔ 이미지 생성 AI로 글자 박힌 카드뉴스
- ⛔ 다국어 (한국어 우선)
- ⛔ 모바일 / 클라우드 / CI 배포
- ⛔ 진짜 파인튜닝 / 벡터 DB (자료 1,000개 미만)
- ⛔ 윤리/유사도 자동 차단 (제거 결정됨)
- ⛔ `quality_score` / 날짜형 ID

---

## 4. V2 사용자 흐름

### 4.1 정상 흐름 (Happy Path)

```
1. factory init                          → 자료실 + products/[slug] 시드 생성
2. (사용자) brand/voice/channels 편집      → 자료실 첫 채움
3. factory extract prompt --batch b001    → P1 추출 프롬프트 .md 생성
4. (사용자) 외부 LLM에 P1 붙여넣고 결과 회수
5. factory extract import --batch b001    → 회수 결과 → original_extracted.md + review.md
6. (사용자) review.md 누락/중복 검수 후 승인
7. factory blocks add ...                 → blocks.jsonl 1줄씩 사람이 추가
8. factory recipe new ...                 → recipes.md 1행 추가
9. factory generate --recipe r01          → P3 STYLE + P5 CHANNEL 프롬프트 .md 생성
10.(사용자) 외부 LLM에 붙여넣고 카피 회수
11.factory copy import --recipe r01       → copy_master.json 적재
12.factory sample --recipe r01            → page_manifest.json + page_NNN.html
13.factory render --recipe r01            → page_NNN.png + quality_report.json
14.(사용자) PNG 확인 후 사용
```

### 4.2 부분 재실행

- 카피만 다시: 11번부터 (10번 결과만 교체)
- 렌더만 다시: 13번부터 (디자인 토큰 변경 시)
- 채널만 다른 출력 추가: 9번부터 (recipe만 channel 바꿔 새로 생성)

### 4.3 증거 없을 때

- `products/[slug]/proof_assets.md` 비워두기
- P5 CHANNEL 프롬프트가 자동으로 "증거 없음 — 일반 설명형으로" 컨텍스트 주입
- 이전 V1의 evidence_substitute / watermark 강제는 폐기 (v1.2 §31)

---

## 5. V2 CLI 명령어 초안

### 5.1 명령 표

| 명령 | 인자 | 출력 | MVP/Post |
|---|---|---|---|
| `factory init` | `[--dry-run]` | `content-learning-library/`, `products/`, `extract/`, `output/` 시드 | MVP |
| `factory doctor` | `[--check playwright|encoding|fonts]` | stdout 진단 | MVP |
| `factory products new <slug>` | | `products/<slug>/{meta.yml, product_summary.md, source_images/}` | MVP |
| `factory extract prompt` | `--batch <id> [--mode manual|ocr]` | `extract/[batch]/prompt_P1.md` | MVP |
| `factory extract import` | `--batch <id> --from <path.md>` | `extract/[batch]/{extracted.md, extracted.json, review.md}` | MVP |
| `factory extract apply` | `--batch <id> --product <slug>` | `products/<slug>/original_extracted.md` | MVP |
| `factory blocks add` | `--from <md_file>` (interactive review) | `blocks/blocks.jsonl` 1줄 추가 | MVP |
| `factory blocks list` | `[--filter tone=... channel=...]` | stdout 표 | MVP |
| `factory recipe new` | `--name <s> --channel <c> --profile <p>` | `recipes/recipes.md` 1행 | MVP |
| `factory generate` | `--recipe <id>` | `prompts/[recipe]/{P3_style.md, P4_image.md, P5_channel.md}` | MVP |
| `factory copy import` | `--recipe <id> --from <path.md>` | `output/[recipe]/copy_master.json` | MVP |
| `factory sample` | `--recipe <id> [--only N]` | `output/[recipe]/{page_manifest.json, page_NNN.html}` | MVP |
| `factory render` | `--recipe <id> [--only N]` | `output/[recipe]/{page_NNN.png, quality_report.json}` | MVP |
| `factory ai send` | `--prompt <path> --model <m>` | (Phase 2) API 호출 결과 회수 | Post-MVP |
| `factory publish` | `--recipe <id> --channel <c>` | (Phase 3) | Post-MVP |
| `factory expand` | `--recipe <id> --pages N` | (Phase 4 — 대량 생성) | Post-MVP |

### 5.2 명령 그룹화 원칙

```
factory
  ├── init / doctor                       (환경)
  ├── products new                        (제품 컨텍스트)
  ├── extract prompt|import|apply         (P1 EXTRACT)
  ├── blocks add|list                     (자료실 적재)
  ├── recipe new|list                     (P2/P3/P5 입력)
  ├── generate                            (P3+P4+P5 프롬프트 생성)
  ├── copy import                         (외부 LLM 응답 적재)
  ├── sample / render                     (P4 IMAGE_ELEM + 렌더링)
  └── ai send / publish / expand          (Post-MVP)
```

> 명령은 4-단어 이하. `factory analyze` 는 의도가 모호해 V2에서는 사용 안 함.

---

## 6. V2 폴더 구조 초안

### 6.1 새 폴더 전체

```
a4-page-flow-factory-v2/
├── pyproject.toml
├── uv.lock
├── .python-version
├── .gitignore
├── README.md
├── SYSTEM_TECH.md             ← 본 문서의 V2 정착본 (단일 진실 원본)
├── PLAN.md                    ← 마일스톤 체크박스
├── CLAUDE.md                  ← AI 행동 규칙
├── docs/
│   ├── V2_FULL_REDESIGN_DRAFT.md   ← 본 문서 (참조용)
│   └── lessons/
│       └── L001_*.md          ← 발견된 학습 (V1 L1~L3 즉시 복사)
├── src/
│   └── factoryv2/
│       ├── __init__.py
│       ├── __main__.py        ← Typer entry + @app.callback() (L1 적용)
│       ├── schema/
│       │   ├── enums.py       ← block_type/tone/channel/etc.
│       │   ├── blocks.py      ← BlockRow dataclass + JSONL r/w
│       │   ├── manifest.py    ← PageManifest dataclass
│       │   └── meta.py        ← product meta.yml r/w
│       ├── utils/
│       │   ├── cache.py       ← V1에서 복사 (sharded JSON)
│       │   ├── ascii_safe.py  ← L2 적용 (em-dash 검사)
│       │   └── encoding.py    ← L3 적용 (PYTHONIOENCODING 점검)
│       ├── library/
│       │   ├── loader.py      ← 자료실 파일 읽기
│       │   ├── seed.py        ← init용 SEED_FILES dict
│       │   └── routing.py     ← SKILL.md 라우팅 표 구현
│       ├── extract/
│       │   ├── slicer.py      ← V1에서 복사 (PIL)
│       │   ├── prompt_builder.py  ← P1 EXTRACT 빌더
│       │   ├── importer.py    ← LLM 응답 파일 → extracted.{md,json}
│       │   └── review.py      ← review.md 생성
│       ├── prompts/
│       │   ├── p1_extract.py
│       │   ├── p2_classify.py
│       │   ├── p3_style.py
│       │   ├── p4_image.py
│       │   ├── p5_channel.py
│       │   └── frontmatter.py ← 공통 frontmatter
│       ├── render/
│       │   ├── html_builder.py
│       │   ├── browser.py     ← Playwright 관리
│       │   ├── screenshot.py
│       │   └── quality_check.py
│       ├── ai/                ← Post-MVP placeholder만
│       │   └── adapter_base.py
│       └── cli/
│           ├── init_cmd.py
│           ├── doctor_cmd.py
│           ├── products_cmd.py
│           ├── extract_cmd.py
│           ├── blocks_cmd.py
│           ├── recipe_cmd.py
│           ├── generate_cmd.py
│           ├── copy_cmd.py
│           ├── sample_cmd.py
│           └── render_cmd.py
├── tests/
│   ├── conftest.py            ← _make_png 등 공통 헬퍼
│   ├── golden/
│   │   └── prompts/
│   │       ├── P1_extract_v1.md
│   │       ├── P3_style_v1.md
│   │       └── P5_channel_v1.md
│   ├── test_enums.py
│   ├── test_cache.py          ← V1에서 복사
│   ├── test_slicer.py         ← V1에서 복사
│   ├── test_seed_init.py
│   ├── test_blocks_schema.py
│   ├── test_prompts_deterministic.py   ← G2 검증
│   ├── test_prompts_structure.py       ← frontmatter 누락 검증
│   ├── test_prompts_enum_consistency.py
│   ├── test_prompts_forbidden_leak.py
│   ├── test_html_builder.py
│   ├── test_screenshot_quality.py
│   ├── test_output_profile.py
│   └── test_e2e_minimal.py             ← MVP 게이트
├── content-learning-library/   ← 자료실 (사용자 데이터, .gitignore?)
│   ├── SKILL.md
│   ├── README.md
│   ├── system/
│   ├── brand/
│   ├── voice/
│   ├── channels/
│   ├── logic/
│   ├── products/
│   ├── blocks/blocks.jsonl
│   ├── benchmarks/
│   ├── recipes/recipes.md
│   └── examples/
├── extract/                    ← 추출 작업 폴더 (.gitignore)
│   ├── input/[batch_id]/
│   └── output/[batch_id]/
├── prompts/                    ← 생성된 프롬프트 (.gitignore)
│   └── [recipe_id]/
└── output/                     ← 렌더 결과 (.gitignore)
    └── [recipe_id]/
```

### 6.2 .gitignore 정책

| 폴더 | gitignore? | 이유 |
|---|---|---|
| `content-learning-library/` | **부분** | 구조(폴더+README+seed)는 commit, 사용자 자료(.jsonl/.yml의 user data)는 ignore — `.gitignore`로 패턴 분리 |
| `extract/` | **전체** | 사용자 데이터 |
| `prompts/` | **전체** | 생성물 (deterministic이므로 재생성 가능) |
| `output/` | **전체** | 생성물 |
| `docs/lessons/` | **commit** | 학습 자산 |

### 6.3 SSOT 문서 운영

| 문서 | 역할 | Lines 상한 |
|---|---|---|
| `SYSTEM_TECH.md` | 무엇을 만들 것인가 (제품/구조/스키마) | 500 |
| `PLAN.md` | 지금 무엇을 할 것인가 (체크박스) | 300 |
| `CLAUDE.md` | 어떻게 행동할 것인가 (AI 규칙) | 200 |

> 4번째 메타 문서 생성 금지 (사용자 외부 지시). 자료실 내부 문서(SKILL.md 등)는 자료이지 메타 아님.

---

## 7. V2 Prompt Layer P1~P5 구조

> **충돌 #2 해결**: v1.2 잠금본은 P1~P5 코드를 직접 쓰지 않음 → 본 §에서 외부 사용자 지시(P1~P5 잠금)와 v1.2의 4가지 프롬프트(`analyze` / 텍스트 / OCR/Vision / 이미지)를 표준화 매핑.

### 7.1 P1~P5 표준 정의

| 코드 | 이름 | 단위 | v1.2 매핑 | MVP/Post |
|---|---|---|---|---|
| **P1** | EXTRACT | batch 1개 (이미지 N장) | §20 OCR/Vision 반자동 | MVP |
| **P2** | CLASSIFY | batch 1개의 추출 결과 | §3-3 `factory analyze` (블록 추출용) | MVP |
| **P3** | STYLE | recipe 1개 | §21 텍스트 반자동 (voice/brand 부분) | MVP |
| **P4** | IMAGE_ELEM | image_slot 1개 | §21 이미지 반자동 | MVP (프롬프트만) |
| **P5** | CHANNEL | recipe 1개 | §21 텍스트 반자동 (channel/recipe 부분) | MVP |

### 7.2 P1~P5 책임 경계

| 코드 | 외부 LLM에게 시키는 것 | CLI가 하지 않는 것 |
|---|---|---|
| P1 EXTRACT | 이미지 → 텍스트/표/캡션을 마크다운으로 정리 | 이미지 직접 OCR (extract/ocr_only.py는 Post-MVP) |
| P2 CLASSIFY | 추출 텍스트 → block_type 12개로 분류, intent_summary 작성 | 자동 분류기 코드 (block_types.yaml + classifier.py 폐기) |
| P3 STYLE | brand + voice 입력 → 톤 가이드 1쪽 (LLM이 흡수) | 톤 자체 평가 |
| P4 IMAGE_ELEM | image_slot 메타 → 이미지 생성용 자연어 프롬프트 | 이미지 직접 생성 (DALL·E 등 호출 금지) |
| P5 CHANNEL | P2 블록 + P3 톤 + channel 가이드 → 페이지 카피 (copy_master.json 스펙으로) | 카피 자동 평가, 유사도 검사 (제거됨) |

### 7.3 P1~P5 출력 디렉토리

```
prompts/
├── [batch_id]/
│   ├── P1_extract.md         ← extract prompt
│   └── P2_classify.md        ← 추출 회수 후 실행
└── [recipe_id]/
    ├── P3_style.md
    ├── P4_image_NNN.md       ← slot 수만큼
    └── P5_channel.md
```

---

## 8. 각 Prompt Layer의 입력/출력

### 8.1 입출력 표

| 코드 | 입력 (CLI 읽음) | 출력 (.md) | 외부 LLM 결과 회수 (CLI 적재) |
|---|---|---|---|
| **P1** | `extract/input/[batch]/*.png` (메타만, 사용자가 본문 안 보냄) + `brand/forbidden_words.md` + `system/naming_rules.md` | `prompts/[batch]/P1_extract.md` | `extract/[batch]/extracted.md` + `extracted.json` → `factory extract import` |
| **P2** | `extract/[batch]/extracted.md` (사람 검수 후) + enum 정의 | `prompts/[batch]/P2_classify.md` | `extract/[batch]/classified.jsonl` → `factory blocks add` (interactive) |
| **P3** | `brand/brand_identity.md` + `brand/forbidden_words.md` + `voice/[tone].md` + recipe의 tone 필드 | `prompts/[recipe]/P3_style.md` | (출력 없음, P5 안에 흡수) |
| **P4** | recipe + 각 image_slot 메타 (channel/profile/position) | `prompts/[recipe]/P4_image_NNN.md` (slot 수만큼) | (선택) image 생성 외부 도구 사용 후 `output/[recipe]/assets/img_NNN.png` |
| **P5** | recipe + 매칭된 `blocks.jsonl` 행 + P3 결과 + `channels/[ch].md` + `logic/[role].md` + `examples/[good_NN].md` | `prompts/[recipe]/P5_channel.md` | `output/[recipe]/copy_master.json` → `factory copy import` |

### 8.2 P5 → copy_master.json 스펙 (LLM이 채워야 할 형식)

LLM이 다음 JSON을 응답해야 함 (P5 프롬프트 안에 스키마 명시):

```json
{
  "recipe_id": "r01",
  "channel": "instagram_cardnews",
  "output_profile": "instagram_portrait",
  "pages": [
    {
      "page_no": 1,
      "template_role": "hook",
      "blocks_used": ["b_kj92"],
      "copy": { "headline": "...", "subhead": "...", "body": "..." },
      "image_slot_id": "img_001",
      "notes": ""
    }
  ]
}
```

### 8.3 frontmatter 공통 규격 (모든 P1~P5 출력에 강제)

```yaml
---
prompt_code: P3
prompt_version: v1
generated_by: factoryv2 0.1.0
generated_at: 2026-05-24T10:00:00+09:00
deterministic_hash: <sha256>     # G2 검증용
inputs:
  - content-learning-library/brand/brand_identity.md
  - content-learning-library/voice/expert_saas_tone.md
recipe_id: r01                    # P3/P5만
batch_id: b001                    # P1/P2만
---
```

---

## 9. P1~P5 산출물 검증 기준

### 9.1 7개 검증 축

| 검증 ID | 이름 | 자동/수동 | 통과 기준 |
|---|---|---|---|
| V-STRUCT | frontmatter + 필수 섹션 | 자동 | frontmatter 6필드 + 본문 5섹션 (역할/맥락/입력자료/원하는 출력형식/제약) 전부 존재 |
| V-DETERM | 재현성 | 자동 | 같은 입력 2회 → 바이트 동일 (`deterministic_hash` 검증, `generated_at` 제외 비교) |
| V-REF | 참조 무결성 | 자동 | frontmatter `inputs:`의 모든 경로 존재 + 인용 anchor 유효 |
| V-ENUM | enum 정합성 | 자동 | tone/channel/block_type 값이 `schema/enums.py`에 정의됨 |
| V-FORB | 금지어 미포함 | 자동 | `brand/forbidden_words.md` 라인이 프롬프트 본문에 들어가지 않음 (역설명 OK, 직접 사용 X) |
| V-GOLD | 골든 diff | 자동 | `tests/golden/prompts/P*_*.md`와 차이가 frontmatter `generated_at` 외 0 |
| V-MANUAL | 외부 LLM 분산 | 수동 | 같은 프롬프트 → 외부 LLM 3회 → 결과의 핵심 메시지가 사람 평가 일치 |

### 9.2 P1~P5별 추가 검증

| 코드 | 추가 검증 |
|---|---|
| P1 | extract input batch 폴더에 이미지가 1장 이상 존재해야 빌드 가능 |
| P2 | P1 회수 결과(extracted.md)가 사람 승인(review.md APPROVED) 후에만 빌드 가능 |
| P3 | recipe.tone이 `voice/` 폴더의 파일과 1:1 대응 |
| P4 | slot 수와 channel의 권장 image_slot 수 일치 (instagram_cardnews=6 등) |
| P5 | recipe.channel + recipe.output_profile + recipe.tone 3중 정합성 |

### 9.3 검증 실패 시 동작

- V-STRUCT/V-DETERM/V-REF/V-ENUM/V-FORB/V-GOLD: **CLI exit 1** + 어떤 검증이 깨졌는지 표시
- V-MANUAL: CLI 강제 없음. `examples/`에 골든 샘플 1~2개를 만들어 사람이 비교

### 9.4 골든 픽스처 운영

```
tests/golden/prompts/
├── P1_extract_v1.md      ← 한 번 만들고 lock. 변경하려면 PR 리뷰 필수
├── P2_classify_v1.md
├── P3_style_v1.md
├── P4_image_v1.md
└── P5_channel_v1.md
```

생성 절차:
1. 픽스처용 입력 `tests/fixtures/recipe_golden.yml` 등 고정
2. `factory generate --recipe golden` 실행
3. 첫 결과를 `tests/golden/`에 복사
4. 이후 `test_prompts_deterministic.py`가 매번 비교

---

## 10. V2 데이터 흐름

### 10.1 다이아몬드 흐름 (V1 5계층 폐기 후 신규)

```
                       ┌────────────────────┐
                       │ content-learning-  │
                       │ library/ (SSOT)    │
                       └────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        ┌──────────┐      ┌──────────┐      ┌──────────┐
        │ products/│      │ blocks.  │      │ recipes. │
        │ [slug]/  │      │ jsonl    │      │ md       │
        └────┬─────┘      └────┬─────┘      └────┬─────┘
             │                 │                 │
             └────────┬────────┴────────┬────────┘
                      │                 │
                      ▼                 ▼
              ┌─────────────┐   ┌──────────────┐
              │ P1 EXTRACT  │   │ P3 STYLE     │
              │ prompt .md  │   │ prompt .md   │
              └──────┬──────┘   └──────┬───────┘
                     │                 │
                     ▼                 ▼
              [외부 LLM 웹]     [외부 LLM 웹]
                     │                 │
                     ▼                 ▼
              extracted.md       P5 CHANNEL prompt
                     │                 │
                     ▼                 ▼
              P2 CLASSIFY        [외부 LLM 웹]
                     │                 │
                     ▼                 ▼
              blocks.jsonl       copy_master.json
              (사람 검수)              │
                                       ▼
                                 page_manifest.json
                                       │
                                       ▼
                                 page_NNN.html
                                       │
                                       ▼
                                 Playwright
                                       │
                                       ▼
                                 page_NNN.png + quality_report.json
```

### 10.2 모듈 간 통신 원칙

| 원칙 | 설명 |
|---|---|
| **파일로만 통신** | 모듈 직접 함수 호출 금지. 파일 입출력으로만 (V1 원칙 계승) |
| **사람 게이트 명시** | 외부 LLM 결과 회수 후 `factory ... import`가 필수 — 직접 적재 금지 |
| **idempotent** | 모든 `factory *` 명령은 재실행해도 안전 (V1 `init` 패턴 계승) |
| **deterministic 빌드** | 프롬프트 .md는 같은 입력이면 같은 출력 (G2) |
| **render는 매번 새로** | PNG 생성은 비결정적이라 idempotent 보장 안 함 |

---

## 11. blocks.jsonl 스키마 (V2 잠금)

> v1.2 §6과 동일. 본 문서에서 한 번 더 잠금.

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `id` | string | ✅ | 짧은 랜덤 (예: `b_kj92x`). 날짜 미포함 |
| `block_type` | enum(12) | ✅ | hook/problem/pain_point/solution/benefit/proof/before_after/objection/cta/trust/feature/comparison |
| `text` | string | ✅ | 블록 본문 |
| `source_type` | enum(3) | ✅ | own/benchmark/user_added |
| `source_product` | string\|null | | 제품 slug |
| `source_file` | string | ✅ | 경로 |
| `source_anchor` | string | ✅ | `#section-slug` 또는 `L42-L58` |
| `industry` | snake_case | | 자유 입력, snake_case 강제 |
| `tone` | enum | | voice/ 파일명 |
| `channel` | enum(6) | | channels/ 파일명 |
| `logic_role` | enum(5) | | logic/ 파일명 |
| `tags` | string[] | | snake_case, 영문만 |
| `status` | enum(3) | ✅ | draft/approved/archived |
| `reuse_note` | string | | |
| `created_at` | ISO8601 | ✅ | |
| `updated_at` | ISO8601 | ✅ | |

**금지**: `quality_score`, 날짜형 `id`

---

## 12. 추출(extract) 파이프라인 — 공급망 분리 원칙

### 12.1 4단계

| 단계 | CLI 명령 | 책임 |
|---|---|---|
| 1. 준비 | (없음 — 사용자가 `extract/input/[batch]/*.png` 배치) | 사용자 |
| 2. 프롬프트 빌드 | `factory extract prompt --batch b001` | CLI (P1 출력) |
| 3. 외부 회수 + 검수 | `factory extract import --batch b001 --from <path>` | CLI 적재 + 사용자 검수 (review.md) |
| 4. 적용 | `factory extract apply --batch b001 --product trusta` | products/[slug]/original_extracted.md |

### 12.2 자동 적재 금지

- LLM 응답을 받아도 **`blocks.jsonl`에 자동 추가하지 않음**
- 항상 사람이 `factory blocks add`로 한 줄씩 추가 (interactive review)
- `review.md`에 누락/중복 표기 후 승인 절차

### 12.3 모드 (v1.2 §20 차용)

| 모드 | MVP/Post | 비고 |
|---|---|---|
| `manual` | MVP | 사용자가 외부 LLM에 P1 붙여넣음. CLI는 결과 파일만 받음 |
| `ocr` | Post | Tesseract 등 로컬 OCR. 비용 거의 0 |
| `vision` | Post | Claude/GPT-4V Vision API. 비용 있음 |
| `ocr+vision` | Post | OCR 1차 + Vision 2차 검수 |

---

## 13. output_profile 5개 잠금 (v1.2 §16)

| profile | width | height | 용도 | MVP/Post |
|---|---|---|---|---|
| `instagram_portrait` | 1080 | 1350 | 4:5 카드뉴스 | **MVP 1순위** |
| `a4_document` | 1240 | 1754 | A4 세로 | MVP 2순위 |
| `instagram_square` | 1080 | 1080 | 1:1 카드뉴스 | MVP 3순위 |
| `story_9_16` | 1080 | 1920 | 스토리/릴스 커버 | MVP 4순위 |
| `threads_image` | 1080 | 1350 | Threads 이미지형 | MVP 5순위 |

### Playwright 규칙 (v1.2 §17)

| 규칙 | 값 |
|---|---|
| device_scale_factor | ≥ 2 |
| viewport | profile width/height |
| full_page | false |
| 배경 | white 또는 theme bg (투명 금지) |
| 폰트 대기 | `document.fonts.ready` |
| 네트워크 | `networkidle` |
| 파일명 | `page_001.png`, `page_002.png`, ... |

### PNG 품질 체크리스트 (v1.2 §18, 11항목)

자동 검증 — `quality_report.json`에 결과 기록.

---

## 14. V1 재사용 자산 목록 (최종)

| 자산 | V1 경로 | V2 경로 | 변경 |
|---|---|---|---|
| cache.py | `src/factory/analyze/cache.py` | `src/factoryv2/utils/cache.py` | 위치만 변경 |
| slicer.py | `src/factory/analyze/slicer.py` | `src/factoryv2/extract/slicer.py` | 위치만 변경 |
| Typer entry + callback | `src/factory/__main__.py` | `src/factoryv2/__main__.py` | help 텍스트 ASCII 강제 (L2) |
| init_cmd 패턴 | `src/factory/cli/init_cmd.py` | `src/factoryv2/cli/init_cmd.py` + `library/seed.py` | SEED_FILES 콘텐츠 전면 교체 |
| forbidden_phrases 시드 | `input/forbidden_phrases.txt` | `content-learning-library/brand/forbidden_words.md` | 마크다운 형식으로 |
| business_profile 시드 | `input/business_profile.md` | `content-learning-library/products/_example/meta.yml` + `product_summary.md` | 분해 |
| style_notes 시드 | `input/style_notes.md` | `content-learning-library/voice/expert_saas_tone.md` (등) | 분해 |
| asset_pack/manifest.yaml | `asset_pack/manifest.yaml` | `content-learning-library/products/_example/source_images/manifest.yaml` | 위치만 |
| pyproject.toml ruff/mypy | `pyproject.toml` | `pyproject.toml` | anthropic/tenacity 제거 |
| test_cache.py | `tests/test_cache.py` | `tests/test_cache.py` | 그대로 |
| test_slicer.py | `tests/test_slicer.py` | `tests/test_slicer.py` | 그대로 |
| _make_png 헬퍼 | (각 테스트 파일 내부) | `tests/conftest.py` | 승격 |
| LessonsLearned L1/L2/L3 | `LessonsLearned.md` | `docs/lessons/L001~L003.md` | 분리 + 즉시 적용 |

**총 재사용 코드: ~220 lines** (cache 106 + slicer 78 + main 36)
**총 재사용 시드: ~30 lines** (선별 후)

---

## 15. V1 폐기 자산 목록 (최종)

| 폐기 대상 | 위치 | 폐기 사유 |
|---|---|---|
| `flow/` 모듈 가정 | (미생성) | recipes.md로 대체 |
| `synthesis/` 모듈 가정 | (미생성) | 외부 LLM이 처리 |
| `ethics/` 모듈 가정 | (미생성) | v1.2 §31 하드게이트 제거 |
| `manifest/` 모듈 가정 | (미생성) | page_manifest.csv 폐기 |
| `analyze/vision.py` 가정 | (미생성) | API 직접 호출 폐기 |
| `analyze/classifier.py` 가정 | (미생성) | P2 프롬프트로 대체 |
| `config/block_types.yaml` 가정 | (미생성) | enums.py 단일 정의 |
| `config/templates.yaml` 가정 | (미생성) | output_profile로 대체 |
| `config/ethics_rules.yaml` 가정 | (미생성) | 폐기 |
| `config/vision_prompts.yaml` 가정 | (미생성) | prompts/p1_extract.py로 대체 |
| `factory analyze --set` 시그니처 | V1 미구현 | `factory extract prompt/import/apply` 3분할 |
| `factory sample --only N` 시그니처 | V1 미구현 | `factory sample --recipe <id> [--only N]` |
| `factory expand --pages 40` | V1 미구현 | Post-MVP로 보류 |
| `factory cache clear/stats` | V1 미구현 | Post-MVP |
| `anthropic>=0.40` 의존성 | pyproject.toml | MVP는 SDK 없음 |
| `tenacity>=9.0` 의존성 | pyproject.toml | MVP는 retry 불필요 |
| 9개 문서 (CLAUDE/spec/plan/PRD/DESIGN_SPEC/SAMPLE_FLOW/PAGE_MANIFEST_SPEC/REVIEW_CHECKLIST/README) | V1 root | 3개로 압축 (SYSTEM_TECH/PLAN/CLAUDE) |
| `docs/future/UIUX_PRD.md` (651 lines) | V1 docs/future | v1.2 잠금본과 정합성 재검토 후 결정 (V3) |
| `DESIGN_SPEC.md` 5계층 다이어그램 | V1 root | 본 §10 다이아몬드 흐름으로 교체 |
| `REVIEW_CHECKLIST.md` 자동 점수 산식 | V1 root | PNG 품질 체크리스트(§13)로 대체 |
| `quality_score` 필드 가정 | (개념) | v1.2 §25 명시 금지 |
| 날짜형 ID 가정 | (개념) | v1.2 §6 금지 |

---

## 16. MVP 범위 (잠금)

### 16.1 MVP에 반드시 포함 (v1.2 §23 기반)

| # | 항목 | 검증 게이트 |
|---|---|---|
| 1 | `content-learning-library/` 폴더 구조 + `SKILL.md` + `naming_rules.md` | `pytest tests/test_seed_init.py` |
| 2 | `blocks.jsonl` 스키마 + 시드 10~20개 | `pytest tests/test_blocks_schema.py` |
| 3 | enum 12개 (block_type) + tone/channel/source_type/status/logic_role enum 정의 | `pytest tests/test_enums.py` |
| 4 | `factory init` (idempotent, L1/L2 적용) | `pytest tests/test_seed_init.py` |
| 5 | `factory doctor` (Playwright + encoding + fonts) | 수동 |
| 6 | `factory extract prompt` (P1 EXTRACT) | V-STRUCT/V-DETERM/V-REF |
| 7 | `factory extract import/apply` | V-STRUCT |
| 8 | `factory blocks add/list` (interactive) | 수동 + 적재 후 schema 검증 |
| 9 | `factory recipe new` | `pytest tests/test_recipe_schema.py` |
| 10 | `factory generate` (P3 + P4 + P5) | V-STRUCT/V-DETERM/V-REF/V-ENUM/V-FORB/V-GOLD 6종 |
| 11 | `factory copy import` (copy_master.json 스키마 검증) | `pytest tests/test_copy_schema.py` |
| 12 | `factory sample --recipe r01` (HTML + page_manifest.json) | `pytest tests/test_html_builder.py` |
| 13 | `factory render --recipe r01` (Playwright PNG, instagram_portrait만) | PNG 품질 체크리스트 11/11 |
| 14 | E2E 최소 흐름: init → generate (golden recipe) → copy import (fixture) → sample → render → PNG 통과 | `pytest tests/test_e2e_minimal.py` |

### 16.2 MVP에 포함하지 않는 것

- output_profile 4개 (a4_document/instagram_square/story_9_16/threads_image — instagram_portrait 1개만)
- OCR / Vision API 직접 호출
- 텍스트/이미지 LLM API 직접 호출
- 자동 발행 / Telegram / 캘린더
- 웹 UI / DB
- `factory expand --pages N` 대량 생성

---

## 17. Post-MVP 범위

| Phase | 항목 | 트리거 |
|---|---|---|
| Phase 2 | 텍스트 LLM API adapter (OpenAI/Anthropic/Google) | MVP 안정화 + 사용자 빈도 1주 5회 이상 |
| Phase 2 | Vision API 직접 호출 + cost_guard | 텍스트 adapter 완료 후 |
| Phase 2 | 나머지 4개 output_profile (a4_document → square → story → threads) | instagram_portrait 골든 1개 통과 후 |
| Phase 3 | 이미지 생성 API (배경/일러스트 슬롯만) | Phase 2 cost_guard 검증 후 |
| Phase 3 | `publish_adapter` 실제 구현 (Threads → IG → X 순) | 사용자 명시 요청 후 |
| Phase 4 | Telegram/PC 승인 알림 | Phase 3 운영 누적 후 |
| Phase 4 | 캘린더 자동 배치 / 대량 자동 생성 | 사용자 명시 요청 후 |
| Phase 5 | 벡터 DB 검토 | `blocks.jsonl` 1,000행 이상 시 |
| 미정 | 진짜 파인튜닝 | **검토 안 함 (당분간)** — v1.2 §24 |

---

## 18. 테스트 전략

### 18.1 테스트 피라미드 (V2 신규)

```
                     ┌─────────────────┐
                     │  E2E (1개)      │ test_e2e_minimal.py
                     │  init→render    │
                     └────────┬────────┘
                  ┌───────────┴────────────┐
                  │  Golden / Snapshot     │ test_prompts_deterministic.py
                  │  P1~P5 골든 픽스처     │ test_html_builder.py (HTML snapshot)
                  │  (5~8개)               │ test_screenshot_quality.py
                  └───────────┬────────────┘
            ┌─────────────────┴──────────────────┐
            │  Schema / Structure (15~20개)      │ test_enums.py
            │  enum/blocks/recipe/copy/prompt    │ test_blocks_schema.py
            │  스키마 단위 테스트                │ test_prompts_structure.py
            └─────────────────┬──────────────────┘
  ┌───────────────────────────┴──────────────────────────────┐
  │  Unit / Utility (20~30개)                                │ test_cache.py (V1)
  │  cache, slicer, ascii_safe, encoding, frontmatter        │ test_slicer.py (V1)
  └──────────────────────────────────────────────────────────┘
```

### 18.2 V1 대비 변화

| V1 비율 | V2 비율 |
|---|---|
| 인프라 100% (47/47) | 인프라 30% / 스키마 30% / 골든 25% / E2E 15% |

### 18.3 TDD 운영 규칙 (V1 계승)

- 모든 신규 모듈은 RED → GREEN 분리 커밋 (`feat(...): add X with RED→GREEN tests`)
- 100 lines/commit 원칙
- 한 커밋 = 한 마일스톤 (PLAN.md 체크박스 1개)

### 18.4 골든 픽스처 변경 정책

- `tests/golden/` 변경은 PR 본문에 "왜 골든을 바꿔야 하는가" 명시 + 사용자 승인
- 자동 갱신 금지 (`--update-snapshot` 옵션 비활성)

### 18.5 E2E 최소 흐름

```python
# tests/test_e2e_minimal.py (의사 코드)
def test_e2e_init_to_png(tmp_path):
    # 1. init
    factory init --project-root tmp_path
    # 2. golden recipe + golden copy_master fixture 복사
    copy fixtures/recipe_golden.yml → tmp_path/.../recipes/recipes.md
    copy fixtures/copy_master_golden.json → tmp_path/output/golden/copy_master.json
    # 3. sample
    factory sample --recipe golden
    assert (tmp_path/"output/golden/page_manifest.json").exists()
    # 4. render (1장만)
    factory render --recipe golden --only 1
    assert (tmp_path/"output/golden/page_001.png").exists()
    assert (tmp_path/"output/golden/quality_report.json").exists()
    # 5. quality check 11/11
    report = json.loads(...)
    assert report["passed"] == 11
```

---

## 19. 첫 1주차 구현 순서 (PLAN.md 초안)

### Week 1 (5일, 1일 6h 기준)

| Day | 마일스톤 | 산출 | 검증 |
|---|---|---|---|
| **Day 1** | V0 셋업 | 새 폴더 + pyproject.toml + .gitignore + SYSTEM_TECH/PLAN/CLAUDE 3문서 + uv sync + `factory init` 빈 명령 + `@app.callback()` | `uv sync` exit 0 + `factory --help` cp949 안전 |
| **Day 1** | V0+ V1 자산 이식 | cache.py + slicer.py + test_cache.py + test_slicer.py 복사 + conftest.py에 _make_png 승격 | `pytest tests/test_cache.py tests/test_slicer.py` 33/33 PASS |
| **Day 2** | V1 schema 정의 | `schema/enums.py` (12 block_type + 5 channel + 3 status + 3 source_type + 5 logic_role) + `schema/blocks.py` (BlockRow dataclass + JSONL r/w) | test_enums.py + test_blocks_schema.py PASS |
| **Day 2** | V2 자료실 시드 | `library/seed.py` SEED_FILES + `factory init` 구현 + 14개 폴더/시드 파일 생성 idempotent | test_seed_init.py 14/14 PASS |
| **Day 3** | V3 P3 STYLE 프롬프트 | `prompts/frontmatter.py` + `prompts/p3_style.py` + `factory generate --recipe golden --only-p3` + 골든 픽스처 1개 | V-STRUCT + V-DETERM + V-REF + V-ENUM + V-FORB + V-GOLD 6/6 PASS |
| **Day 3** | V3+ P5 CHANNEL 프롬프트 | `prompts/p5_channel.py` + `factory generate --recipe golden` | 동일 6/6 PASS |
| **Day 4** | V4 카피 적재 | `schema/manifest.py` (copy_master.json + page_manifest.json) + `factory copy import` + `factory sample --recipe golden` (HTML만) | test_copy_schema + test_html_builder PASS |
| **Day 5** | V5 렌더 (instagram_portrait) | `render/browser.py` + `render/screenshot.py` + `render/quality_check.py` + `factory render --recipe golden --only 1` | PNG 11/11 품질 통과 + test_e2e_minimal PASS |

### Week 1 종료 게이트

- [ ] `factory init → generate → copy import → sample → render` 1회 통과 (PNG 1장)
- [ ] 인프라 33 PASS + schema 15 PASS + 골든 6 PASS + E2E 1 PASS ≈ **55 tests**
- [ ] `pyproject.toml`에 `anthropic` 미설치 확인
- [ ] L1/L2/L3 모두 코드에 반영
- [ ] PLAN.md / SYSTEM_TECH.md 인수인계 메모 작성

### 1주차 위험 신호 (즉시 HALT)

- Day 1 종료까지 `factory --help`가 cp949에서 깨짐 → L2 미적용
- Day 2 종료까지 `_example` 시드 외 `blocks.jsonl` 시드 0줄 → SSOT 미실현
- Day 3 종료까지 P5 골든 픽스처 미생성 → 검증 축 부재
- Day 5까지 PNG 1장 미생성 → 렌더 함정에 빠진 것 (V1 재현)
- `anthropic` 또는 LLM SDK가 의존성에 추가됨 → V2 기본 가정 위반

---

## 20. 하지 말아야 할 것 (V2 최종 잠금)

> v1.2 §25 + 본 분석 추가

| 금지 항목 | 출처 |
|---|---|
| 자동게시 / OAuth / Instagram·Threads·X API | v1.2 §25 |
| DB / Supabase / SQLite | v1.2 §25 |
| 웹앱 로그인 / 결제 | v1.2 §25 |
| Framer식 드래그 편집 UI | v1.2 §25 |
| 복잡한 Insight 대시보드 | v1.2 §25 |
| 전환율 수치 단정 ("30% 향상") | v1.2 §1, §25 |
| 진짜 파인튜닝 | v1.2 §24 |
| 벡터 DB 즉시 도입 | v1.2 §24 |
| 제품 폴더와 blocks 폴더에 같은 블록 중복 | v1.2 §25 |
| `quality_score` 필드 | v1.2 §25 |
| 날짜형 ID | v1.2 §25 |
| 이미지 생성 AI로 글자 박힌 카드뉴스 | v1.2 §29 |
| 모델 API key부터 넣기 | v1.2 §25 |
| 대량 생성부터 만들기 | v1.2 §25 |
| DB migration부터 만들기 | v1.2 §25 |
| 여러 기능을 한 커밋에 섞기 | v1.2 §25 |
| **V1 폴더 직접 수정** | 사용자 외부 지시 |
| **`anthropic` MVP 의존성 추가** | 본 §3 G5 |
| **4번째 메타 문서 생성** | 사용자 외부 지시 |
| **자료실에 한글 enum 도입** | v1.2 §7 |
| **LLM 응답 자동 → blocks.jsonl 직접 적재** | v1.2 §20 |

---

## 21. V3 반영을 위해 남겨둘 열린 질문

> 본 V2 잠금본 단계에서 결정하지 않고, MVP 운영 경험 후 V3에서 재검토.

| ID | 질문 | 결정 시점 |
|---|---|---|
| Q1 | 5개 output_profile 외 신규 profile 허용 기준? (예: linkedin / x_card) | V3 |
| Q2 | `blocks.jsonl` 1,000행 도달 시 벡터 검색 도입할까? | 1,000행 달성 후 V3 |
| Q3 | recipes.md 한 파일 한계는? (recipes/ 폴더 분할 임계) | recipes 100행 도달 후 V3 |
| Q4 | golden 픽스처 변경 PR 자동 승인 정책? | MVP 3개월 운영 후 V3 |
| Q5 | `examples/` 골든 샘플 자동 생성 가능 시점? | MVP 6개월 후 V3 |
| Q6 | `docs/future/UIUX_PRD.md` (V1 651 lines) 채택 여부 | V3에서 결정 |
| Q7 | Phase 2 LLM adapter 1순위 (OpenAI vs Anthropic vs Google)? | 사용자 결제 환경 확정 후 V3 |
| Q8 | `factory ai send`가 단일 모델 vs 멀티 모델 fan-out? | Phase 2 진입 시 V3 |
| Q9 | 이미지 생성 API의 배경 슬롯 한정 사용 vs 일러스트 허용? | Phase 3 검토 시 V3 |
| Q10 | V1 cache.py를 그대로 vs 메타데이터 추가 버전? | MVP 캐시 사용 빈도 측정 후 V3 |
| Q11 | `factory expand --pages N` 대량 생성 재도입 여부 | Phase 4 검토 시 V3 |
| Q12 | 윤리 가드레일을 다시 도입할지 (v1.2 §31에서 제거됨) | MVP 운영 사고 사례 발생 시 V3 |
| Q13 | `industry` snake_case 자유 입력의 enum화 시점 | 누적 50종 도달 후 V3 |
| Q14 | `extract/` Tesseract OCR 도입 여부 | 사용자가 외부 LLM 비용 부담 호소할 때 V3 |
| Q15 | 멀티 사용자 / 팀 협업 기능 검토 | "한 명만 쓴다"는 가정이 깨질 때 V3 |

---

## 22. 결론

### 22.1 본 문서의 자기 평가

| 평가 축 | 결과 |
|---|---|
| V1 리뷰와 v1.2 잠금본의 충돌 6건을 명시적으로 해결했는가? | ✅ §1.4 |
| P1~P5 표준 매핑을 v1.2의 흩어진 프롬프트와 정합시켰는가? | ✅ §7 |
| MVP 게이트가 측정 가능한가? | ✅ §3 G1~G5 + §16 14개 |
| 1주차 위험 신호가 명시되어 있는가? | ✅ §19 5개 신호 |
| 코드 작성 지시 없이 설계만 다뤘는가? | ✅ |
| V2 4번째 메타 문서 생성 위험을 회피했는가? | ✅ §6.3 (3개 잠금) |

### 22.2 다음 액션 (본 문서 *외부*)

1. 본 문서를 사용자 검토 → 승인되면 새 폴더 `a4-page-flow-factory-v2/` 생성
2. 새 폴더에 본 문서를 `SYSTEM_TECH.md`의 초안으로 이식 (압축본으로)
3. `PLAN.md`는 §19의 Week 1 마일스톤만 체크박스화
4. `CLAUDE.md`는 V1의 §1.1 절대 금지 + §6 윤리 + 본 §20 추가
5. Day 1부터 §19 순서대로 실행

### 22.3 한 줄 마무리

> **V1은 인프라 함정에 빠져 멈췄다. V2는 1주차에 PNG 1장을 끝까지 통과시키는 것이 유일한 게이트다. 그 외 모든 욕심은 Phase 2로 미룬다.**

---

*작성: 2026-05-24 · 본 문서는 V2 재설계 Draft. 사용자 승인 후 새 폴더의 `SYSTEM_TECH.md`로 압축 이식 예정.*
*근거: `docs/V1_CURRENT_STATE_REVIEW.md` + 사용자 첨부 v1.2 잠금본 + 외부 사용자 지시*
*충돌 해결 우선순위: 외부 사용자 지시 > v1.2 잠금본 > V1 리뷰*
