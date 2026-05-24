# a4-page-flow-factory-v2 핸드오프 패키지

> 이 폴더는 새 폴더 `a4-page-flow-factory-v2`에서 처음부터 개발을 시작하기 위한 **단일 핸드오프 패키지**다.
> 다른 AI / 다른 CLI / 다른 PC 어디로든 이 폴더를 통째로 가져가서 사용하면 된다.

---

## 0. 한 줄 요약

**`A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md`만 읽으면 된다.** 나머지 2개는 참고용 이력이다.

---

## 1. 파일 목록

| 파일 | 줄수 | 역할 | 누가 읽어야 하나 |
|---|---|---|---|
| **`A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md`** | 1360 | **최종 단일 잠금본** (spec + PRD + plan) | **다음 Captain AI 필수** |
| **`A4_PAGE_FLOW_FACTORY_V1_3_1_PATCH_EVIDENCE.md`** | ~450 | **v1.3.1 패치** — evidence 파이프라인 (Day 6~7 추가) | Day 6 진입 시 |
| `V1_CURRENT_STATE_REVIEW.md` | 372 | V1 진단 리포트 (왜 새로 만드는지) | 참고용 |
| `V2_FULL_REDESIGN_DRAFT.md` | 967 | V2 재설계 중간 draft | 참고용 (v1.3에 흡수됨) |
| `HANDOFF_README.md` | (본 문서) | 이 패키지 사용 안내 | 모두 |

---

## 2. 어디서 어떻게 쓰나

### 시나리오 A — 다른 AI / 다른 CLI에서 새 폴더로 개발 시작

```
1. 새 PC / 새 폴더에 본 폴더(v2_handoff) 통째로 복사
2. 다른 AI 세션 시작
3. 첫 프롬프트로 아래 §3 "Captain 첫 프롬프트 (복붙용)" 그대로 붙여넣기
4. AI가 A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md를 읽고 Day 1부터 시작
```

### 시나리오 B — 본인이 직접 개발

```
1. 새 폴더 a4-page-flow-factory-v2 생성
2. A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md를 가장 먼저 읽기
3. §26 "1주차 5일 구현 계획"의 Day 1부터 순서대로
4. §33 "최종 하지 말아야 할 것" 13개를 매 커밋 전에 체크
```

---

## 3. Captain 첫 프롬프트 (복붙용)

다른 AI 세션에 그대로 복사해서 붙여넣으면 된다.

```
너는 a4-page-flow-factory-v2 구현자다.

전제:
- 본 세션 시작 시 A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md를 먼저 읽고 시작한다.
- 새 폴더 a4-page-flow-factory-v2를 부모 디렉토리에 생성한다.
- 기존 V1 폴더(a4-page-flow-factory)가 있다면 절대 수정하지 않는다.
- V1에서 가져올 자산은 §43 표에 명시되어 있다 (cache.py / slicer.py / __main__.py / test_*).

이번 세션 작업 (Day 1만):
1. 새 폴더 a4-page-flow-factory-v2 생성
2. pyproject.toml (anthropic / tenacity 제거, ruff/mypy strict 유지)
3. .gitignore
4. SYSTEM_TECH.md (본 문서 v1.3을 500줄 이하로 압축)
5. PLAN.md (Day 1~5 체크박스만, 300줄 이하)
6. CLAUDE.md (AI 규칙, 200줄 이하)
7. uv sync
8. src/factory/__main__.py — Typer + @app.callback() (L1 적용)
9. factory --help가 Windows cp949에서 깨지지 않음 확인 (L2)
10. factory doctor 빈 명령 (L3 안내)
11. V1에서 cache.py / slicer.py / test_cache.py / test_slicer.py 선별 복사
12. tests/conftest.py에 _make_png 헬퍼 승격
13. 33/33 PASS 확인

금지 (v1.3 §33):
- Day 2 이후 작업 손대기 금지
- anthropic / openai / google-genai / tenacity 의존성 추가 금지
- 4번째 메타 문서 생성 금지
- V1 폴더 수정 금지
- 사용자 UI에 "Pack" 단어 노출 금지
- 이미지 생성 AI로 글자 박힌 카드뉴스 만들기 금지
- DB / Supabase / 자동게시 / 자체 변경이력 시스템 금지

종료 조건:
- uv run pytest tests/test_cache.py tests/test_slicer.py → 33/33 PASS
- uv run factory --help cp949에서 깨짐 없음
- PLAN.md의 Day 1 체크박스만 [x]
- HALT 후 인수인계 메모 작성
```

---

## 4. v1.3 잠금본 핵심 (5줄 요약)

1. **새 폴더에서 처음부터** 만든다 (V1 폴더 수정 금지)
2. **반자동 프롬프트 우선** — CLI는 P1~P5 `.md` 파일만 만들고, 외부 LLM 웹(GPT/Claude/Gemini)에 사용자가 붙여넣음. **LLM SDK 의존성 0개**
3. **HTML/CSS → Playwright PNG**가 최종물. 이미지 생성 AI로 카드뉴스 안 만듦
4. **Day 5 게이트 단 1개**: `instagram_portrait` PNG 1장 + 품질 체크 **12/12** 통과
5. **사용자 UI 단어 6개만**: 프로젝트 / 프롬프트 / 블록 / 디자인 블록 / 이미지 슬롯 / 채널 (Pack 금지)

---

## 5. 1주차 5일 (요약)

| Day | 작업 | 완료 조건 |
|---|---|---|
| 1 | 셋업 + V1 자산 이식 | 새 폴더, CLI help cp949 안전, 33/33 PASS |
| 2 | enums / blocks schema + 자료실 시드 | `blocks.jsonl` / `design_blocks.jsonl` 검증 |
| 3 | P3 + P5 프롬프트 + 골든 픽스처 | 7개 검증축(V-STRUCT/V-DETERM/V-REF/V-ENUM/V-FORB/V-GOLD) 통과 |
| 4 | copy import + sample HTML | instagram_portrait HTML 1장 |
| 5 | Playwright PNG 1장 + E2E | PNG 1장 + **12/12 품질 체크** |

---

## 6. 위험 신호 (즉시 HALT)

| 신호 | 대응 |
|---|---|
| Day 5까지 PNG 1장 미생성 | 인프라 함정 재발 → 범위 축소 |
| `anthropic` 또는 LLM SDK 의존성 추가 | MVP 기본 가정 위반 → 즉시 제거 |
| 4번째 메타 문서 생성 | 문서 폭발 → 3개로 복귀 |
| output_profile 10개 한 번에 구현 | 범위 과대 → instagram_portrait만 |
| 사용자 UI에 "Pack" 단어 등장 | v1.3 §4.2 위반 → 즉시 교체 |
| V1 폴더 수정 | v1.3 §33-1 위반 → 즉시 롤백 |

---

## 7. 폴더 구조 예고 (다음 세션이 만들 것)

```
a4-page-flow-factory-v2/         ← 새 폴더 (다음 세션이 생성)
├── pyproject.toml / .gitignore / README.md
├── SYSTEM_TECH.md               ← v1.3 압축본 ≤500 lines
├── PLAN.md                       ← Day 1~5 체크박스 ≤300 lines
├── CLAUDE.md                     ← AI 규칙 ≤200 lines
├── docs/
│   └── A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md  ← 본 파일 그대로 복사
├── src/factory/
│   ├── __main__.py / __init__.py
│   ├── schema/{enums,blocks,design_blocks,manifest,meta}.py
│   ├── utils/{cache,ascii_safe,encoding}.py    ← V1 cache 이식
│   ├── library/{loader,seed,routing}.py
│   ├── extract/{slicer,prompt_builder,importer,review}.py  ← V1 slicer 이식
│   ├── prompts/{base,p1_extract,p2_classify,p3_style,p4_image_elem,p5_channel}.py
│   ├── render/{html_builder,browser,screenshot,quality_check}.py
│   ├── ai/{adapter_base}.py     ← Post-MVP placeholder만
│   └── cli/*.py
├── tests/
├── content-learning-library/    ← 자료실 (구조 commit, 데이터 ignore)
├── projects/[slug]/prompts/      ← 생성 프롬프트 .gitignore
├── extract/                     ← .gitignore
└── output/[slug]/                ← .gitignore
```

---

## 8. 출처 추적

본 패키지의 모든 결정은 아래 4개 출처 중 하나로 추적 가능:

| 출처 | 본 패키지에서 위치 |
|---|---|
| 사용자 외부 지시 | (대화 메시지) |
| v1.3 합의안 §1~§33 | `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` §1~§33 |
| v1.2 잠금본 | `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` §34~§42 (계승 표시) |
| V2 재설계 draft | `V2_FULL_REDESIGN_DRAFT.md` (전체) |
| V1 진단 | `V1_CURRENT_STATE_REVIEW.md` (전체) |

충돌 해결 우선순위:
**사용자 외부 지시 > v1.3 합의안 > v1.2 잠금본 > V2 draft > V1 진단**

---

## 9. 다음 액션 체크리스트

- [ ] 본 폴더(`v2_handoff/`) 통째로 USB / 클라우드 / 새 PC로 이동
- [ ] 새 작업 환경에서 새 AI / 새 CLI 세션 시작
- [ ] §3 "Captain 첫 프롬프트"를 그대로 붙여넣기
- [ ] AI가 `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` 읽었는지 확인
- [ ] Day 1 진행 → 33/33 PASS 확인 → HALT
- [ ] Day 2부터는 새 세션에서 진행 (한 세션 = 한 Day 원칙)

---

*생성: 2026-05-24*
*패키지 위치: `c:\Users\llorr\Music\홈피 트러\v2_handoff\`*
*총 4개 파일 (본 README + 3 MD 잠금본/리뷰/draft)*
