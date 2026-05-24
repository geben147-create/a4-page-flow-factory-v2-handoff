# A4 Page Flow Factory — v1.3 통합 잠금본 (Redesign)

> **상태**: 설계 잠금본 (Locked)
> **버전**: v1.3 (v1.3 합의안 §1~§33 전체 통합본)
> **위치**: 새 폴더 `a4-page-flow-factory-v2`의 **spec / PRD / plan** 역할을 모두 한다.
> **선행본**: v1.0 → v1.1 → v1.2 → V2 재설계 논의 → 본 v1.3
> **작성 목적**: V1 진단 + v1.2 잠금본 + V2 재설계 draft + v1.3 합의안(§1~§33)을 단일 잠금본으로 통합. 새 폴더에서 처음부터 구현하기 위한 단일 출처(Single Source of Truth).
> **출처 원칙**: 본 문서의 모든 내용은 V1 진단 / v1.2 잠금본 / V2 draft / v1.3 합의안 4개 출처 중 하나로 추적 가능. 새 아이디어 추가 금지.

---

## 0. 문서 정보

| 항목 | 내용 |
|---|---|
| 문서명 | `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` |
| 역할 | 새 폴더 v2의 spec + PRD + plan 통합 |
| 우선순위 | `SYSTEM_TECH.md`(=본 문서 압축본) > `PLAN.md` > `CLAUDE.md` |
| 코드 포함 | 없음 (YAML/JSON 스키마 예시만 허용) |
| 작업 모드 | 설계 문서 1개만. 코드/파일/커밋/패키지 설치 금지 |
| 출처 우선순위 | 사용자 외부 지시 > v1.3 합의안 > v1.2 잠금본 > V2 draft > V1 진단 |
| 변경 이력 | git tag로 관리 (자체 변경이력 시스템 없음 — v1.3 §18) |

---

## 1. 큰 원칙 (확정) — v1.3 §3

| # | 원칙 |
|---|---|
| 1 | 반자동 프롬프트 우선 |
| 2 | 외부 GPT / Claude / Gemini 웹에 복사해서 사용하는 흐름이 MVP 1순위 |
| 3 | HTML/CSS → Playwright PNG로 최종 산출물 렌더 |
| 4 | 이미지 생성 AI로 글자 박힌 카드뉴스를 만들지 않음 |
| 5 | 이미지 생성 AI는 배경 / 일러스트 / 아이콘 슬롯용으로만 검토 |
| 6 | 자동게시 / OAuth / DB / 이미지 API 자동 생성은 Post-MVP |
| 7 | 수치 단정 금지 |
| 8 | "30% 향상" 같은 검증되지 않은 표현 금지 |
| 9 | 진짜 파인튜닝 도입 안 함 |
| 10 | 벡터 DB는 당분간 도입 안 함 |
| 11 | MVP는 작게 유지 |
| 12 | **Day 5까지 instagram_portrait PNG 1장 생성이 단일 게이트** |
| 13 | Single Source of Truth — 본 문서가 충돌 시 기준 (v1.2 §1) |
| 14 | 학습 자료실(content-learning-library)이 도메인 SSOT (v1.2 §6) |

---

## 2. Prompt Layer / Prompt Pack — v1.3 §4

### 2.1 Prompt Layer (내부 개념)

시스템 전체에서 프롬프트를 **1급 산출물(first-class artifact)** 로 다루는 설계 축.

체계적으로 만드는 산출물:
- 추출 프롬프트 (P1)
- 분류 프롬프트 (P2)
- 스타일 프롬프트 (P3)
- 이미지 요소 프롬프트 (P4)
- 채널 변환 프롬프트 (P5)
- HTML/CSS 렌더용 manifest
- 최종 PNG

### 2.2 Prompt Pack (내부 개념)

프로젝트별 프롬프트 저장 묶음. **export / import 단위로만** 사용.

### 2.3 사용자 UI 용어 잠금 — v1.3 §4.2

| 내부 용어 (코드/문서) | 사용자 화면 용어 |
|---|---|
| Prompt Pack | 프로젝트 |
| Prompt Layer | 프롬프트 |
| text block | 블록 |
| design block | 디자인 블록 |
| image slot | 이미지 슬롯 |
| channel adapter | 채널 |

**핵심 규칙**:
- 사용자 UI에 **"Pack" 단어 노출 금지**
- Pack은 **내부 저장 단위 + export/import 단위로만** 사용
- 사용자 화면 단어는 **프로젝트 / 프롬프트 / 블록 / 디자인 블록 / 이미지 슬롯 / 채널** 6가지로 제한

---

## 3. P1~P5 프롬프트 구조 — v1.3 §5

| Layer | 이름 | 역할 | 단위 | MVP |
|---|---|---|---|---|
| **P1** | P1_EXTRACT | 이미지/PDF/문서에서 텍스트 추출용 프롬프트 생성 | 추출 배치 1개 | ✅ |
| **P2** | P2_CLASSIFY | 추출 텍스트를 블록으로 분류 | 추출 배치 1개 | ✅ |
| **P3** | P3_STYLE | UI/UX 스타일 프롬프트 생성 | 프로젝트 1개 | ✅ |
| **P4** | P4_IMAGE_ELEM | 이미지 요소별 프롬프트 생성 | 이미지 슬롯 1개 (N개) | ✅ |
| **P5** | P5_CHANNEL | 채널 변환 프롬프트 생성 | 채널 1개 (N개) | ✅ |

**MVP 핵심 잠금**: 각 Layer는 **API 호출이 아니라 `.md` 프롬프트 파일을 출력**.

---

## 4. 3층 범위 구조 (scope) — v1.3 §6

### 4.1 3개 scope만 사용

| scope | 의미 |
|---|---|
| `global` | 프로젝트 전체에 적용 |
| `block` | 특정 텍스트 / 디자인 블록에 적용 |
| `asset` | 특정 이미지 슬롯 / 시각 요소에 적용 |

### 4.2 폴더는 평평하게 유지

- ❌ P1~P5 × 3층 매트릭스 폴더 **만들지 않는다**
- ✅ 폴더는 평평하게 유지
- ✅ 각 `.md` 파일의 frontmatter `scope` 필드로 처리

### 4.3 Frontmatter 원본 예시 (v1.3 §6)

```yaml
---
prompt_type: P3_STYLE
scope: block
applies_to: problem
model_hint: gpt
---
```

### 4.4 frontmatter 필수 필드 (v1.3 §6 + V2 draft §8.3 통합)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `prompt_type` | enum | ✅ | P1_EXTRACT / P2_CLASSIFY / P3_STYLE / P4_IMAGE_ELEM / P5_CHANNEL |
| `scope` | enum | ✅ | global / block / asset |
| `applies_to` | string | | scope가 block이면 block_type, asset이면 slot_id |
| `model_hint` | enum | | gpt / claude / gemini / any |
| `prompt_version` | string | ✅ | 예: v1 |
| `generated_by` | string | ✅ | 예: factoryv2 0.1.0 |
| `generated_at` | ISO8601 | ✅ | |
| `deterministic_hash` | sha256 | ✅ | 재현성 검증 (`generated_at` 제외 비교) |
| `inputs` | list[path] | ✅ | 인용한 자료실 파일 경로 |
| `project_slug` | string | ✅ (생성 프롬프트) | projects/[slug]에 적재 시 |
| `batch_id` | string | P1/P2만 | |
| `slot_id` | string | P4만 | |

---

## 5. V2 제품 정의 (PRD)

### 5.1 제품 메타

| 항목 | 내용 |
|---|---|
| 프로젝트명 | `a4-page-flow-factory-v2` (V1 폴더와 물리적으로 분리) |
| 한 줄 정의 | 외부 LLM에 붙여넣을 **반자동 프롬프트 P1~P5**를 생성하고, 사람이 검수한 카피를 받아 **HTML/CSS → Playwright PNG**로 9개 채널 결과물을 만드는 로컬 CLI |
| 형태 | 로컬 CLI (Mode A 소형 MVP) |
| 사용자 | 1인 사업자 / 브랜드 운영자 (한국어) |
| 비용 모델 | MVP는 **무료** (외부 LLM은 사용자의 무료/유료 계정) |
| 최종 산출물 | `.png` (10개 output_profile 중 선택, MVP는 instagram_portrait) + `.html` + `.json` manifest |

### 5.2 핵심 목표 (Goals)

| ID | 목표 | 측정 방법 |
|---|---|---|
| G1 | 외부 LLM 사용 → 프롬프트 1개 출력 → 결과 적재 → PNG 1장까지 **15분 이내** | 사용자 실측 |
| G2 | 같은 입력 → 같은 프롬프트 .md 바이트 동일 (deterministic) | `test_prompts_deterministic.py` |
| G3 | `instagram_portrait` 1개 profile에서 PNG 1장 + 품질 체크 **12/12** 통과 | 자동 검증 |
| G4 | 외부 LLM 변경(GPT↔Claude↔Gemini)에 CLI 변경 0줄 (모델 토글로 흡수) | 코드 의존성 grep |
| G5 | MVP 종료까지 LLM SDK 의존성 0개 | `pyproject.toml` 검증 |

### 5.3 비목표 (Non-Goals)

| ⛔ 항목 | 출처 |
|---|---|
| 자동게시 / OAuth / Instagram·Threads·X API | v1.3 §3-6 |
| DB / Supabase / 로그인 / 결제 / 다중 사용자 | v1.2 §1, v1.3 §33 |
| 이미지 생성 AI로 글자 박힌 카드뉴스 | v1.3 §3-4 |
| 다국어 지원 (한국어 우선) | v1.2 §2 |
| 모바일 / 클라우드 / CI 배포 | v1.2 §2 |
| 진짜 파인튜닝 | v1.3 §3-9 |
| 벡터 DB 즉시 도입 | v1.3 §3-10 |
| 윤리·유사도·워터마크 하드게이트 | v1.2 §31 |
| `quality_score` / 날짜형 ID | v1.2 §25 |
| 수치 단정 ("30% 향상" 등) | v1.3 §3-7, §3-8 |
| 자체 변경 이력 시스템 | v1.3 §18 |
| Framer식 풀 편집 UI | v1.3 §32 |

---

## 6. V2 사용자 흐름 (Happy Path)

```
1. factory init                          → 자료실 + projects/_example 시드 생성
2. (사용자) brand/voice/channels 편집      → 자료실 첫 채움
3. factory extract prompt --batch b001    → P1 추출 프롬프트 .md 생성
4. (사용자) 외부 LLM에 P1 붙여넣고 결과 회수
5. factory extract import --batch b001    → 회수 → extracted.md + review.md
6. (사용자) review.md 누락/중복 검수 후 승인
7. factory blocks add ...                 → blocks.jsonl 1줄씩 사람이 추가
8. factory design-blocks add ...          → design_blocks.jsonl 1줄씩 추가
9. factory project new <slug>             → projects/<slug>/prompts/ 생성
10. factory generate --project <slug>     → P3 + P4 + P5 프롬프트 .md 생성
11.(사용자) 외부 LLM에 붙여넣고 카피 회수
12.factory copy import --project <slug>   → copy_master.json 적재
13.factory sample --project <slug>        → page_manifest.json + page_NNN.html
14.factory render --project <slug>        → page_NNN.png + quality_report.json (12 항목)
15.(사용자) PNG 확인 후 사용
```

부분 재실행:
- 카피만 다시: 12번부터
- 렌더만 다시: 14번부터
- 채널만 다른 출력 추가: 10번부터 다른 channel adapter 선택

---

## 7. 최종 폴더 구조 — v1.3 §7

### 7.1 content-learning-library — v1.3 §7.1

```
content-learning-library/
├── SKILL.md
├── README.md
├── system/
│   ├── schema_blocks.md
│   ├── naming_rules.md
│   └── retrieval_rules.md
├── brand/
│   ├── brand_identity.md
│   ├── business_summary.md
│   ├── strengths.md
│   ├── proof_assets.md
│   ├── forbidden_words.md
│   └── anti_patterns.md
├── voice/
│   ├── expert_saas_tone.md
│   ├── friendly_sns_tone.md
│   ├── luxury_brand_tone.md
│   └── user_favorite_phrases.md
├── channels/
│   ├── threads.md
│   ├── instagram_cardnews.md
│   ├── instagram_story.md
│   ├── instagram_post.md
│   ├── linkedin_post.md
│   ├── medium_article.md
│   ├── pinterest_pin.md
│   ├── x_thread.md
│   ├── facebook_post.md
│   ├── a4_report.md
│   ├── email.md
│   └── landing_page.md
├── logic/
│   ├── problem_agitation_solution.md
│   ├── lead_capture.md
│   ├── conversion_boost.md
│   ├── before_after.md
│   └── comparison.md
├── products/
│   └── [product_slug]/
│       ├── meta.yml
│       ├── original_extracted.md
│       ├── product_summary.md
│       └── source_images/
├── blocks/
│   ├── blocks.jsonl              ← 텍스트 블록 단일 원본
│   └── design_blocks.jsonl       ← 디자인 블록 단일 원본 (v1.3 신규)
├── benchmarks/
│   ├── landing_pages/
│   ├── cardnews/
│   ├── threads/
│   └── emails/
├── recipes/
│   └── recipes.md
├── prompts/                       ← 자료실의 프롬프트 템플릿 (default)
│   ├── extract_default.md
│   ├── classify_default.md
│   ├── style_minimal.md
│   ├── style_luxury.md
│   ├── style_friendly.md
│   ├── image_elem_default.md
│   └── channel_adapters/
│       ├── instagram_cardnews.md
│       ├── linkedin_post.md
│       ├── medium_article.md
│       ├── pinterest_pin.md
│       ├── x_thread.md
│       ├── facebook_post.md
│       └── threads.md
└── examples/
    ├── trusta_cardnews_good_01.md
    ├── trusta_threads_good_01.md
    ├── trusta_landing_good_01.md
    └── trusta_email_good_01.md
```

### 7.2 projects/ 구조 — v1.3 §7.2

```
projects/[project_slug]/
└── prompts/
    ├── P1_extract.md
    ├── P2_classify.md
    ├── P3_style.md
    ├── P4_image_elem/
    │   ├── page_01_icon_01.md
    │   ├── page_01_illust_01.md
    │   └── page_02_icon_01.md
    └── P5_channel/
        ├── instagram_cardnews.md
        ├── linkedin_post.md
        └── ...
```

원칙:
- 자료실의 `prompts/*.md`는 **default 템플릿** (재사용)
- 프로젝트의 `projects/[slug]/prompts/`는 **인스턴스화된 P1~P5** (생성물)
- 폴더는 평평하게 유지하되, P4는 slot 수가 많아 하위 폴더 허용 (v1.3 §7.2)

### 7.3 전체 v2 루트 통합

```
a4-page-flow-factory-v2/
├── pyproject.toml / uv.lock / .python-version / .gitignore / README.md
├── SYSTEM_TECH.md / PLAN.md / CLAUDE.md   ← 메타 문서 3개 잠금 (4번째 금지)
├── docs/A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md (본 문서)
├── docs/lessons/L001~L003.md              ← V1 학습 즉시 적용
├── src/factory/                            ← v1.3 §19 모듈 구조
├── tests/
├── content-learning-library/               ← 자료실 (§7.1)
├── projects/[slug]/prompts/                ← 생성 프롬프트 (§7.2)
├── extract/input|output/[batch_id]/        ← .gitignore
└── output/[project_slug]/                  ← 렌더 결과, .gitignore
```

---

## 8. blocks.jsonl 스키마 — v1.3 §8

**형식**: JSONL (한 줄에 한 객체. JSON 배열 아님)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | 짧은 랜덤 고유값 |
| `block_type` | enum | hook / problem / pain_point / solution / benefit / proof / before_after / objection / cta / trust / feature / comparison |
| `text` | string | 블록 본문 |
| `source_type` | enum | own / benchmark / user_added |
| `source_product` | string | 제품 slug |
| `source_file` | string | 파일 경로 |
| `source_anchor` | string | 원문 내부 위치 |
| `industry` | string | snake_case |
| `tone` | enum | voice/ 파일명과 동일 |
| `channel` | enum | channels/ 파일명과 동일 |
| `logic_role` | enum | logic/ 파일명과 동일 |
| `tags` | string[] | snake_case |
| `status` | enum | draft / approved / archived |
| `reuse_note` | string | 재사용 메모 |
| `created_at` | ISO8601 | 생성일 |
| `updated_at` | ISO8601 | 수정일 |

❌ 사용 안 함: `quality_score`, 날짜형 ID

---

## 9. design_blocks.jsonl 스키마 — v1.3 §9 (신규)

**형식**: JSONL

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | 짧은 랜덤 고유값 |
| `design_block_type` | enum | icon / illustration / hero_image / chart / divider / cta_button / tag / card_layout / color_palette / typography |
| `source_image` | string | 레퍼런스 이미지 경로 |
| `crop_box` | object | `{x, y, w, h}` |
| `description` | string | 사람이 쓴 짧은 설명 |
| `style_prompt` | string | 이 블록의 스타일을 묘사하는 프롬프트 |
| `image_gen_prompt` | string | 외부 GPT / Midjourney / 이미지 생성툴용 프롬프트 |
| `tone` | enum | voice/와 동일 |
| `palette` | string[] | hex 색상 코드 |
| `tags` | string[] | snake_case |
| `status` | enum | draft / approved / archived |
| `linked_text_block` | string | 짝이 되는 텍스트 블록 id. 선택 |
| `created_at` | ISO8601 | 생성일 |
| `updated_at` | ISO8601 | 수정일 |

**원칙**:
- 텍스트 블록과 디자인 블록은 **분리된 JSONL**
- `linked_text_block`로 1:1/N:M 매칭 가능
- 디자인 블록도 사람 검수 후 추가 (자동 적재 금지)

---

## 10. image_slots 스키마 — v1.3 §10 (신규)

`page_manifest.json`에 포함되는 이미지 슬롯 구조.

| 필드 | 타입 | 설명 |
|---|---|---|
| `slot_id` | string | 슬롯 고유 ID |
| `elem_type` | enum | icon / illustration / hero_image / chart_visual / pattern / divider |
| `position` | object | `{x, y, w, h}`. 0~1 비율 |
| `linked_design_block` | string | design_blocks의 id. 선택 |
| `prompt_file` | string | P4 프롬프트 .md 경로 |
| `generated_image` | string\|null | 사용자가 업로드한 이미지 경로 |
| `status` | enum | prompt_only / pending / ready / approved |
| `required` | boolean | 필수 / 선택 구분 |

**원칙** (v1.3 §10):
- 사용자가 외부 GPT / Gemini / 이미지 생성툴에서 만든 이미지를 **drag&drop**으로 슬롯에 배치
- 이미지 API 자동 생성은 **MVP 제외**
- 슬롯별 프롬프트를 표시
- 슬롯은 `required` 여부를 가진다

---

## 11. 채널 구조 — v1.3 §11

채널은 **common_source + channel_profiles** 방식으로 설계.

### 11.1 본질 4개 (v1.3 §11.1)

| 본질 | 설명 |
|---|---|
| `long_form_text` | 긴 글 중심 |
| `short_form_text` | 짧은 글 중심 |
| `card_visual` | 카드뉴스 / 이미지 중심 |
| `mixed` | 텍스트와 이미지 혼합 |

### 11.2 플랫폼 어댑터 9개 (v1.3 §11.2)

| adapter | 본질 |
|---|---|
| `instagram_cardnews` | card_visual |
| `instagram_story` | card_visual |
| `instagram_post` | mixed |
| `linkedin_post` | mixed |
| `medium_article` | long_form_text |
| `pinterest_pin` | card_visual |
| `x_thread` | short_form_text |
| `facebook_post` | mixed |
| `threads` | short_form_text |

**원칙**:
- 9개 채널을 **완전히 따로 만들지 않는다**
- 본질 4개 + 플랫폼 어댑터 구조로 만든다
- 채널 변환은 별도 단계가 아니라 **미리보기 화면의 탭**으로 제공 (§14.4)

> 자료실 `channels/` 폴더에는 위 9개 + `a4_report.md` + `email.md` + `landing_page.md`까지 총 12개 파일 존재 (§7.1).

---

## 12. output_profile 10개 규격 — v1.3 §12

| profile | width | height | 용도 |
|---|---|---|---|
| `a4_document` | 1240 | 1754 | A4 세로 |
| `instagram_square` | 1080 | 1080 | 인스타 정사각 |
| `instagram_portrait` | 1080 | 1350 | 인스타 4:5 카드뉴스 |
| `story_9_16` | 1080 | 1920 | 스토리 / 릴스 |
| `threads_image` | 1080 | 1350 | Threads |
| `linkedin_post` | 1080 | 1080 | LinkedIn |
| `pinterest_pin` | 1000 | 1500 | Pinterest 2:3 |
| `x_card` | 1200 | 675 | X 이미지 카드 |
| `facebook_post` | 1200 | 630 | Facebook |
| `medium_header` | 1200 | 675 | Medium 헤더 |

### MVP 우선순위

1. **instagram_portrait** (Day 5 단일 게이트)
2. a4_document
3. instagram_square
4. story_9_16
5. threads_image
6. 나머지 profile (Phase 2 이후)

---

## 13. 워크플로우 9단계 — v1.3 §13

| 단계 | 설명 | MVP |
|---|---|---|
| 1. 자료 입력 | 이미지 / PDF / 문서 / 텍스트 입력 | 포함 |
| 2. 추출 확인 UI6 | 추출 텍스트와 추출 프롬프트 확인 (4-pane) | 포함 |
| 3. 블록 분류 | 텍스트 블록 / 디자인 블록 분리 | 포함 |
| 4. 자료실 저장 | blocks / design_blocks 저장 | 포함 |
| 5. 프롬프트 설계 | P1~P5 프롬프트 편집 (독립 단계) | 포함 |
| 6. 레시피 선택 | 채널 / 목적 / 톤 선택 | 포함 |
| 7. 미리보기 | 채널 탭 / 이미지 슬롯 탭 | 포함 |
| 8. 품질 검사 | PNG 품질 체크 (12 항목) | 포함 |
| 9. 최종 산출 | HTML / PNG / prompt files 출력 | 포함 |

---

## 14. UI 설계 핵심 화면 — v1.3 §14

### 14.1 UI6 추출 화면 (4-pane) — v1.3 §14.1

| 영역 | 역할 |
|---|---|
| 페이지 썸네일 | 입력 이미지 / 문서 페이지 목록 |
| 추출된 텍스트 | 외부 LLM 또는 OCR로 추출된 텍스트 확인 |
| 추출 프롬프트 | P1_EXTRACT 프롬프트 확인 / 복사 |
| 추출 품질 | 누락 / 중복 / 작은 글씨 / 표 검수 |

**모바일**: 4-pane 대신 **탭 전환 구조**

**추출 프롬프트 패널 요소**:
- 모델 토글: Claude / GPT / Gemini
- 선택한 모델용 프롬프트로 복사
- 프롬프트 복사
- 기본 템플릿으로 복원

### 14.2 블록 분류 화면 — v1.3 §14.2

**탭**:
- 텍스트 블록
- 디자인 블록

**텍스트 블록 타입 12개**:
hook / problem / pain_point / solution / benefit / proof / before_after / objection / cta / trust / feature / comparison

**디자인 블록 타입 10개**:
icon / illustration / hero_image / chart / divider / cta_button / tag / card_layout / color_palette / typography

### 14.3 프롬프트 설계 화면 — v1.3 §14.3

- P1~P5 파일을 **평평하게 나열**
- 파일 클릭 시 편집
- 모델별 토글 제공
- 사용자 화면 용어는 "프로젝트 / 프롬프트 / 블록 / 이미지 슬롯 / 채널"만 사용

### 14.4 미리보기 화면 — v1.3 §14.4

**탭**: instagram_cardnews / linkedin_post / medium_article / pinterest_pin / x_thread / facebook_post / threads / **이미지 슬롯**

**원칙**:
- 채널 변환은 별도 단계가 아님
- 미리보기 화면 안에서 탭으로 제공

### 14.5 이미지 슬롯 매니저 — v1.3 §14.5

기능:
- drag & drop 업로드
- 슬롯별 프롬프트 표시
- required 슬롯 표시
- generated_image 연결
- 상태 표시: `prompt_only / pending / ready / approved`

> ⚠️ 본 UI 설계는 향후 GUI 도입 시 적용. **MVP CLI 단계에서는 CLI 인자/파일 출력으로 동등한 기능 제공** (예: `factory ui6 view`는 CLI에서 마크다운 리포트로 4-pane 정보를 한꺼번에 출력).

---

## 15. 모델별 프롬프트 토글 — v1.3 §15

**MVP 포함**.

| 모델 | 프롬프트 형식 |
|---|---|
| GPT | Markdown 구조 |
| Claude | XML 태그 구조 |
| Gemini | 예시 기반 구조 |

**적용 화면**:
- UI6 추출 화면 (§14.1)
- 프롬프트 설계 화면 (§14.3)

**CLI 매핑**: `factory generate --project <s> --model gpt|claude|gemini` → 동일 입력에 형식만 다른 P1~P5 출력. 모델 변경 시 CLI 코드 0줄 변경(G4).

---

## 16. 룰 기반 자동 초안 — v1.3 §16

**MVP 포함**. `block_type`에 따라 기본 스타일 / 이미지 요소 / 채널 프롬프트 초안을 **자동 생성**.

| block_type | 자동 초안 방향 |
|---|---|
| `problem` | 경고형 카드, 리스크 배지, 고민하는 실무자 아이콘 |
| `proof` | 숫자 / 근거 카드, 체크 배지, 표 / 차트 요소 |
| `solution` | 정리된 SaaS 대시보드, 안심 톤 |
| `cta` | 버튼, 짧은 행동 문구, 명확한 전환 영역 |
| `benefit` | Before/After 대비, 가치 요약 |
| `trust` | 검증 / 인증 / 근거 중심 |
| `comparison` | 좌우 비교표, 선택 기준 강조 |

> 나머지 block_type(hook/pain_point/before_after/objection/feature)은 디폴트 채택. v1.3에 명시된 7개 매핑만 잠금.

---

## 17. 재사용 UI — v1.3 §17

**사용자 화면 용어**: "다른 프로젝트에서 가져오기"

**체크박스**:
- 디자인 블록
- 이미지 요소 프롬프트
- 채널 설정
- 전체 프롬프트

**내부**: Pack import/export. UI에는 "Pack" 단어 노출 금지.

**CLI 매핑**: `factory project import --from <other_slug> [--include design_blocks|image_prompts|channel|all]`

---

## 18. 변경 이력 — v1.3 §18

**원칙**:
- 자체 변경 이력 시스템을 만들지 않는다
- git에 위임한다
- 필요 시 `.bak` 파일 정도만 허용

**금지**:
- 자체 버전관리 DB
- 자체 히스토리 UI
- 복잡한 변경 추적 시스템

---

## 19. 개발 모듈 구조 — v1.3 §19

```
src/factory/
├── prompts/
│   ├── base.py
│   ├── p1_extract.py
│   ├── p2_classify.py
│   ├── p3_style.py
│   ├── p4_image_elem.py
│   └── p5_channel.py
├── ai/
│   ├── adapter_base.py
│   ├── adapter_openai.py
│   ├── adapter_anthropic.py
│   ├── adapter_google.py
│   └── cost_guard.py
├── extract/
│   ├── ocr_only.py
│   ├── vision_only.py
│   └── prompt_builder.py
└── render/
    ├── html_builder.py
    ├── browser.py
    ├── screenshot.py
    └── quality_check.py
```

**원칙** (v1.3 §19):
- `src/factory/prompts/`는 **MVP 구현 대상**
- `src/factory/ai/`는 **Phase 2 placeholder**
- MVP에서는 실제 AI API 호출을 구현하지 않는다
- `extract/`는 반자동 P1 프롬프트 흐름을 우선
- `render/`는 Playwright PNG 출력 흐름 담당

### 19.1 추가 모듈 (V2 draft + V1 자산 이식 포함)

| 추가 모듈 | 역할 |
|---|---|
| `src/factory/schema/{enums,blocks,design_blocks,manifest,meta}.py` | 단일 enum + 스키마 |
| `src/factory/utils/{cache,ascii_safe,encoding}.py` | V1 cache 이식 + L1/L2/L3 |
| `src/factory/library/{loader,seed,routing}.py` | 자료실 r/w + SKILL 라우팅 |
| `src/factory/cli/*.py` | Typer 명령 그룹 |

---

## 20. Playwright 렌더링 규칙 — v1.3 §20

| 규칙 | 내용 |
|---|---|
| device_scale_factor | 2 이상 |
| viewport | output_profile width / height와 일치 |
| full_page | false 기본 |
| 폰트 대기 | `document.fonts.ready` |
| 네트워크 대기 | `networkidle` 또는 `domcontentloaded` 후 안정화 |
| 한글 검사 | 한글 폰트 깨짐 확인 |
| PNG 검사 | 파일 크기 / 해상도 / 빈 화면 검증 |
| 파일명 | `page_001.png`, `page_002.png` 순차 저장 |
| 대응 | HTML과 PNG 1:1 대응 |

---

## 21. PNG 품질 체크리스트 — v1.3 §21 (12 항목 잠금)

> **주의** (v1.3 §21): V2 draft에서 11/11로 표현된 경우가 있으나, **v1.3 합의안의 체크리스트는 12개 항목**. 본 문서는 **12개로 잠근다**.

| # | 항목 |
|---|---|
| 1 | 생성 성공 |
| 2 | 해상도 일치 |
| 3 | 파일 용량 0 아님 |
| 4 | 빈 화면 아님 |
| 5 | 한글 텍스트 렌더링 확인 |
| 6 | safe area 침범 없음 |
| 7 | 제목 / 본문 / CTA 잘림 없음 |
| 8 | 페이지 번호 정상 |
| 9 | 이미지 슬롯 깨짐 없음 |
| 10 | 폰트 fallback 과다 사용 없음 |
| 11 | output_profile 규격 준수 |
| 12 | **HTML 파일과 PNG 파일 개수 일치** |

→ `quality_report.json`에 12항목 결과 기록.

---

## 22. MVP 포함 범위 — v1.3 §22

| 항목 | 포함 여부 |
|---|---|
| Prompt Layer / Prompt Pack 내부 구조 | 포함 |
| P1~P5 프롬프트 파일 | 포함 |
| frontmatter scope | 포함 |
| `design_blocks.jsonl` | 포함 |
| image_slots + required 필드 | 포함 |
| 모델별 토글 Claude/GPT/Gemini | 포함 |
| 룰 기반 자동 초안 | 포함 |
| 이미지 슬롯 매니저 drag&drop | 포함 (CLI는 파일 배치 기반) |
| 채널 프로파일 구조 | 포함 |
| instagram_portrait 우선 구현 | 포함 |
| 프롬프트 설계 독립 단계 | 포함 |
| HTML/CSS → Playwright PNG 렌더 | 포함 |
| `src/factory/prompts/` 모듈 골격 | 포함 |
| PNG 품질 체크리스트 자동화 (12 항목) | 포함 |
| cache.py / slicer.py / main.py 선별 이식 | 포함 |

---

## 23. MVP 제외 / Post-MVP — v1.3 §23

| 항목 | 처리 |
|---|---|
| 이미지 API 자동 생성 | Post-MVP |
| SNS 자동게시 Threads / IG / X | Post-MVP |
| OAuth | Post-MVP |
| DB / Supabase | Post-MVP |
| 자동 삽입 고도화 | Post-MVP |
| 자체 변경이력 시스템 | **제외** |
| 대량 자동 생성 | Post-MVP |
| 3초 / 6초 자동 업로드 | Post-MVP |
| 벡터 DB | 당분간 제외 |
| 진짜 파인튜닝 | **제외** |
| Telegram 승인 알림 | Post-MVP |
| 캘린더 자동 배치 | Post-MVP |
| `src/factory/ai/` 실제 구현 | Phase 2 |
| Framer식 풀 편집 UI | **제외** |

---

## 24. CLI 마이그레이션 결정 — v1.3 §24

| 항목 | 결정 |
|---|---|
| 기존 V1 폴더 | **유지**. archive / git tag로 마킹 |
| 새 폴더 | `a4-page-flow-factory-v2` |
| 개발 방식 | 기존 V1 패치가 아니라 **새 프로젝트** |
| v1.2 코드 | **직접 계승하지 않음** |
| 재사용 코드 | `cache.py` / `slicer.py` / `main.py` **선별 복사** |
| 첫 커밋 | 재사용 코드와 기본 scaffold 흡수 |
| `anthropic` 의존성 | MVP **제거** |
| `block_types.yaml` | **폐기** |
| `enums.py` | **단일 enum 정의로 신규 작성** |

---

## 25. MVP 단일 게이트 — v1.3 §25

MVP의 단일 성공 게이트:

```
factory init
→ factory generate
→ factory copy import
→ factory sample
→ factory render
→ instagram_portrait PNG 1장 생성
→ PNG 품질 체크 (12 항목) 통과
```

### 최소 성공 조건

| 조건 | 기준 |
|---|---|
| 새 프로젝트 폴더 생성 | `a4-page-flow-factory-v2` |
| CLI 실행 | help 출력 (cp949 안전) |
| init | 기본 폴더 / 시드 생성 |
| generate | P1~P5 프롬프트 파일 생성 |
| copy import | 외부 LLM 결과를 내부 구조로 import |
| sample | instagram_portrait HTML 1장 생성 |
| render | Playwright PNG 1장 생성 |
| quality_check | **12개 체크 통과** |

---

## 26. 1주차 5일 구현 계획 — v1.3 §26

| Day | 작업 | 완료 조건 |
|---|---|---|
| **Day 1** | 셋업 + V1 자산 이식 | 새 폴더, pyproject, CLI help, lint 통과 |
| **Day 2** | enums / blocks schema + 자료실 시드 | `blocks.jsonl` / `design_blocks.jsonl` 검증 |
| **Day 3** | P3 + P5 프롬프트 + 골든 픽스처 | P3/P5 산출물 생성 및 fixture 통과 |
| **Day 4** | copy import + sample HTML | imported copy로 instagram_portrait HTML 1장 생성 |
| **Day 5** | Playwright PNG 1장 + E2E | PNG 1장 생성 + 품질 체크 12/12 통과 |

### 26.1 Week 1 종료 게이트

- [ ] `factory init → generate → copy import → sample → render` 1회 통과 (PNG 1장)
- [ ] PNG 품질 체크리스트 12/12
- [ ] `pyproject.toml`에 `anthropic` 미설치
- [ ] L1/L2/L3 코드 반영
- [ ] PLAN.md / SYSTEM_TECH.md 인수인계 메모

---

## 27. 위험 신호 — v1.3 §27

| 위험 신호 | 의미 | 대응 |
|---|---|---|
| Day 5까지 PNG 1장 미생성 | 인프라 함정 재발 | 범위 축소 |
| `anthropic` 의존성 추가 | MVP 기본 가정 위반 | 제거하고 prompt-only로 복귀 |
| 4번째 메타 문서 생성 | 문서 폭발 | 3개 문서 잠금으로 복귀 |
| output_profile을 한 번에 10개 구현 | 범위 과대 | instagram_portrait만 남김 |
| 이미지 API를 MVP에 추가 | 비용/복잡도 증가 | Post-MVP로 이동 |
| DB / Supabase 도입 | MVP 범위 초과 | 로컬 파일 기반 유지 |
| 자체 변경 이력 구현 | 불필요한 시스템화 | git 위임 |

---

## 28. 난이도 / 예상 시간 / 선행조건 — v1.3 §28

| 작업 | 난이도 | 예상 시간 | 선행조건 |
|---|---|---|---|
| 새 폴더 셋업 | 낮음 | 0.5일 | 없음 |
| V1 자산 선별 이식 | 낮음 | 0.5일 | 새 폴더 |
| `enums.py` 작성 | 낮음 | 0.5일 | 스키마 결정 |
| `blocks.jsonl` 검증 | 낮음 | 0.5일 | enums.py |
| `design_blocks.jsonl` 검증 | 중간 | 0.5일 | enums.py |
| P1~P5 프롬프트 생성 | 중간 | 1일 | prompts 구조 |
| 모델별 토글 | 중간 | 0.5일 | P1/P3/P5 |
| 룰 기반 자동 초안 | 중간 | 1일 | block_type enum |
| copy import | 중간 | 1일 | P1/P2 구조 |
| sample HTML | 중간 | 1일 | imported copy |
| Playwright render | 중간 | 1일 | sample HTML |
| PNG 품질 체크 (12) | 중간 | 0.5일 | render |
| 이미지 슬롯 매니저 | 중간~높음 | 1~2일 | image_slots schema |
| 채널 adapter 확장 | 중간 | 1~2일 | instagram_portrait 성공 |
| 실제 AI API 호출 | 높음 | 3일+ | Phase 2 |

---

## 29. 같은 세션 / 다른 세션 — v1.3 §29

| 작업 | 세션 |
|---|---|
| 새 폴더 셋업 + pyproject + CLI help | **같은 세션** |
| V1 자산 이식 | **같은 세션** |
| lint / 기본 테스트 | **같은 세션** |
| `enums.py` + schema validator | **별도 세션** |
| `blocks.jsonl` / `design_blocks.jsonl` 시드 | **별도 세션** |
| P1~P5 프롬프트 생성 | **별도 세션** |
| copy import | **별도 세션** |
| sample HTML | **별도 세션** |
| Playwright PNG | **별도 세션** |
| 이미지 슬롯 매니저 | **별도 세션** |
| 채널 adapter 확장 | **별도 세션** |
| AI adapter 실제 구현 | **Phase 2 별도 세션** |

---

## 30. 비용 발생 / 비용 없음 — v1.3 §30

| 작업 | 비용 |
|---|---|
| 로컬 CLI 셋업 | **없음** |
| P1~P5 프롬프트 파일 생성 | **없음** |
| 외부 GPT/Claude/Gemini 웹 복사 사용 | 사용 계정 비용 범위 |
| HTML/CSS 생성 | **없음** |
| Playwright PNG 렌더 | **없음** |
| OCR Tesseract | 거의 없음 |
| 이미지 슬롯 프롬프트 생성 | **없음** |
| 이미지 생성툴 사용 | 사용 툴별 비용 |
| 텍스트 API 자동 호출 | Phase 2 비용 발생 |
| Vision API 자동 호출 | Phase 2 비용 발생 |
| 이미지 API 자동 생성 | Post-MVP 비용 큼 |
| SNS 자동게시 | API/운영비 발생 가능 |
| DB / Supabase | Post-MVP 비용 가능 |

---

## 31. 우선순위 Top 10 — v1.3 §31

| 순위 | 작업 | 이유 |
|---|---|---|
| 1 | 새 폴더 셋업 | 기존 V1 패치 방지 |
| 2 | V1 자산 선별 이식 | 검증된 유틸만 가져오기 |
| 3 | CLI help / init | 최소 실행 가능 |
| 4 | `enums.py` | 단일 정의 원본 |
| 5 | `blocks` / `design_blocks` schema | 텍스트와 디자인 블록 분리 |
| 6 | P1~P5 prompt files | v1.3 핵심 |
| 7 | model toggle | Claude/GPT/Gemini 복사 흐름 |
| 8 | copy import | 반자동 워크플로우 연결 |
| 9 | sample HTML | 산출물 가시화 |
| 10 | Playwright PNG + 품질 체크 (12) | MVP 성공 게이트 |

---

## 32. V3 보류 항목 — v1.3 §32

| 항목 | 보류 이유 |
|---|---|
| 전체 output_profile 동시 구현 | MVP 범위 과대 |
| 벡터 DB 임계값 | 실제 데이터 1,000개 이상 축적 후 판단 |
| 윤리 가드레일 재도입 | MVP 후 필요성 검토 |
| 멀티 사용자 | 로컬 단일 사용자 후 판단 |
| Supabase | 파일 기반 한계 확인 후 판단 |
| 이미지 자동 삽입 고도화 | 슬롯 기반 반자동 검증 후 판단 |
| 채널 자동게시 | 수동 산출물 품질 확인 후 판단 |
| Telegram 승인 알림 | 승인 흐름 안정화 후 판단 |
| 캘린더 자동 배치 | 운영 루틴 생긴 후 판단 |
| API 모델 호출 | prompt-only 안정화 후 판단 |
| 비용 가드 | API 도입 시점에 판단 |
| 풀 편집 UI | HTML/PNG 품질 검증 후 판단 |
| 대량 생성 | 1장 품질 게이트 통과 후 판단 |
| 템플릿 마켓 | 재사용 패턴 축적 후 판단 |
| 팀 협업 기능 | 단일 사용자 워크플로우 안정화 후 판단 |

---

## 33. 최종 하지 말아야 할 것 — v1.3 §33

1. 기존 V1 폴더를 계속 패치하지 않는다.
2. v1.2의 M2-5 흐름으로 돌아가지 않는다.
3. API 자동 호출을 MVP에 넣지 않는다.
4. `anthropic` 의존성을 MVP에 넣지 않는다.
5. DB / Supabase를 MVP에 넣지 않는다.
6. 자동게시를 MVP에 넣지 않는다.
7. 이미지 생성 AI로 글자 박힌 카드뉴스를 만들지 않는다.
8. output_profile 10개를 한 번에 구현하지 않는다.
9. 문서를 4개 이상으로 늘리지 않는다.
10. 자체 변경 이력 시스템을 만들지 않는다.
11. **Day 5 PNG 1장 게이트를 삭제하지 않는다.**
12. P1~P5를 흐릿하게 만들지 않는다.
13. **"Pack"이라는 단어를 사용자 UI에 노출하지 않는다.**

---

## 34. P1~P5 입력/출력 (참조 표)

(V2 draft §8 + v1.3 §7.2 통합)

| 코드 | 입력 (CLI 읽음) | 출력 (.md) | 외부 LLM 결과 회수 (CLI 적재) |
|---|---|---|---|
| **P1** | `extract/input/[batch]/*.png` (메타) + `brand/forbidden_words.md` + `system/naming_rules.md` + `prompts/extract_default.md` | `projects/[slug]/prompts/P1_extract.md` | `extract/[batch]/extracted.{md,json}` → `factory extract import` |
| **P2** | `extract/[batch]/extracted.md` (사람 검수 후) + enum 정의 + `prompts/classify_default.md` | `projects/[slug]/prompts/P2_classify.md` | `extract/[batch]/classified.jsonl` → `factory blocks add` (interactive) |
| **P3** | `brand/*.md` + `voice/[tone].md` + `prompts/style_*.md` | `projects/[slug]/prompts/P3_style.md` | (P5 안에 흡수) |
| **P4** | 각 image_slot 메타 + `prompts/image_elem_default.md` + (선택) `design_blocks.jsonl` 매칭 | `projects/[slug]/prompts/P4_image_elem/page_NN_<elem>_NN.md` (slot 수만큼) | (선택) 사용자가 만든 이미지를 슬롯에 drag&drop → `output/[slug]/assets/img_NNN.png` |
| **P5** | 매칭된 `blocks.jsonl` 행 + P3 결과 + `channels/[ch].md` + `prompts/channel_adapters/[ch].md` + `logic/[role].md` + `examples/*.md` | `projects/[slug]/prompts/P5_channel/[ch].md` | `output/[slug]/copy_master.json` → `factory copy import` |

---

## 35. P1~P5 검증 7개 축

(V2 draft §9 계승)

| 검증 ID | 이름 | 자동/수동 | 통과 기준 |
|---|---|---|---|
| V-STRUCT | frontmatter + 필수 섹션 | 자동 | frontmatter 필수 필드(§4.4) + 본문 5섹션(역할/맥락/입력자료/원하는 출력형식/제약) 전부 존재 |
| V-DETERM | 재현성 | 자동 | 같은 입력 2회 → 바이트 동일 (`deterministic_hash`) |
| V-REF | 참조 무결성 | 자동 | `inputs:` 경로 모두 존재 + 인용 anchor 유효 |
| V-ENUM | enum 정합성 | 자동 | tone/channel/block_type/design_block_type/scope/prompt_type/model_hint 모두 `enums.py`에 정의됨 |
| V-FORB | 금지어 미포함 | 자동 | `brand/forbidden_words.md` 라인이 프롬프트 본문에 들어가지 않음 |
| V-GOLD | 골든 diff | 자동 | `tests/golden/prompts/P*_*.md`와 차이가 `generated_at` 외 0 |
| V-MANUAL | 외부 LLM 분산 | 수동 | 같은 프롬프트 → 외부 LLM 3회 → 결과 핵심 메시지 사람 평가 일치 |

### 35.1 P1~P5 추가 검증

| 코드 | 추가 검증 |
|---|---|
| P1 | extract input batch에 이미지 1장 이상 |
| P2 | extracted.md가 사람 승인(review.md APPROVED) 후에만 빌드 가능 |
| P3 | recipe.tone이 `voice/` 파일과 1:1 대응 |
| P4 | image_slot의 `required` 필드와 channel의 권장 슬롯 수 일치 |
| P5 | channel + output_profile + tone + adapter(`prompts/channel_adapters/`) 4중 정합성 |

---

## 36. V2 CLI 명령어 (잠금)

(V2 draft §5 + v1.3 §13 워크플로우 통합)

| 명령 | 인자 | 출력 | MVP/Post |
|---|---|---|---|
| `factory init` | `[--dry-run]` | 자료실 + `projects/_example` 시드 | MVP |
| `factory doctor` | `[--check playwright\|encoding\|fonts]` | stdout 진단 | MVP |
| `factory project new <slug>` | | `projects/[slug]/prompts/` | MVP |
| `factory project import` | `--from <slug> [--include design_blocks\|image_prompts\|channel\|all]` | (재사용 UI §17) | MVP |
| `factory extract prompt` | `--batch <id> [--mode manual\|ocr]` | `projects/[slug]/prompts/P1_extract.md` | MVP |
| `factory extract import` | `--batch <id> --from <path.md>` | `extract/[batch]/{extracted.md, extracted.json, review.md}` | MVP |
| `factory extract apply` | `--batch <id> --product <slug>` | `products/[slug]/original_extracted.md` | MVP |
| `factory blocks add` | `--from <md_file>` (interactive) | `blocks/blocks.jsonl` 1줄 | MVP |
| `factory blocks list` | `[--filter tone=... channel=...]` | stdout 표 | MVP |
| `factory design-blocks add` | `--from <md_file>` (interactive) | `blocks/design_blocks.jsonl` 1줄 | MVP |
| `factory design-blocks list` | `[--filter design_block_type=...]` | stdout 표 | MVP |
| `factory recipe new` | `--name <s> --channel <c> --profile <p>` | `recipes/recipes.md` 1행 | MVP |
| `factory generate` | `--project <slug> [--model gpt\|claude\|gemini]` | `projects/[slug]/prompts/{P3_style.md, P4_image_elem/*, P5_channel/*}` | MVP |
| `factory copy import` | `--project <slug> --from <path.md>` | `output/[slug]/copy_master.json` | MVP |
| `factory sample` | `--project <slug> [--only N]` | `output/[slug]/{page_manifest.json, page_NNN.html}` | MVP |
| `factory render` | `--project <slug> [--only N]` | `output/[slug]/{page_NNN.png, quality_report.json}` (12 항목) | MVP |
| `factory ai send` | `--prompt <path> --model <m>` | Phase 2 — API 호출 | Post-MVP |
| `factory publish` | `--project <slug> --channel <c>` | Phase 3 | Post-MVP |
| `factory expand` | `--project <slug> --pages N` | Phase 4 | Post-MVP |

원칙: 명령은 4-단어 이하. 모호한 이름(예: `factory analyze`) 사용 안 함.

---

## 37. 학습 자료실 운영 보조

(v1.2 §4-§14 계승. 상세 표는 v1.2 잠금본 참조)

### 37.1 SKILL.md 라우팅 (5개 이하 파일)

| 요청 유형 | 1순위 | 2순위 | 3순위 | 4순위 | 5순위 |
|---|---|---|---|---|---|
| 카드뉴스 새로 만들기 | `channels/instagram_cardnews.md` | `recipes/recipes.md` | `voice/[tone].md` | `brand/brand_identity.md` | `blocks/blocks.jsonl` |
| 랜딩페이지 카피 | `channels/landing_page.md` | `logic/problem_agitation_solution.md` | `brand/proof_assets.md` | `voice/[tone].md` | `blocks/blocks.jsonl` |
| Threads 글 | `channels/threads.md` | `voice/friendly_sns_tone.md` | `examples/trusta_threads_good_01.md` | `brand/forbidden_words.md` | - |
| 이메일 | `channels/email.md` | `logic/lead_capture.md` | `voice/[tone].md` | `brand/anti_patterns.md` | - |
| 디자인 블록 검토 | `blocks/design_blocks.jsonl` | `system/schema_blocks.md` | `system/naming_rules.md` | - | - |
| 제품 맥락 파악 | `products/[slug]/meta.yml` | `products/[slug]/product_summary.md` | `products/[slug]/original_extracted.md` | - | - |

### 37.2 voice/[tone].md 필수 섹션

1. 톤 정의
2. 사용 맥락
3. 샘플 문장 (핵심 톤 20개 / 보조 톤 12~20개)
4. 금지 표현
5. 자주 쓰는 어미/접속사
6. 길이/리듬 가이드

### 37.3 examples/ 운영

- 채널당 1~2개의 완성된 골든 샘플
- 파일명: `[product_slug]_[channel]_good_[NN].md`
- frontmatter에 사용한 recipe / blocks / tone 명시

---

## 38. naming_rules enum (잠금)

(v1.2 §7 + v1.3 §6 + §9 + §11 통합)

| 카테고리 | 허용값 |
|---|---|
| `tone` | `expert_saas` / `friendly_sns` / `luxury_brand` (voice/ 파일명과 일치) |
| `channel` | `threads` / `instagram_cardnews` / `instagram_story` / `instagram_post` / `linkedin_post` / `medium_article` / `pinterest_pin` / `x_thread` / `facebook_post` / `a4_report` / `email` / `landing_page` (12개) |
| `channel_essence` | `long_form_text` / `short_form_text` / `card_visual` / `mixed` (v1.3 §11.1) |
| `block_type` | `hook` / `problem` / `pain_point` / `solution` / `benefit` / `proof` / `before_after` / `objection` / `cta` / `trust` / `feature` / `comparison` (12개) |
| `design_block_type` | `icon` / `illustration` / `hero_image` / `chart` / `divider` / `cta_button` / `tag` / `card_layout` / `color_palette` / `typography` (10개, v1.3 §9) |
| `elem_type` (image_slots) | `icon` / `illustration` / `hero_image` / `chart_visual` / `pattern` / `divider` (6개, v1.3 §10) |
| `image_slot_status` | `prompt_only` / `pending` / `ready` / `approved` (v1.3 §10) |
| `status` | `draft` / `approved` / `archived` |
| `source_type` | `own` / `benchmark` / `user_added` |
| `industry` | snake_case (자유 입력, snake_case 강제) |
| `logic_role` | `problem_agitation_solution` / `lead_capture` / `conversion_boost` / `before_after` / `comparison` |
| `scope` (v1.3 §6) | `global` / `block` / `asset` |
| `prompt_type` (v1.3 §5) | `P1_EXTRACT` / `P2_CLASSIFY` / `P3_STYLE` / `P4_IMAGE_ELEM` / `P5_CHANNEL` |
| `model_hint` (v1.3 §6) | `gpt` / `claude` / `gemini` / `any` |
| `output_profile` | `a4_document` / `instagram_square` / `instagram_portrait` / `story_9_16` / `threads_image` / `linkedin_post` / `pinterest_pin` / `x_card` / `facebook_post` / `medium_header` (10개) |

**규칙**:
- 영문 snake_case만 (한글 enum 금지)
- enum 외 값은 적재 시 거부 → `system/retrieval_rules.md`에 거부 사례 기록
- enum 단일 정의 위치: `src/factory/schema/enums.py` (YAML 분류기 폐기)

---

## 39. recipes.md 컬럼

(v1.2 §9 계승)

| 컬럼 | 의미 |
|---|---|
| `recipe_id` | 짧은 고유값 |
| `name` | 사람이 부르는 이름 |
| `goal` | 만들고 싶은 결과 |
| `channel` | 대상 채널 (channels enum 12개 중) |
| `output_profile` | 10개 중 |
| `target_pages` | 목표 장수 N |
| `sample_count` | 기본 6 |
| `logic_role` | PAS / B/A / 등 |
| `tone` | voice enum |
| `required_blocks` | 필수 block_type 시퀀스 |
| `notes` | 비고 |

---

## 40. products/[slug]/meta.yml

(v1.2 §10)

```yaml
product_slug: trusta_pro
product_name: "TRUSTA Pro"
industry: legal_services
tone: expert_saas
positioning: ""
target_audience: ""
key_strengths: []
proof_assets: []
forbidden_words: []
source_files:
  - original_extracted.md
  - product_summary.md
created_at: ""
updated_at: ""
```

---

## 41. 추출(extract) 파이프라인 — 공급망 분리

(v1.2 §20)

### 41.1 4단계

| 단계 | CLI 명령 | 책임 |
|---|---|---|
| 1. 준비 | (사용자가 `extract/input/[batch]/*.png` 배치) | 사용자 |
| 2. 프롬프트 빌드 | `factory extract prompt --batch b001` | CLI (P1 출력) |
| 3. 외부 회수 + 검수 | `factory extract import --batch b001 --from <path>` | CLI 적재 + 사용자 검수 |
| 4. 적용 | `factory extract apply --batch b001 --product <slug>` | products/[slug]/original_extracted.md |

### 41.2 자동 적재 금지

- LLM 응답 받아도 `blocks.jsonl`/`design_blocks.jsonl`에 자동 추가하지 않음
- 항상 사람이 `factory blocks add` / `factory design-blocks add`로 1줄씩

### 41.3 모드

| 모드 | 비용 | MVP/Post |
|---|---|---|
| 반자동 프롬프트 | **없음** | MVP 1순위 |
| OCR (Tesseract) | 낮음 | 보류 |
| Vision (Claude/GPT-4V/Gemini Vision) | 높음 | Post-MVP |
| OCR + Vision 검수 | 중간 | Post-MVP |

---

## 42. AI 모델 어댑터 구조 (Phase 2 설계만)

(v1.2 §21 + v1.3 §19)

| 모드 | 비용 | MVP/Post |
|---|---|---|
| **반자동 프롬프트 출력** (P1~P5 .md) | 없음 | **MVP** |
| API 자동 생성 (GPT/Claude/Gemini) | 있음 | Phase 2 |
| 이미지 API (DALL·E / Imagen / Midjourney) | 매우 높음 | Post-MVP (배경/일러스트만) |

**핵심**: 최종물은 **HTML/CSS → Playwright PNG**. 이미지 생성은 배경/일러스트/아이콘 슬롯용으로만 검토 (v1.3 §3-4, §3-5).

### 42.1 어댑터 폴더 (Phase 2 placeholder만 MVP에 둠)

```
src/factory/ai/
├── adapter_base.py         ← MVP placeholder
├── adapter_openai.py       ← Phase 2
├── adapter_anthropic.py    ← Phase 2
├── adapter_google.py       ← Phase 2
└── cost_guard.py           ← Phase 2
```

### 42.2 API key 보안

- 클라이언트 노출 금지
- 로컬 환경변수에서만 읽음
- Phase 2 웹앱화 시 서버 route에서만 처리

---

## 43. V1 재사용 자산 (선별 이식)

(V1 진단 §2 + v1.3 §24 계승)

### 43.1 코드 이식

| 자산 | V1 경로 | V2 경로 | 변경 |
|---|---|---|---|
| `cache.py` (Windows-safe SHA256 sharded JSON) | `src/factory/analyze/cache.py` | `src/factory/utils/cache.py` | 위치 변경 |
| `slicer.py` (PIL PNG 슬라이서) | `src/factory/analyze/slicer.py` | `src/factory/extract/slicer.py` | 위치 변경 |
| Typer entry + `@app.callback()` (L1) | `src/factory/__main__.py` | `src/factory/__main__.py` | help 텍스트 ASCII (L2) |
| `init_cmd.py` SEED_FILES 패턴 | `src/factory/cli/init_cmd.py` | `src/factory/cli/init_cmd.py` + `library/seed.py` | SEED 내용 v1.3 자료실로 전면 교체 |
| `test_cache.py` (14 tests) | `tests/test_cache.py` | `tests/test_cache.py` | 그대로 |
| `test_slicer.py` (19 tests) | `tests/test_slicer.py` | `tests/test_slicer.py` | 그대로 |
| `_make_png` 헬퍼 | (각 테스트 내부) | `tests/conftest.py` | 승격 |

### 43.2 시드 이식

| V1 자산 | V2 경로 | 변경 |
|---|---|---|
| `input/forbidden_phrases.txt` 20개 | `content-learning-library/brand/forbidden_words.md` | 마크다운 형식 |
| `input/business_profile.md` | `content-learning-library/products/_example/meta.yml` + `product_summary.md` | 분해 |
| `input/style_notes.md` | `content-learning-library/voice/expert_saas_tone.md` (등) | 분해 |
| `asset_pack/manifest.yaml` 스키마 | `content-learning-library/products/_example/source_images/manifest.yaml` | 위치 변경 |

### 43.3 학습 (L1/L2/L3) — V2 Day 1 즉시 적용

| ID | 학습 | V2 적용 |
|---|---|---|
| **L1** | Typer 단일 command 자동 root → `@app.callback()` 강제 | `__main__.py`에 즉시 |
| **L2** | Windows cp949 em-dash UnicodeEncodeError | typer help / CLI 메시지 ASCII만 |
| **L3** | `PYTHONIOENCODING=utf-8` 필요 | README + `factory doctor`에 안내 |

### 43.4 도구 설정

| 자산 | V2 반영 |
|---|---|
| `pyproject.toml` ruff strict (E/W/F/I/B/C4/UP/SIM) | 그대로 |
| `pyproject.toml` mypy strict | 그대로 |
| `pyproject.toml` `anthropic>=0.40` | **제거** (v1.3 §3-6, G5) |
| `pyproject.toml` `tenacity>=9.0` | **MVP 제거**, Phase 2 adapter와 함께 |
| uv workflow | 유지 |
| `.gitignore` | 그대로 |

---

## 44. V1 폐기 자산 (최종)

| 폐기 대상 | 사유 |
|---|---|
| 5계층 단방향 파이프라인 | v1.3 다이아몬드 흐름 |
| Anthropic SDK 직접 호출 | v1.3 §3-2 외부 LLM 웹 |
| 5개 고정 HTML 템플릿 (T1~T5) | output_profile 10개로 교체 |
| `block_types.yaml` 12 타입 고정 | enums.py 단일 정의 (§38) |
| `flow/synthesis/render/ethics/manifest` 기존 모듈 | §19 새 구조 |
| `factory analyze --set` 기존 시그니처 | §36 새 명령 표 |
| `factory expand --pages 40` | Phase 4 |
| `factory cache clear/stats` | Post-MVP |
| page_manifest.csv 18 컬럼 | copy_master.json + page_manifest.json |
| 9개 메타 문서 | 3개 (SYSTEM_TECH/PLAN/CLAUDE) |
| REVIEW_CHECKLIST 자동 점수 ≥80 | PNG 품질 체크 12 (§21) |
| 윤리·유사도·워터마크 하드게이트 | v1.2 §31 제거 |
| `quality_score` 필드 | v1.2 §25 금지 |
| 날짜형 ID | v1.2 §6 금지 |
| 기존 M2-5 GREEN 흐름 | 새 폴더 처음부터 (v1.3 §24) |
| V1 폴더 직접 패치 | v1.3 §33-1 |

---

## 45. v1.0 → v1.1 → v1.2 → v1.3 변경 누적

| 항목 | v1.1 | v1.2 | v1.3 (본 문서) |
|---|---|---|---|
| 40장 고정 제거 → `target_pages=N` | 신규 | 유지 | 유지 |
| `sample_count=6` 기본 | 신규 | 유지 | 유지 |
| `output_profile` 분기 | 신규 (5개) | 유지 | **확장 10개** |
| `publish_adapter: none\|future` placeholder | 신규 | 유지 | 유지 |
| 자동게시 미구현 | 신규 | 유지 | 유지 |
| Framer식 풀 편집 UI 제외 | 신규 | 유지 | 유지 |
| 윤리/유사도/워터마크 하드게이트 제거 | 신규 | 유지 | 유지 |
| JSON schema / 파일 존재 / 캐시 검증 | 신규 | 유지 | 유지 |
| OCR/Vision 파이프라인 분리 | — | 신규 | 유지 |
| AI 모델 어댑터 구조 (Phase 2 설계) | — | 신규 | 유지 |
| 반자동 프롬프트 출력 1순위 잠금 | — | 신규 | 유지 |
| **Prompt Layer / Prompt Pack 개념** | — | — | **v1.3 신규** |
| **사용자 UI 용어 6개 제한 ("Pack" 노출 금지)** | — | — | **v1.3 신규** |
| **P1~P5 표준 코드 잠금** | — | — | **v1.3 신규** |
| **3층 scope (global/block/asset) frontmatter** | — | — | **v1.3 신규** |
| **새 폴더 `a4-page-flow-factory-v2` 결정** | — | — | **v1.3 신규** |
| **Day 5 instagram_portrait PNG 1장 단일 게이트** | — | — | **v1.3 신규** |
| **design_blocks.jsonl 분리** | — | — | **v1.3 신규** |
| **image_slots schema (required, drag&drop)** | — | — | **v1.3 신규** |
| **channels 9개 + 본질 4개 어댑터** | — | — | **v1.3 신규** |
| **워크플로우 9단계** | — | — | **v1.3 신규** |
| **UI6 4-pane 추출 화면** | — | — | **v1.3 신규** |
| **모델별 토글 (GPT/Claude/Gemini 형식)** | — | — | **v1.3 신규** |
| **룰 기반 자동 초안** | — | — | **v1.3 신규** |
| **재사용 UI "다른 프로젝트에서 가져오기"** | — | — | **v1.3 신규** |
| **변경 이력 git에 위임** | — | — | **v1.3 신규** |
| **PNG 품질 체크 12 항목 (V2 draft 11 → 12 교정)** | — | — | **v1.3 신규** |
| **projects/[slug]/prompts/ 폴더 구조** | — | — | **v1.3 신규** |
| **자료실에 prompts/ 템플릿 폴더** | — | — | **v1.3 신규** |

---

## 46. 다음 Captain 실행 프롬프트 (Day 1 초안)

> **주의**: 본 문서에서는 코드 작성하지 않는다. 아래는 다음 세션용 프롬프트 *초안*.

```
너는 a4-page-flow-factory-v2 구현자다.

전제:
- 본 세션 시작 시 docs/A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md를 먼저 읽고 시작한다.
- 새 폴더 a4-page-flow-factory-v2를 부모 디렉토리에 생성한다.
- 기존 V1 폴더(a4-page-flow-factory)는 절대 수정하지 않는다.
- V1에서 가져올 자산은 본 문서 §43 표에 명시되어 있다.

이번 세션 작업 (Day 1만):
1. 새 폴더 a4-page-flow-factory-v2 생성
2. pyproject.toml (anthropic / tenacity 제거, ruff/mypy strict 유지)
3. .gitignore
4. SYSTEM_TECH.md (본 문서 ≤500 lines 압축본)
5. PLAN.md (Day 1~5 체크박스만)
6. CLAUDE.md (≤200 lines AI 규칙)
7. uv sync
8. src/factory/__main__.py — Typer + @app.callback() (L1 적용)
9. factory --help가 Windows cp949에서 깨지지 않음 확인 (L2)
10. factory doctor 빈 명령 (L3 안내)
11. V1에서 cache.py / slicer.py / test_cache.py / test_slicer.py 복사
12. tests/conftest.py에 _make_png 헬퍼 승격
13. 33/33 PASS 확인

금지:
- Day 2 이후 작업 손대기
- anthropic / openai / google-genai / tenacity 의존성 추가
- 4번째 메타 문서 생성
- V1 폴더 수정
- 사용자 UI 단어 "Pack" 등장

종료 조건:
- uv run pytest tests/test_cache.py tests/test_slicer.py → 33/33 PASS
- uv run factory --help cp949에서 깨짐 없음
- PLAN.md의 Day 1 체크박스만 [x]
- HALT 후 인수인계 메모
```

---

## 47. 최종 결론

### 47.1 본 문서의 자기 평가

| 평가 축 | 결과 |
|---|---|
| v1.0 → v1.3까지 모든 잠금 결정을 단일 문서에 통합? | ✅ §45 변경 누적표 |
| v1.3 합의안 §1~§33 전체 반영? | ✅ §1~§33 직접 매핑 + §34~§46 보강 |
| 새 아이디어 추가 금지 준수? | ✅ 모든 섹션이 V1진단/v1.2/V2draft/v1.3 4개 출처 추적 가능 |
| PNG 품질 체크 12개 잠금? | ✅ §21 (V2 draft 11 → 12 교정 명시) |
| design_blocks.jsonl 분리 반영? | ✅ §9, §11 enum, §44 |
| image_slots `required` 필드 반영? | ✅ §10, §14.5 |
| 채널 9개 + 본질 4개 어댑터 반영? | ✅ §11, §38 enum |
| output_profile 10개 잠금? | ✅ §12, §38 enum |
| 워크플로우 9단계 반영? | ✅ §13 |
| UI6 4-pane / 모델 토글 / 룰 기반 자동 초안 반영? | ✅ §14, §15, §16 |
| 사용자 UI "Pack" 노출 금지 강제? | ✅ §2.3, §17, §33-13, §46 |
| Day 5 단일 게이트 잠금? | ✅ §1-12, §25, §26 |
| V1 폴더 수정 금지 명시? | ✅ §24, §33-1, §44, §46 |
| 코드 작성 지시 없음? | ✅ |

### 47.2 본 문서의 위치

- **새 폴더 `a4-page-flow-factory-v2`의 spec + PRD + plan**
- 다음 Captain 세션은 본 문서를 가장 먼저 읽고 시작
- 본 문서의 압축본을 `SYSTEM_TECH.md`로 (≤500 lines), Day 1~5 체크박스만 `PLAN.md`로 (≤300 lines), AI 규칙은 `CLAUDE.md`로 (≤200 lines) 분리

### 47.3 한 줄 마무리

> **V1은 인프라 함정에 빠져 멈췄다. v1.3 V2는 Day 5에 instagram_portrait PNG 1장을 12/12 품질 체크로 끝까지 통과시키는 것이 유일한 게이트다. 그 외 모든 욕심(API 자동 호출, 자동게시, DB, 이미지 생성 AI, 풀 편집 UI, 자체 변경 이력, output_profile 10개 동시 구현)은 Phase 2~5로 미룬다.**

---

*작성: 2026-05-24*
*근거: V1 진단(`docs/V1_CURRENT_STATE_REVIEW.md`) + v1.2 잠금본 + V2 재설계 draft(`docs/V2_FULL_REDESIGN_DRAFT.md`) + 사용자 v1.3 합의안 §1~§33*
*충돌 해결 우선순위: 사용자 외부 지시 > v1.3 합의안 > v1.2 잠금본 > V2 draft > V1 진단*
*다음 액션: 본 문서 승인 → 새 폴더 `a4-page-flow-factory-v2` 생성 → 본 문서 압축 이식 → Day 1 시작*
