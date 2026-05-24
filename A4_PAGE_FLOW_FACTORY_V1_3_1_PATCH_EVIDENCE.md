# A4 Page Flow Factory v1.3.1 패치 — evidence 파이프라인

> **상태**: 패치 잠금본 (Locked)
> **버전**: v1.3.1
> **역할**: v1.3 잠금본 위에 덧붙이는 **증거(evidence) 파이프라인** 추가본
> **원본 영향**: v1.3 잠금본 0줄 수정. Day 1~5 단일 게이트 그대로 보호.
> **작성일**: 2026-05-24
> **출처**: 사용자 요청 워크플로우 (레퍼런스 증거 → 후보 제안 → 생성 프롬프트 → 외부 수집 → proof_assets 적재 → P5 카피 사용)

---

## 0. 패치 정보

| 항목 | 값 |
|---|---|
| 패치명 | `A4_PAGE_FLOW_FACTORY_V1_3_1_PATCH_EVIDENCE.md` |
| 적용 대상 | `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` (v1.3 잠금본) |
| 위치 | 새 폴더 `a4-page-flow-factory-v2`의 `docs/` |
| 원본 수정 | **0줄** (별도 패치 파일로 첨가) |
| Day 1~5 게이트 영향 | **없음** (Day 6~7에 추가) |
| P1~P5 잠금 | **그대로** (P6 만들지 않음) |
| 새 의존성 | **0개** |

---

## 1. 적용 위치 요약

| § | 변경 종류 | 내용 |
|---|---|---|
| §48 | 신규 | evidence 파이프라인 4단계 |
| §49 | 신규 | `proof_assets.jsonl` 스키마 |
| §38 | 추가 | enum 2개 (proof_type, proof_source_type) |
| §7.1 | 추가 | `content-learning-library/proof_assets/` 폴더 |
| §34 | 1줄 추가 | P5_CHANNEL 입력에 `proof_assets` 매칭 |
| §36 | 4줄 추가 | CLI `factory evidence suggest/prompt/import/apply` |
| §26 | Day 6~7 추가 | 1주차 계획 확장 |
| §22 | 1줄 추가 | MVP 포함 표 |
| §33 | 2개 추가 | 금지 14~15 |

---

## 2. 사용자 워크플로우 매핑

| # | 사용자 요구 | v1.3.1 매핑 |
|---|---|---|
| 1 | 레퍼런스에 있는 증거 표현을 보고 | §48 단계 1 (suggest는 `benchmarks/*` + `blocks.jsonl` 매칭) |
| 2 | 내 사업에서 대체 가능한 증거 후보 2~3개 제안 | `factory evidence suggest --benchmark <id>` |
| 3 | 외부 LLM에 붙여넣을 프롬프트 생성 | `factory evidence prompt --candidate <id> [--model gpt\|claude\|gemini]` |
| 4 | 외부 사람이 베타테스트/설문/샘플 리포트/논문/크롤링으로 수집 | `proof_source_type` enum: measured/survey/paper/crawl/sample_report/user_provided |
| 5 | Factory에 다시 import → proof_assets 저장 | `factory evidence import` → `factory evidence apply` → `proof_assets.jsonl` 1줄 |
| 6 | 카드뉴스/상세페이지/보고서 문구에 사용 | §34 P5 입력에 `proof_assets/proof_assets.jsonl` 매칭 추가 |

---

## 3. §48 (신규) — evidence 파이프라인 4단계

extract 파이프라인과 **동일한 패턴**으로 구성. P1~P5 잠금 안 깸.

### 3.1 4단계

| 단계 | CLI 명령 | 책임 | 출력 |
|---|---|---|---|
| 1. 증거 후보 제안 | `factory evidence suggest --benchmark <id> [--product <slug>]` | CLI가 벤치마크 증거 패턴 → 내 사업 대체 후보 2~3개 출력 | stdout 표 + `proof_assets/_candidates/<cand>.md` |
| 2. 생성 프롬프트 빌드 | `factory evidence prompt --candidate <id> [--model gpt\|claude\|gemini]` | 외부 LLM에 붙여넣을 .md (모델별 형식) | `proof_assets/_candidates/<cand>_prompt_<model>.md` |
| 3. 외부 회수 + 검수 | `factory evidence import --candidate <id> --from <path>` | 외부 수집 결과 회수 + review.md 생성 | `proof_assets/_candidates/<cand>_review.md` |
| 4. proof_assets 적재 | `factory evidence apply --candidate <id> --product <slug>` | 사람 검수 후 jsonl 1줄 추가 | `proof_assets/proof_assets.jsonl` 1줄 |

### 3.2 원칙

- 외부 사람(베타테스터/연구자/연하)이 **비동기로 수집** (베타테스트/설문/샘플 리포트/논문 조사/크롤링)
- LLM 응답을 `proof_assets.jsonl`에 **자동 적재 금지** (사람 검수 필수, extract와 동일 원칙)
- `proof_source_type=sample_report`인 경우 `watermark_recommended=true` 자동 설정
- 후보 제안 단계는 **API 호출 없음** (벤치마크의 증거 패턴 + 내 product meta.yml 매칭으로 룰 기반 추천)

### 3.3 데이터 흐름

```
                ┌──────────────────────────────────────┐
                │  factory evidence suggest            │
                │  benchmarks/<bench> + product meta   │
                │  → 2~3 후보 .md                       │
                └──────────────┬───────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────┐
                │  factory evidence prompt --model X   │
                │  → 외부 LLM용 .md (Markdown/XML/예시) │
                └──────────────┬───────────────────────┘
                               │
                       [외부 LLM 웹]
                               │
                               ▼
                ┌──────────────────────────────────────┐
                │  연하/외부 사람이 수집                │
                │  베타테스트/설문/논문/크롤링/샘플리포트│
                └──────────────┬───────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────┐
                │  factory evidence import --from X    │
                │  → review.md (누락/중복/출처 검수)    │
                └──────────────┬───────────────────────┘
                               │
                       [사람 검수 + 승인]
                               │
                               ▼
                ┌──────────────────────────────────────┐
                │  factory evidence apply              │
                │  → proof_assets.jsonl 1줄            │
                └──────────────┬───────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────────┐
                │  P5_CHANNEL (§34) 입력에 자동 매칭   │
                │  → 카드뉴스/상세페이지/보고서 카피   │
                └──────────────────────────────────────┘
```

---

## 4. §49 (신규) — proof_assets.jsonl 스키마

**형식**: JSONL (한 줄에 한 객체. JSON 배열 아님)
**위치**: `content-learning-library/proof_assets/proof_assets.jsonl`
**원칙**: blocks.jsonl / design_blocks.jsonl과 동급의 **단일 진실 원본**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `id` | string | ✅ | 짧은 랜덤 (예: `pf_a3k9x`) |
| `proof_type` | enum(6) | ✅ | stat / quote / case_study / before_after / certification / screenshot |
| `proof_source_type` | enum(6) | ✅ | measured / survey / paper / crawl / sample_report / user_provided |
| `claim` | string | ✅ | 한 줄 핵심 주장 (수치 단정 금지) |
| `evidence_text` | string | ✅ | 증거 본문 |
| `source_url` | string\|null | | 출처 URL (논문/크롤링 시) |
| `source_doc` | string\|null | | 첨부 파일 경로 (PDF/이미지) |
| `collected_by` | string | ✅ | 수집자 이름 |
| `collected_at` | ISO8601 | ✅ | 수집일 |
| `tags` | string[] | | snake_case |
| `product_slug` | string | ✅ | 어느 제품용 |
| `linked_text_block` | string\|null | | `blocks.jsonl` id (선택) |
| `status` | enum(3) | ✅ | draft / approved / archived |
| `watermark_recommended` | boolean | ✅ | `proof_source_type=sample_report` 이면 자동 true |
| `created_at` | ISO8601 | ✅ | |
| `updated_at` | ISO8601 | ✅ | |

### 4.1 금지

- ❌ `quality_score` 필드 (v1.2 §25)
- ❌ 날짜형 `id` (v1.2 §6)
- ❌ "100% / 완벽 / 유일 / 보장" 같은 단정 표현 (`brand/forbidden_words.md` 적용)
- ❌ 자동 적재 (사람 검수 필수)

### 4.2 예시 1줄 (참고용)

```json
{"id":"pf_a3k9x","proof_type":"survey","proof_source_type":"survey","claim":"베타테스트 참여자 다수가 1주차에 첫 결과를 받았다","evidence_text":"2026-04 베타테스트 N=23, 1주차 첫 결과 수령 비율 보고","source_doc":"proof_assets/_archive/beta_202604_report.pdf","collected_by":"내부 베타팀","collected_at":"2026-04-30T18:00:00+09:00","tags":["beta","trusta_pro","week1"],"product_slug":"trusta_pro","linked_text_block":"b_kj92","status":"approved","watermark_recommended":false,"created_at":"2026-05-02T10:00:00+09:00","updated_at":"2026-05-02T10:00:00+09:00"}
```

---

## 5. §38 (추가) — enum 2개

단일 정의 위치: `src/factory/schema/enums.py` (v1.3 §38과 같은 파일)

| 카테고리 | 허용값 |
|---|---|
| `proof_type` | `stat` / `quote` / `case_study` / `before_after` / `certification` / `screenshot` (6개) |
| `proof_source_type` | `measured` / `survey` / `paper` / `crawl` / `sample_report` / `user_provided` (6개) |

**규칙** (v1.3 §38 계승):
- 영문 snake_case만 (한글 enum 금지)
- enum 외 값은 적재 시 거부
- enum 단일 정의 위치는 `enums.py`

---

## 6. §7.1 (추가) — 자료실 폴더

`content-learning-library/` 아래에 추가:

```
content-learning-library/
└── proof_assets/                  ← v1.3.1 신규
    ├── proof_assets.jsonl         ← 단일 원본 (blocks/design_blocks와 동급)
    ├── _candidates/               ← 외부 수집 전 후보 .md (factory evidence suggest 출력)
    │   ├── <cand_id>.md
    │   ├── <cand_id>_prompt_gpt.md
    │   ├── <cand_id>_prompt_claude.md
    │   ├── <cand_id>_prompt_gemini.md
    │   └── <cand_id>_review.md
    └── _archive/                  ← 만료/원본 첨부 (PDF/이미지)
```

기존 `brand/proof_assets.md`는 **"증거 운용 원칙 문서"로 역할 재정의** (스키마 아님, 가이드).

---

## 7. §34 (수정 1줄) — P5_CHANNEL 입력에 추가

기존 §34 P5 행:

> | **P5** | 매칭된 `blocks.jsonl` 행 + P3 결과 + `channels/[ch].md` + `prompts/channel_adapters/[ch].md` + `logic/[role].md` + `examples/*.md` | `projects/[slug]/prompts/P5_channel/[ch].md` | `output/[slug]/copy_master.json` → `factory copy import` |

**변경 후**:

> | **P5** | 매칭된 `blocks.jsonl` 행 + **매칭된 `proof_assets/proof_assets.jsonl` 행 (product_slug + linked_text_block 기준)** + P3 결과 + `channels/[ch].md` + `prompts/channel_adapters/[ch].md` + `logic/[role].md` + `examples/*.md` | `projects/[slug]/prompts/P5_channel/[ch].md` | `output/[slug]/copy_master.json` → `factory copy import` |

→ P5 CHANNEL 프롬프트가 카드뉴스/상세페이지/보고서 카피 생성 시 `proof_assets`를 명시적으로 인용.

---

## 8. §36 (추가) — CLI 4개

기존 §36 CLI 표에 다음 4줄 추가:

| 명령 | 인자 | 출력 | MVP/Post |
|---|---|---|---|
| `factory evidence suggest` | `--benchmark <id> [--product <slug>]` | `proof_assets/_candidates/<cand>.md` (2~3개 후보) | MVP (Day 6) |
| `factory evidence prompt` | `--candidate <id> [--model gpt\|claude\|gemini]` | `proof_assets/_candidates/<cand>_prompt_<model>.md` | MVP (Day 6) |
| `factory evidence import` | `--candidate <id> --from <path>` | `proof_assets/_candidates/<cand>_review.md` | MVP (Day 6) |
| `factory evidence apply` | `--candidate <id> --product <slug>` | `proof_assets/proof_assets.jsonl` 1줄 | MVP (Day 6) |

**명령 그룹화** (§7.2 v1.3 계승):

```
factory
  ├── ...기존 18개 명령
  └── evidence suggest|prompt|import|apply   ← v1.3.1 신규 (extract와 동일 패턴)
```

---

## 9. §26 (Day 6~7 추가) — 1주차 계획 확장

> ⚠️ **Day 1~5는 절대 변경 금지** (v1.3 §3-12 단일 게이트 보호). Day 6~7만 추가.

| Day | 작업 | 완료 조건 |
|---|---|---|
| **Day 1** | (v1.3 §26 그대로) | 33/33 PASS |
| **Day 2** | (v1.3 §26 그대로) | blocks 검증 |
| **Day 3** | (v1.3 §26 그대로) | P3+P5 골든 |
| **Day 4** | (v1.3 §26 그대로) | HTML 1장 |
| **Day 5** | (v1.3 §26 그대로) | **PNG 1장 + 12/12** ← 단일 게이트 |
| **Day 6** | evidence 파이프라인 추가 | `proof_assets.jsonl` 검증 + `factory evidence suggest` 1회 통과 + enum 2개 추가 + CLI 4개 |
| **Day 7** | proof_assets 시드 + 골든 픽스처 | 시드 5줄 + golden 1개 + `evidence apply` → P5 인용 1회 통과 |

### 9.1 Week 1 종료 게이트 확장

- [x] (v1.3 §26.1 그대로) PNG 1장 + 12/12 + L1/L2/L3
- [ ] **(추가) `proof_assets.jsonl` 시드 5줄 적재**
- [ ] **(추가) `factory evidence` 4개 명령 통과**
- [ ] **(추가) P5 CHANNEL 프롬프트에 `proof_assets` 인용 1회 통과**

### 9.2 추가 일정 영향

| 항목 | 값 |
|---|---|
| 추가 일수 | 2일 (Day 6~7) |
| 추가 코드 라인 | 약 250 lines (CLI 4개 + enum 2개, extract 패턴 복제) |
| 추가 테스트 | proof_assets schema 1 + evidence CLI 4 + golden 1 = 6 tests |
| 새 의존성 | 0개 |

---

## 10. §22 (1줄 추가) — MVP 포함

기존 §22 MVP 포함 표에 한 줄 추가:

| 항목 | 포함 여부 |
|---|---|
| (v1.3 §22 기존 15개 항목) | 포함 |
| **evidence 파이프라인 (suggest → prompt → import → apply)** | **포함 (Day 6~7)** |

---

## 11. §33 (금지 2개 추가) — 최종 하지 말아야 할 것

기존 v1.3 §33 13개에 다음 2개 추가:

```
14. 가짜 증거를 proof_assets.jsonl에 적재하지 않는다.
    (proof_source_type enum 강제. measured/survey/paper/crawl/sample_report/user_provided 외 값 거부)

15. evidence 파이프라인을 P6_PROOF로 P1~P5 잠금에 추가하지 않는다.
    (extract와 동급의 별도 파이프라인으로 유지. P1~P5 5개 잠금 보호)
```

---

## 12. 검증 추가 (V2 draft §9 7개 축 계승)

evidence 산출물(`.md` 후보 + `proof_assets.jsonl`)에 적용할 검증:

| 검증 ID | 이름 | 자동/수동 | 통과 기준 |
|---|---|---|---|
| V-EV-SCHEMA | proof_assets.jsonl 스키마 | 자동 | §49 17개 필수/선택 필드 정합성 |
| V-EV-ENUM | proof_type / proof_source_type enum | 자동 | §38 추가 enum 12개 값 외 거부 |
| V-EV-FORB | 단정 표현 미포함 | 자동 | `forbidden_words.md` 라인이 `claim`/`evidence_text`에 없음 |
| V-EV-WATERMARK | sample_report 자동 워터마크 | 자동 | `proof_source_type=sample_report` → `watermark_recommended=true` |
| V-EV-LINK | linked_text_block 무결성 | 자동 | 참조한 `blocks.jsonl` id가 실제 존재 |
| V-EV-HUMAN | 사람 승인 게이트 | 수동 | `factory evidence apply`는 `review.md` APPROVED 후에만 가능 |

---

## 13. 변경 영향 요약

| 측면 | 변화 |
|---|---|
| 원본 v1.3 잠금본 수정 | **0줄** |
| Day 1~5 단일 게이트 (PNG 1장 12/12) | **그대로 보호** |
| P1~P5 잠금 (5개) | **그대로** (P6 만들지 않음) |
| 메타 문서 3개 잠금 (SYSTEM_TECH/PLAN/CLAUDE) | **그대로** (본 패치는 docs/ 안 4번째 메타 문서 아님) |
| 사용자 UI 단어 6개 잠금 | **그대로** ("Pack" 노출 없음) |
| 새 의존성 | **0개** (LLM SDK 추가 없음) |
| 새 enum | **2개** (proof_type, proof_source_type) |
| 새 jsonl | **1개** (proof_assets.jsonl) |
| 새 CLI 명령 | **4개** (evidence suggest/prompt/import/apply) |
| 새 자료실 폴더 | **1개** (`content-learning-library/proof_assets/`) |
| 새 검증 축 | **6개** (V-EV-*) |
| 추가 일정 | **Day 6~7 (2일)** |
| 추가 코드 라인 | **~250 lines** (extract 패턴 복제) |
| 추가 테스트 | **6 tests** |

---

## 14. 다음 액션

1. 사용자 검토
2. 승인 시 `git tag v1.3.1`
3. 새 폴더 `a4-page-flow-factory-v2/docs/` 에 본 패치 파일 추가
4. Day 1~5 (v1.3 §26) 완료 후 Day 6~7 (본 §9) 진행
5. Day 7 종료 시 evidence 파이프라인 통과 → MVP 전체 완료

---

## 15. 가짜 증거 방지 (재강조)

| 단계 | 가짜 차단 메커니즘 |
|---|---|
| 1. suggest | 벤치마크 패턴 매칭만, 사실 생성 금지 |
| 2. prompt | "사실 조작 금지" 문구를 외부 LLM 프롬프트에 강제 삽입 |
| 3. import | review.md에 누락/중복 표기 + 출처 URL 필수 |
| 4. apply | `proof_source_type` enum 강제 + `watermark_recommended` 자동 |
| 검증 | V-EV-FORB로 단정 표현 차단 + V-EV-HUMAN으로 사람 승인 강제 |

→ v1.3 §3-7, §3-8 ("수치 단정 금지", "30% 향상 같은 표현 금지") 그대로 계승.

---

*작성: 2026-05-24*
*근거: 사용자 워크플로우 요청 (레퍼런스 증거 → 후보 제안 → 생성 프롬프트 → 외부 수집 → proof_assets 적재 → P5 카피 사용)*
*적용 대상: v1.3 잠금본 §7.1 / §22 / §26 / §33 / §34 / §36 / §38 + 신규 §48 / §49*
*충돌 해결 우선순위: v1.3 본 패치 > v1.3 잠금본 (충돌 시 본 패치가 evidence 영역만 덮어씀)*
*승인 → git tag v1.3.1 → Day 6~7 추가*
