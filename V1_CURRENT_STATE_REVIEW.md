# V1_CURRENT_STATE_REVIEW.md

> **목적**: a4-page-flow-factory V1 CLI의 현재 구현 상태를 분석하여, V2/V3 재설계 시 재사용할 자산과 폐기할 구조를 명확히 구분한다.
> **분석 시점**: 2026-05-24
> **분석 대상**: `C:\Users\llorr\dev\repos\a4-page-flow-factory` (main 브랜치, HEAD=e388f4b)
> **분석 범위**: 코드/테스트/문서/git 이력 — **코드 수정·커밋 없음**
> **재설계 참고**: `C:\Users\llorr\Music\홈피 트러\outputs\A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` (v1.3 잠금본)

---

## 0. 한 줄 요약

V1은 **MVP 인프라(폴더 스캐폴드 + PNG 슬라이서 + JSON 캐시)까지 17.5h 계획 중 약 5h(~30%) 완료**한 시점에서 멈췄으며, **도메인 로직(Vision/Classifier/Rewriter/Render/Ethics)은 0줄**이다. v1.3 재설계가 "Prompt Layer + P1~P5 외부 LLM 복사-붙여넣기" 방향으로 축을 바꿨기 때문에, V1의 **5계층 파이프라인·5 HTML 템플릿·Anthropic SDK 직접 호출 가정은 폐기**해야 하지만, **인프라 유틸(cache/slicer)·TDD 구조·windows 환경 학습(L1~L3)·시드 콘텐츠는 그대로 재사용** 가능하다.

---

## 1. 현재 상태 요약

### 1.1 프로젝트 메타

| 항목 | 값 |
|---|---|
| 위치 | `C:\Users\llorr\dev\repos\a4-page-flow-factory` |
| Git 브랜치 | `main` (working tree clean) |
| 커밋 수 | 4 |
| Python | 3.11 (uv 환경) |
| 의존성 lock | 47 패키지 (uv.lock 268KB) |
| MVP 계획 진행률 | M0/M1 완료, M2 부분, M3~M5 미착수 |
| 테스트 | **47/47 PASS** (cache 14 + slicer 19 + input 14) |
| 도메인 코드 라인 | 약 184 lines (cache 106 + slicer 78) |
| 인프라/CLI 코드 | 약 300 lines (init_cmd 266 + __main__ 36) |

### 1.2 Git 커밋 이력

```
e388f4b feat(analyze): add JSON cache with RED→GREEN tests
367ff91 feat(analyze): add PNG slicer with RED→GREEN tests
49e8d4f feat(cli): implement `factory init` (M1-1~M1-6 RED→GREEN)
2bccc19 chore: initialize a4 page flow factory specs and uv project
```

### 1.3 폴더 구조 (실측)

```
a4-page-flow-factory/
├── 문서 8개 (CLAUDE/spec/plan/PRD/DESIGN_SPEC/SAMPLE_FLOW/PAGE_MANIFEST_SPEC/REVIEW_CHECKLIST/README/LessonsLearned)
├── pyproject.toml / uv.lock / .python-version / .gitignore
├── asset_pack/
│   ├── manifest.yaml          ← 시드 (faces/characters/icons/backgrounds/illustrations 빈 슬롯)
│   ├── faces/  characters/  icons/  backgrounds/  illustrations/   ← 모두 빈 폴더
├── docs/future/UIUX_PRD.md    ← Post-MVP 참고 (651 lines)
├── input/                     ← .gitignore 대상이지만 시드 파일 존재
│   ├── business_profile.md    forbidden_phrases.txt   style_notes.md
│   ├── my_evidence/README.md  reference_png/(README.md만)
├── src/factory/
│   ├── __init__.py            (7 lines)
│   ├── __main__.py            (36 lines — typer entry)
│   ├── analyze/
│   │   ├── __init__.py / cache.py (106) / slicer.py (78)
│   │   └── ❌ vision.py / classifier.py / manual.py 미구현
│   ├── cli/
│   │   ├── __init__.py / init_cmd.py (266)
│   │   └── ❌ analyze_cmd / sample_cmd / expand_cmd / doctor 미구현
│   ├── ❌ flow/  synthesis/  render/  ethics/  manifest/  config/  전부 미생성
├── tests/
│   ├── test_cache.py (14 tests) / test_slicer.py (19) / test_input_structure.py (14)
│   └── ❌ test_vision / classifier / rewriter / template / browser / ... 미작성
├── .venv/  .mypy_cache/  .pytest_cache/  .ruff_cache/
```

### 1.4 의존성 (pyproject.toml)

| 그룹 | 패키지 |
|---|---|
| Core | anthropic≥0.40, pillow≥10, pyyaml≥6, jinja2≥3.1, **playwright≥1.49**, typer≥0.15, tenacity≥9, rich≥13.9 |
| Dev | pytest≥8, pytest-cov≥5, ruff≥0.8, mypy≥1.10, types-pyyaml/pillow |

> ⚠️ **`anthropic`은 설치되었으나 코드에서 한 번도 import되지 않음.** v1.3 방향(외부 LLM 복사-붙여넣기 1순위)과 정합성 재검토 필요.

---

## 2. 구현 완료 (PASS)

### M0 — 셋업 ✅
- [x] `pyproject.toml` (deps + tooling 설정 완비)
- [x] `.gitignore` (Python/venv/input/output/.env/IDE/캐시)
- [x] `.python-version` (3.11)
- [x] 9개 문서 (CLAUDE/spec/plan/PRD/DESIGN_SPEC/SAMPLE_FLOW/PAGE_MANIFEST_SPEC/REVIEW_CHECKLIST/README)
- [x] `uv sync` 47 패키지 정상
- [x] 첫 커밋 (`2bccc19`)

### M1 — 입력 폴더 구조 ✅ (M1-7만 보류)
- [x] M1-1: `test_input_structure.py` RED (14 tests, 의도된 실패)
- [x] M1-2: `factory init` 구현 + 13 경로 자동 생성 (14/14 PASS)
- [x] M1-3~M1-6: 시드 파일 6종 자동 포함 (init_cmd의 `SEED_FILES` dict)
- [ ] M1-7: 샘플 PNG 1장 — **보류** (저작권 출처 미결)
- [x] M1-8: 커밋 (`49e8d4f`)

### M2 — PNG 분석 ⚠️ 부분 완료 (인프라만)
- [x] M2-1: `test_slicer.py` RED (19 tests)
- [x] M2-2: `slicer.py` GREEN (PIL 기반, 78 lines)
- [x] M2-3: `test_cache.py` RED (14 tests)
- [x] M2-4: `cache.py` GREEN (SHA256 + sharded JSON, 106 lines, Windows-safe key)

### 누적 테스트 결과 (실측)
```
============================= 47 passed in 0.68s ==============================
```

### 동작하는 CLI 명령
```bash
factory init [--dry-run]
factory --help
```

---

## 3. 미구현 / RED 상태 (FAIL)

### M2 — PNG 분석 (인프라 외)
- [ ] M2-5/6: `manual.py` (수동 JSON 우선) — RED/GREEN 모두 미작성
- [ ] M2-7/8: `vision.py` (Anthropic Vision API + retry + 비용 로깅)
- [ ] M2-9: `analyze_cmd.py` (`factory analyze --set ...`)
- [ ] M2-10: `block_analysis.md` 자동 생성
- [ ] M2-11: ⚠️ 원문 누출 0건 검증 테스트
- [ ] M2-12: 커밋

### M3 — 블록 분류 (전부 미착수)
- [ ] `config/block_types.yaml` (12 타입 + few-shot)
- [ ] `classifier.py` / `flow/abstractor.py`
- [ ] confidence 점수 / 정확도 측정

### M4 — 5~6장 샘플 카피 (전부 미착수)
- [ ] `synthesis/{rewriter,evidence,substitute}.py`
- [ ] `ethics/{forbidden,similarity}.py`
- [ ] `sample_cmd.py`
- [ ] ⚠️ rewriter 원문 전달 X 검증 / forbidden 100% 차단 / 유사도 <30%

### M5 — HTML+PNG 렌더 (전부 미착수)
- [ ] `render/templates_html/T1~T5_*.html` + `_base.html` + `_styles.css`
- [ ] `template_engine.py` / `asset_loader.py` / `browser.py`
- [ ] `ethics/watermark.py`
- [ ] Playwright 통합 + A4 viewport + 한국어 깨짐 방지
- [ ] `score_report.md` / `sample_review.md` 자동 생성
- [ ] `factory doctor` 확장 (Playwright healthcheck)

### Post-MVP (M6~M10) — 전부 미착수
- 40장 확장 / page_manifest CSV builder/validator / 결합 PDF / 다중 set 비교 / Skill화

### CLI 상태 요약
| 명령 | 상태 |
|---|---|
| `factory init` | ✅ 구현됨 |
| `factory doctor` | ❌ 미구현 (README는 언급) |
| `factory analyze` | ❌ 미구현 |
| `factory sample` | ❌ 미구현 |
| `factory expand` | ❌ 미구현 |
| `factory cache clear/stats` | ❌ 미구현 |

---

## 4. 테스트 상태

### 4.1 통과 현황 (실측 2026-05-24)
```
tests/test_cache.py           14 passed
tests/test_input_structure.py 14 passed
tests/test_slicer.py          19 passed
─────────────────────────────────────
TOTAL                         47 passed in 0.68s
```

### 4.2 커버리지 평가
- **인프라 레이어**: cache.py / slicer.py / init_cmd.py — 단위 테스트 완비
- **도메인 레이어**: 0 lines = 0 tests
- **통합/E2E**: 없음
- **회귀 방지**: 인프라 유틸의 부수효과(파일/디렉토리) 테스트 견고

### 4.3 TDD 준수도
- RED → GREEN 사이클 명시적으로 분리되어 커밋됨 (`feat(...): add X with RED→GREEN tests`)
- 모든 PASS 테스트는 의도된 실패 → 구현 → 통과 순서 추적 가능

---

## 5. 재사용 가능 요소 (V2/V3로 가져갈 자산)

### 5.1 코드 자산 (그대로 또는 미미한 수정으로 재사용)

| 자산 | 위치 | 재사용 가치 | 비고 |
|---|---|---|---|
| **`cache.py`** | `src/factory/analyze/cache.py` | ⭐⭐⭐⭐⭐ | 도메인 무관 SHA256+sharded JSON. Windows-safe key. 그대로 사용 가능 |
| **`slicer.py`** | `src/factory/analyze/slicer.py` | ⭐⭐⭐⭐ | PIL 기반 PNG 슬라이서. 외부 LLM에 보낼 청크 만들기에도 동일하게 유효 |
| **Typer `@app.callback()` 패턴** | `__main__.py:25-27` | ⭐⭐⭐⭐ | L1 학습 — 단일 command 함정 회피. V2 CLI에서도 즉시 필요 |
| **`SEED_FILES` dict + idempotent init** | `cli/init_cmd.py` | ⭐⭐⭐ | "사용자 편집 파일 보존" 패턴. V2의 `prompts/` `projects/` 스캐폴드에 그대로 응용 |
| **테스트 픽스처 패턴** | `tests/test_cache.py`, `test_slicer.py` | ⭐⭐⭐⭐ | `_make_png` 헬퍼, `tmp_path` 사용, `parametrize` 활용 — V2 테스트 베이스 |
| **forbidden_phrases.txt 시드** | `input/forbidden_phrases.txt` | ⭐⭐⭐⭐ | 20개 한국어 광고 규제 표현(과장/의료법/비교/가짜후기) — 도메인 자산, 그대로 |
| **business_profile.md 템플릿** | `input/business_profile.md` | ⭐⭐⭐ | 한국어 1인 사업자 입력 양식. V2 프로젝트 입력에 응용 |
| **style_notes.md 템플릿** | `input/style_notes.md` | ⭐⭐ | 톤앤매너 입력 — V2의 P3 STYLE 프롬프트 입력으로 직결 |
| **asset_pack/manifest.yaml 스키마** | `asset_pack/manifest.yaml` | ⭐⭐⭐ | faces/icons/backgrounds 5분류 + source/license/usage 컬럼 — V2 자산 카탈로그 그대로 |

### 5.2 도구/설정 자산

| 자산 | 가치 | 비고 |
|---|---|---|
| **pyproject.toml** (deps + ruff strict + mypy strict + pytest 설정) | ⭐⭐⭐⭐ | `anthropic` 제거 외 거의 그대로 V2로 복제 가능 |
| **ruff lint rules** (`E/W/F/I/B/C4/UP/SIM` + per-file-ignores) | ⭐⭐⭐⭐ | 검증된 규칙 셋 |
| **mypy strict 설정** (disallow_untyped_defs + warn_redundant_casts) | ⭐⭐⭐ | V2도 동일 적용 권장 |
| **uv workflow** (uv sync / uv run) | ⭐⭐⭐⭐ | Python 환경 도구 결정 — 유지 |
| **.gitignore** (Python/venv/input/output/.env/IDE/캐시) | ⭐⭐⭐⭐ | 그대로 |

### 5.3 운영 학습 (LessonsLearned.md)

| ID | 학습 | V2 적용 가치 |
|---|---|---|
| **L1** | Typer 단일 command 자동 root 최적화 → `@app.callback()` 강제 | ⭐⭐⭐⭐⭐ 즉시 적용 필수 |
| **L2** | Windows cp949 콘솔의 em-dash UnicodeEncodeError → ASCII 안전 출력 원칙 | ⭐⭐⭐⭐⭐ V2 모든 typer help/CLI 메시지에 적용 |
| **L3** | `PYTHONIOENCODING=utf-8` 필요 (한국어 console.print) | ⭐⭐⭐⭐⭐ V2 README/doctor에 그대로 |

### 5.4 문서 자산 (부분 재사용)

| 문서 | 재사용 가능 부분 | 폐기 부분 |
|---|---|---|
| `PRD.md` §6 입력 데이터 정의 | business_profile/style/forbidden 구조 | 5계층 파이프라인 가정 |
| `PRD.md` §10 R1~R7 리스크 표 | R3(HTML 깨짐) / R6(가짜 증거) / R7(Playwright) | R1(Vision 비용) — v1.3은 외부 LLM 사용 |
| `DESIGN_SPEC.md` §11 D1~D15 위험 표 | D8(워터마크) / D11(GPT 얼굴 차단) / D14(폰트) / D15(한국어 keep-all) | D2(Vision 분류) / D5(Vision 캐싱) — 외부 LLM 전환 |
| `DESIGN_SPEC.md` 부록 A 12 블록 타입 | 도메인 지식으로 v1.3 P2 CLASSIFY 프롬프트 시드에 사용 | "코드 분류기 구현" 부분은 폐기 |
| `REVIEW_CHECKLIST.md` | 사용자 검수 게이트 항목 | 자동 채점 ≥80 산식은 v1.3 재정의 필요 |
| `docs/future/UIUX_PRD.md` (651 lines) | v1.3과 비교 검토 필요 | "Post-MVP" 가정이 v1.3에서 변경됨 |

---

## 6. 버릴 요소 (V2에 가져가지 말 것)

### 6.1 아키텍처 가정 (전면 폐기)

| 폐기 대상 | 이유 |
|---|---|
| **5계층 단방향 파이프라인** (`Input → Analysis → Synthesis → Render → Validation`) | v1.3은 "Prompt Layer + P1~P5 외부 LLM 복사-붙여넣기"로 축이 바뀜. 파이프라인 자체가 다른 구조 |
| **Anthropic SDK 직접 호출** (`vision.py` 가정) | v1.3 큰원칙 #1: "외부 GPT/Claude/Gemini 웹에 복사해서 쓰는 흐름이 MVP의 1순위" |
| **`factory analyze --set` 명령 디자인** | "PNG → Vision API → 블록 JSON"이 자동 파이프라인 가정. v1.3은 프롬프트 생성·복사가 1차 산출물 |
| **`factory sample` / `factory expand` 명령 디자인** | 동일 이유. v1.3은 프로젝트/프롬프트/블록/이미지 슬롯/채널의 사용자 UI 단위로 재정의 |
| **5개 고정 HTML 템플릿** (T1~T5) | v1.3에서 명시적 확정 안 됨. 재평가 필요 (v1.3 §4 3층 scope 구조와 정합성 재검토) |
| **`config/block_types.yaml` 12 타입 고정** | v1.3 P2 CLASSIFY는 프롬프트로 분류 — YAML+코드 분류기 이중 구조는 SSOT 위반 |

### 6.2 모듈 구조 (재설계 필요)

| 폐기 대상 (`DESIGN_SPEC.md §3`) | 대체 방향 |
|---|---|
| `src/factory/flow/abstractor.py` | v1.3 P2 프롬프트 산출물로 흡수 |
| `src/factory/synthesis/{rewriter,evidence,substitute}.py` | v1.3 P3 STYLE / P5 CHANNEL 프롬프트로 흡수 |
| `src/factory/render/{template_engine,asset_loader,browser}.py` | Playwright 부분은 살리되, `render/` 폴더 구조는 v1.3 ASSET 슬롯 기반으로 재설계 |
| `src/factory/ethics/{forbidden,similarity,watermark}.py` | forbidden 검사 자체는 살리되, "rewriter 원문 누출 검증"은 외부 LLM 사용으로 사라짐 |
| `src/factory/manifest/{builder,validator}.py` | Post-MVP였고 v1.3은 page_manifest.csv 개념 자체를 안 씀 |

### 6.3 워크플로우/검증 가정 (재정의)

| 폐기 대상 | 이유 |
|---|---|
| "5~6장 샘플 게이트" (PRD §11 7개 항목) | 5~6장 단위 자체가 v1.3에서 재정의될 가능성 |
| "40장 확장 조건" (PRD §12) | Post-MVP였고 v1.3에서 재정의 |
| "자동 점수 ≥ 80" (REVIEW_CHECKLIST 산식) | 자동화 가정. v1.3은 수동 검수가 더 무거워질 가능성 |
| `plan.md` M2~M5 체크박스 전체 | 5계층 파이프라인 기반이라 v1.3과 호환 안 됨 |

### 6.4 인프라/문서

| 폐기 대상 | 이유 |
|---|---|
| `anthropic>=0.40.0` 의존성 | 한 줄도 import되지 않음. v1.3 "외부 웹 복사-붙여넣기" 방향이면 영구 미사용 |
| `tenacity>=9.0.0` (Vision API retry 가정) | API 직접 호출 안 하면 불필요 |
| `PAGE_MANIFEST_SPEC.md` (Post-MVP 40장 매니페스트) | v1.3 범위 밖 |

---

## 7. 새 프로젝트 설계 시 주의점

### 7.1 V1이 멈춘 진짜 이유 (재설계의 핵심 교훈)

V1은 **47/47 테스트 통과**라는 표면적 성공에도 불구하고:
- **인프라 5h** 끝낸 시점에 **도메인 12h가 통째로 남아 있고**
- 그 도메인 12h의 가정(5계층 파이프라인 + Anthropic SDK 직접 호출)이 **v1.3에서 폐기**됨

→ **교훈**: V2 착수 전에 **"외부 LLM 복사-붙여넣기"가 진짜 MVP의 흐름인지** PRD 수준에서 잠금하고, P1~P5 프롬프트 자체를 **테스트 가능한 산출물**로 정의해야 한다. 그렇지 않으면 V2도 cache/slicer만 만들고 또 멈춘다.

### 7.2 V2 첫 PLAN.md에 반드시 들어갈 항목

1. **P1~P5 프롬프트 산출물의 검증 방법** (외부 LLM 응답 모킹 또는 골든 출력 비교)
2. **"Pack" 단어를 사용자 UI에서 노출 금지** 검증 (린트 룰 또는 grep 테스트)
3. **scope 3층 frontmatter 스키마** 단위 테스트
4. **Windows cp949 가정** (L1/L2/L3 즉시 적용 — `@app.callback()` + ASCII help + PYTHONIOENCODING 가이드)
5. **`anthropic` 의존성 제거** (또는 명시적 보류 사유 기록)
6. **cache/slicer 재사용 결정** — `src/factory_v2/utils/` 형태로 복제할지, 별도 패키지로 분리할지 1주차에 결정

### 7.3 폴더 구조 결정 시 주의

- V1처럼 **5계층 모듈을 미리 만들지 말 것** — 한 번도 import 안 된 빈 모듈이 양산됨
- v1.3 §4 "폴더는 평평하게, scope는 frontmatter" 원칙을 따른다면, `src/`도 깊은 계층 없이 단일 폴더 + 프롬프트/스코프는 메타데이터로 분리
- `input/` `output/` `asset_pack/` 시드 구조는 V1 패턴 그대로 차용 가능 (`SEED_FILES` dict 재활용)

### 7.4 테스트 전략

- V1의 TDD RED→GREEN 커밋 분리 패턴은 **유지**
- 단, V2는 **인프라 테스트 비율을 줄이고** P1~P5 프롬프트 출력에 대한 **골든 픽스처 테스트**가 핵심
- E2E: "프로젝트 만들기 → P1 EXTRACT 프롬프트 생성 → 외부 LLM 응답 붙여넣기(고정 픽스처) → P2 CLASSIFY → ... → Playwright PNG" 1회 통과를 MVP 게이트로

### 7.5 문서 운영 (Single Source of Truth)

- V1은 9개 문서(spec/plan/PRD/DESIGN_SPEC/SAMPLE_FLOW/PAGE_MANIFEST/REVIEW/README/CLAUDE) → **너무 많음**
- v1.3 잠금본 §0이 이미 `SYSTEM_TECH.md > PLAN.md > CLAUDE.md` 3종으로 압축을 명시 → 이 원칙을 V2 초기부터 강제
- V1의 `DESIGN_SPEC.md` 부록 A(12 블록 타입), 부록 B(CLI 명령 사양)는 **분리된 reference 파일**로 가져갈지 / SYSTEM_TECH에 흡수할지 1주차에 결정

### 7.6 위험 신호 (즉시 HALT 트리거)

V2에서 다음이 발생하면 **인프라 함정에 다시 빠진 것**:
- 1주차 끝에 cache.py / slicer.py만 옮겨졌고 P1 프롬프트 산출물 0개
- `anthropic` 또는 다른 LLM SDK가 의존성에 다시 추가됨 (v1.3 큰원칙 #1 위반)
- 사용자 UI 단어에 "Pack" 노출 (v1.3 §2.2 위반)
- 문서가 4개를 초과 (SYSTEM_TECH/PLAN/CLAUDE 외)

---

## 8. 부록 — 핵심 파일 라인 카운트

| 파일 | Lines | 상태 |
|---|---|---|
| `src/factory/__init__.py` | 7 | ✅ |
| `src/factory/__main__.py` | 36 | ✅ (init만 등록) |
| `src/factory/analyze/__init__.py` | 1 | ✅ |
| `src/factory/analyze/cache.py` | 106 | ✅ (14 tests PASS) |
| `src/factory/analyze/slicer.py` | 78 | ✅ (19 tests PASS) |
| `src/factory/cli/init_cmd.py` | 266 | ✅ (14 tests PASS) |
| `src/factory/cli/__init__.py` | 1 | ✅ |
| **합계 (도메인 코드)** | **495 lines** | M0/M1 + M2 인프라 |
| `tests/test_cache.py` | 230 | 14 PASS |
| `tests/test_slicer.py` | 225 | 19 PASS |
| `tests/test_input_structure.py` | 72 | 14 PASS |
| **합계 (테스트 코드)** | **527 lines** | |

| 문서 | Lines |
|---|---|
| `PRD.md` | 218 |
| `plan.md` | 290 |
| `spec.md` | 105 |
| `CLAUDE.md` | 175 |
| `DESIGN_SPEC.md` | 247 |
| `SAMPLE_FLOW.md` | 293 |
| `PAGE_MANIFEST_SPEC.md` | 286 |
| `REVIEW_CHECKLIST.md` | 356 |
| `README.md` | 200 |
| `LessonsLearned.md` | 146 |
| `docs/future/UIUX_PRD.md` | 651 |
| **문서 합계** | **3,167 lines** |

> 코드(495) : 테스트(527) : 문서(3,167) ≈ **1 : 1 : 6.4**
> 문서가 코드의 6배 이상 — V2에서는 SYSTEM_TECH/PLAN/CLAUDE 3종으로 압축 필요.

---

## 9. 결론 — V2 착수 결정 매트릭스

| 옵션 | 장점 | 단점 |
|---|---|---|
| **A. V1 폴더에서 계속 진행** | 인프라 그대로, 즉시 시작 | v1.3 방향 변경분 vs V1 가정 충돌 — 매 커밋마다 정리 필요 |
| **B. 새 폴더 V2 신규 생성, 자산만 복사** ⭐권장 | v1.3 잠금본을 SYSTEM_TECH로 깨끗하게 시작, 도메인 가정 충돌 없음 | cache/slicer/init_cmd 등 ~500 lines 수동 복사 + 일부 수정 |
| **C. V1 폴더에서 새 브랜치 + 재구조** | git 이력 유지 | 5계층 폴더 잔재가 PR마다 노이즈 |

**권장: B** — v1.3 재설계 문서가 이미 "새 폴더 v2의 spec/PRD/plan 통합본"이라 명시. 본 리뷰의 §5 재사용 자산 목록을 체크리스트로 사용해 V2 1주차에 일괄 이식.

---

*작성: 2026-05-24*
*근거: 실제 코드/테스트/git/문서 검증 (47/47 PASS 확인, 4 커밋 검증, 9 문서 + LessonsLearned 정독)*
*다음 액션: 본 문서를 v1.3 잠금본과 함께 V2 SYSTEM_TECH.md 작성 시 참조*
