# A4 Page Flow Factory — v1.3 합의안 원문 (Verbatim)

> **이 문서의 역할**: 사용자가 직접 작성·합의한 v1.3 §1~§33 원문 그대로 저장.
> **편집·합성·요약 금지**. 다른 문서(예: `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md`)는 본 원문 + v1.2/V2/V1 자료를 통합한 잠금본이다.
> **충돌 시 우선순위**: 본 원문(v1.3 합의안) > 통합 잠금본
> **단일 진실 원본**: P1~P5 / Prompt Layer / scope / design_blocks / image_slots / 9 채널 / 10 output_profile / 12 PNG 체크 / 13 금지 사항의 **최종 권위는 본 원문**.

---

## 1. 컨텍스트

- 이전 세션에서 v1.0 → v1.1 → v1.2까지 설계 잠금본을 만들어왔다.
- 기존 V1 CLI는 M2-4까지 47/47 PASS 상태였다.
- 그러나 CLI 진단 결과, 47개 테스트는 대부분 셋업 / 유틸 / 캐시 중심이었다.
- 도메인 로직은 사실상 0줄에 가까웠다.
- factory analyze / sample / expand / doctor 같은 핵심 명령은 미구현 상태였다.
- 따라서 기존 V1 CLI를 계속 패치하지 않고, 새 폴더에서 v1.3 기반으로 처음부터 시작하기로 결정했다.
- 새 폴더 이름은 `a4-page-flow-factory-v2`다.
- 기존 V1은 참고자료이자 일부 코드 재사용 창고로만 사용한다.
- v1.2 잠금본의 "M2-5 GREEN 후 계속" 흐름보다, 사용자의 "새 폴더에서 처음부터" 지시가 우선한다.

---

## 2. V1 진단 결과 요약

기존 V1에서 재사용 가능한 것:

| 항목 | 판단 |
|---|---|
| cache.py | 재사용 후보. Windows-safe SHA256 sharded JSON 구조 |
| slicer.py | 재사용 후보. PIL 기반 PNG 슬라이싱 구조 |
| __main__.py 패턴 | 재사용 후보. Typer CLI 엔트리 패턴 |
| Typer @app.callback() 학습 | V2 1일차 반영 |
| cp949 / PYTHONIOENCODING 학습 | Windows 환경 안정성을 위해 반영 |
| SEED_FILES idempotent 패턴 | 참고 가능 |
| forbidden_phrases.txt 시드 | 참고 가능 |
| business_profile.md 템플릿 | 참고 가능 |
| ruff / mypy strict 설정 | 유지 가능 |
| anthropic 직접 의존성 | MVP에서는 제거 |

기존 V1에서 폐기할 것:

| 항목 | 판단 |
|---|---|
| 기존 5계층 파이프라인 가정 | 폐기 |
| Anthropic SDK 직접 호출 | MVP 제외, Phase 2 adapter로 분리 |
| 5개 고정 HTML 템플릿 | 폐기 |
| block_types.yaml 12 타입 고정 | 폐기, enums.py 단일 정의로 변경 |
| flow / synthesis / render / ethics / manifest 기존 구조 | 새 구조에 맞게 재설계 |
| 기존 M2-5 GREEN 후 계속 개발 흐름 | 폐기 |
| 기존 V1 폴더 직접 패치 | 금지 |

가장 중요한 교훈:
V1이 멈춘 이유는 인프라 구현 후 도메인 구현 12h 이상이 남아 있었고,
그 도메인 가정 자체가 v1.3 방향과 맞지 않았기 때문이다.

따라서 v1.3에서는 먼저 P1~P5 Prompt Layer 산출물과 검증 방식을 잠그고,
그다음 아주 작은 MVP 게이트를 통과시킨다.

---

## 3. v1.3 큰 원칙

확정 원칙:

1. 반자동 프롬프트 우선
2. 외부 GPT / Claude / Gemini 웹에 복사해서 사용하는 흐름이 MVP 1순위
3. HTML/CSS → Playwright PNG로 최종 산출물 렌더
4. 이미지 생성 AI로 글자 박힌 카드뉴스를 만들지 않음
5. 이미지 생성 AI는 배경 / 일러스트 / 아이콘 슬롯용으로만 검토
6. 자동게시 / OAuth / DB / 이미지 API 자동 생성은 Post-MVP
7. 수치 단정 금지
8. "30% 향상" 같은 검증되지 않은 표현 금지
9. 진짜 파인튜닝 도입 안 함
10. 벡터 DB는 당분간 도입 안 함
11. MVP는 작게 유지
12. Day 5까지 instagram_portrait PNG 1장 생성이 단일 게이트

---

## 4. Prompt Layer / Prompt Pack 정의

### 4.1 Prompt Layer

Prompt Layer는 시스템 전체에서 프롬프트를 1급 산출물로 다루는 설계 축이다.

즉, 이 프로젝트는 단순히 카피를 만드는 CLI가 아니라,
다음 산출물을 체계적으로 만든다.

- 추출 프롬프트
- 분류 프롬프트
- 스타일 프롬프트
- 이미지 요소 프롬프트
- 채널 변환 프롬프트
- HTML/CSS 렌더용 manifest
- 최종 PNG

### 4.2 Prompt Pack

Prompt Pack은 프로젝트별 프롬프트 저장 묶음이다.

단, 사용자 UI에는 "Pack"이라는 단어를 노출하지 않는다.

사용자 화면 용어:

| 내부 용어 | 사용자 화면 용어 |
|---|---|
| Prompt Pack | 프로젝트 |
| Prompt Layer | 프롬프트 |
| text block | 블록 |
| design block | 디자인 블록 |
| image slot | 이미지 슬롯 |
| channel adapter | 채널 |

Pack은 내부 저장 / export / import 단위로만 사용한다.

---

## 5. P1~P5 프롬프트 구조

| Layer | 이름 | 역할 | MVP 포함 |
|---|---|---|---|
| P1 | P1_EXTRACT | 이미지 / PDF / 문서에서 텍스트 추출용 프롬프트 생성 | 포함 |
| P2 | P2_CLASSIFY | 추출 텍스트를 블록으로 분류 | 포함 |
| P3 | P3_STYLE | UI/UX 스타일 프롬프트 생성 | 포함 |
| P4 | P4_IMAGE_ELEM | 이미지 요소별 프롬프트 생성 | 포함 |
| P5 | P5_CHANNEL | 채널 변환 프롬프트 생성 | 포함 |

각 Layer는 API 호출이 아니라 `.md` 프롬프트 파일을 출력하는 것이 MVP의 핵심이다.

---

## 6. 3층 범위 구조

범위는 3층으로만 구분한다.

| scope | 의미 |
|---|---|
| global | 프로젝트 전체에 적용 |
| block | 특정 텍스트 / 디자인 블록에 적용 |
| asset | 특정 이미지 슬롯 / 시각 요소에 적용 |

중요:
- P1~P5 × 3층 매트릭스 폴더를 만들지 않는다.
- 폴더는 평평하게 유지한다.
- 각 `.md` 파일의 frontmatter에 `scope` 필드로 처리한다.

frontmatter 예시:

```yaml
---
prompt_type: P3_STYLE
scope: block
applies_to: problem
model_hint: gpt
---
```

---

## 7. 최종 폴더 구조

### 7.1 content-learning-library

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
│   ├── blocks.jsonl
│   └── design_blocks.jsonl
├── benchmarks/
│   ├── landing_pages/
│   ├── cardnews/
│   ├── threads/
│   └── emails/
├── recipes/
│   └── recipes.md
├── prompts/
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

### 7.2 projects 구조

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

---

## 8. blocks.jsonl 스키마

형식:
JSONL. 한 줄에 한 객체. JSON 배열 아님.

| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 짧은 랜덤 고유값 |
| block_type | enum | hook / problem / pain_point / solution / benefit / proof / before_after / objection / cta / trust / feature / comparison |
| text | string | 블록 본문 |
| source_type | enum | own / benchmark / user_added |
| source_product | string | 제품 slug |
| source_file | string | 파일 경로 |
| source_anchor | string | 원문 내부 위치 |
| industry | string | snake_case |
| tone | enum | voice/ 파일명과 동일 |
| channel | enum | channels/ 파일명과 동일 |
| logic_role | enum | logic/ 파일명과 동일 |
| tags | string[] | snake_case |
| status | enum | draft / approved / archived |
| reuse_note | string | 재사용 메모 |
| created_at | ISO8601 | 생성일 |
| updated_at | ISO8601 | 수정일 |

사용하지 않는 필드:

- quality_score
- 날짜형 ID

---

## 9. design_blocks.jsonl 스키마

형식:
JSONL. 한 줄에 한 객체.

| 필드 | 타입 | 설명 |
|---|---|---|
| id | string | 짧은 랜덤 고유값 |
| design_block_type | enum | icon / illustration / hero_image / chart / divider / cta_button / tag / card_layout / color_palette / typography |
| source_image | string | 레퍼런스 이미지 경로 |
| crop_box | object | {x, y, w, h} |
| description | string | 사람이 쓴 짧은 설명 |
| style_prompt | string | 이 블록의 스타일을 묘사하는 프롬프트 |
| image_gen_prompt | string | 외부 GPT / Midjourney / 이미지 생성툴용 이미지 생성 프롬프트 |
| tone | enum | voice/와 동일 |
| palette | string[] | hex 색상 코드 |
| tags | string[] | snake_case |
| status | enum | draft / approved / archived |
| linked_text_block | string | 짝이 되는 텍스트 블록 id. 선택 |
| created_at | ISO8601 | 생성일 |
| updated_at | ISO8601 | 수정일 |

---

## 10. image_slots 스키마

page_manifest.json에 포함되는 이미지 슬롯 구조다.

| 필드 | 타입 | 설명 |
|---|---|---|
| slot_id | string | 슬롯 고유 ID |
| elem_type | enum | icon / illustration / hero_image / chart_visual / pattern / divider |
| position | object | {x, y, w, h}. 0~1 비율 |
| linked_design_block | string | design_blocks의 id. 선택 |
| prompt_file | string | P4 프롬프트 .md 경로 |
| generated_image | string/null | 사용자가 업로드한 이미지 경로 |
| status | enum | prompt_only / pending / ready / approved |
| required | boolean | 필수 / 선택 구분 |

원칙:

- 사용자가 외부 GPT / Gemini / 이미지 생성툴에서 만든 이미지를 drag&drop으로 슬롯에 배치한다.
- 이미지 API 자동 생성은 MVP 제외다.
- 슬롯별 프롬프트를 표시한다.
- 슬롯은 required 여부를 가진다.

---

## 11. 채널 구조

채널은 common_source + channel_profiles 방식으로 설계한다.

### 11.1 본질 4개

| 본질 | 설명 |
|---|---|
| long_form_text | 긴 글 중심 |
| short_form_text | 짧은 글 중심 |
| card_visual | 카드뉴스 / 이미지 중심 |
| mixed | 텍스트와 이미지 혼합 |

### 11.2 플랫폼 어댑터 9개

| adapter | 본질 |
|---|---|
| instagram_cardnews | card_visual |
| instagram_story | card_visual |
| instagram_post | mixed |
| linkedin_post | mixed |
| medium_article | long_form_text |
| pinterest_pin | card_visual |
| x_thread | short_form_text |
| facebook_post | mixed |
| threads | short_form_text |

원칙:

- 9개 채널을 완전히 따로 만들지 않는다.
- 본질 4개 + 플랫폼 어댑터 구조로 만든다.
- 채널 변환은 별도 단계가 아니라 미리보기 화면의 탭으로 제공한다.

---

## 12. output_profile 규격

| profile | width | height | 용도 |
|---|---|---|---|
| a4_document | 1240 | 1754 | A4 세로 |
| instagram_square | 1080 | 1080 | 인스타 정사각 |
| instagram_portrait | 1080 | 1350 | 인스타 4:5 카드뉴스 |
| story_9_16 | 1080 | 1920 | 스토리 / 릴스 |
| threads_image | 1080 | 1350 | Threads |
| linkedin_post | 1080 | 1080 | LinkedIn |
| pinterest_pin | 1000 | 1500 | Pinterest 2:3 |
| x_card | 1200 | 675 | X 이미지 카드 |
| facebook_post | 1200 | 630 | Facebook |
| medium_header | 1200 | 675 | Medium 헤더 |

MVP 우선순위:

1. instagram_portrait
2. a4_document
3. instagram_square
4. story_9_16
5. threads_image
6. 나머지 profile

---

## 13. 워크플로우 9단계

1. 자료 입력
2. 추출 확인 UI6
3. 블록 분류
4. 자료실 저장
5. 프롬프트 설계
6. 레시피 선택
7. 미리보기
8. 품질 검사
9. 최종 산출

| 단계 | 설명 | MVP |
|---|---|---|
| 1. 자료 입력 | 이미지 / PDF / 문서 / 텍스트 입력 | 포함 |
| 2. 추출 확인 UI6 | 추출 텍스트와 추출 프롬프트 확인 | 포함 |
| 3. 블록 분류 | 텍스트 블록 / 디자인 블록 분리 | 포함 |
| 4. 자료실 저장 | blocks / design_blocks 저장 | 포함 |
| 5. 프롬프트 설계 | P1~P5 프롬프트 편집 | 포함 |
| 6. 레시피 선택 | 채널 / 목적 / 톤 선택 | 포함 |
| 7. 미리보기 | 채널 탭 / 이미지 슬롯 탭 | 포함 |
| 8. 품질 검사 | PNG 품질 체크 | 포함 |
| 9. 최종 산출 | HTML / PNG / prompt files 출력 | 포함 |

---

## 14. UI 설계 핵심 화면

### 14.1 UI6 추출 화면

4-pane 구조:

| 영역 | 역할 |
|---|---|
| 페이지 썸네일 | 입력 이미지 / 문서 페이지 목록 |
| 추출된 텍스트 | 외부 LLM 또는 OCR로 추출된 텍스트 확인 |
| 추출 프롬프트 | P1_EXTRACT 프롬프트 확인 / 복사 |
| 추출 품질 | 누락 / 중복 / 작은 글씨 / 표 검수 |

모바일:

- 4-pane 대신 탭 전환 구조

추출 프롬프트 패널 포함 요소:

- 모델 토글: Claude / GPT / Gemini
- 선택한 모델용 프롬프트로 복사
- 프롬프트 복사
- 기본 템플릿으로 복원

### 14.2 블록 분류 화면

탭:

- 텍스트 블록
- 디자인 블록

텍스트 블록:

- hook
- problem
- pain_point
- solution
- benefit
- proof
- before_after
- objection
- cta
- trust
- feature
- comparison

디자인 블록:

- icon
- illustration
- hero_image
- chart
- divider
- cta_button
- tag
- card_layout
- color_palette
- typography

### 14.3 프롬프트 설계 화면

- P1~P5 파일을 평평하게 나열한다.
- 파일 클릭 시 편집한다.
- 모델별 토글을 제공한다.
- 사용자 화면에서는 "프로젝트 / 프롬프트 / 블록 / 이미지 슬롯 / 채널" 용어만 사용한다.

### 14.4 미리보기 화면

탭:

- instagram_cardnews
- linkedin_post
- medium_article
- pinterest_pin
- x_thread
- facebook_post
- threads
- 이미지 슬롯

원칙:

- 채널 변환은 별도 단계가 아니다.
- 미리보기 화면 안에서 탭으로 제공한다.

### 14.5 이미지 슬롯 매니저

기능:

- drag&drop 업로드
- 슬롯별 프롬프트 표시
- required 슬롯 표시
- generated_image 연결
- prompt_only / pending / ready / approved 상태 표시

---

## 15. 모델별 프롬프트 토글

MVP 포함.

| 모델 | 프롬프트 형식 |
|---|---|
| GPT | Markdown 구조 |
| Claude | XML 태그 구조 |
| Gemini | 예시 기반 구조 |

적용 화면:

- UI6 추출 화면
- 프롬프트 설계 화면

---

## 16. 룰 기반 자동 초안

MVP 포함.

block_type에 따라 기본 스타일 / 이미지 요소 / 채널 프롬프트 초안을 자동 생성한다.

| block_type | 자동 초안 방향 |
|---|---|
| problem | 경고형 카드, 리스크 배지, 고민하는 실무자 아이콘 |
| proof | 숫자 / 근거 카드, 체크 배지, 표 / 차트 요소 |
| solution | 정리된 SaaS 대시보드, 안심 톤 |
| cta | 버튼, 짧은 행동 문구, 명확한 전환 영역 |
| benefit | Before/After 대비, 가치 요약 |
| trust | 검증 / 인증 / 근거 중심 |
| comparison | 좌우 비교표, 선택 기준 강조 |

---

## 17. 재사용 UI

화면 용어:
"다른 프로젝트에서 가져오기"

체크박스:

- 디자인 블록
- 이미지 요소 프롬프트
- 채널 설정
- 전체 프롬프트

내부적으로는 Pack import/export지만, UI에는 Pack 단어를 노출하지 않는다.

---

## 18. 변경 이력

원칙:

- 자체 변경 이력 시스템을 만들지 않는다.
- git에 위임한다.
- 필요 시 .bak 파일 정도만 허용한다.

금지:

- 자체 버전관리 DB
- 자체 히스토리 UI
- 복잡한 변경 추적 시스템

---

## 19. 개발 모듈 구조

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

원칙:

- src/factory/prompts/는 MVP 구현 대상이다.
- src/factory/ai/는 Phase 2 placeholder다.
- MVP에서는 실제 AI API 호출을 구현하지 않는다.
- extract/는 반자동 P1 프롬프트 흐름을 우선한다.
- render/는 Playwright PNG 출력 흐름을 담당한다.

---

## 20. Playwright 렌더링 규칙

| 규칙 | 내용 |
|---|---|
| device_scale_factor | 2 이상 |
| viewport | output_profile width / height와 일치 |
| full_page | false 기본 |
| 폰트 대기 | document.fonts.ready |
| 네트워크 대기 | networkidle 또는 domcontentloaded 후 안정화 |
| 한글 검사 | 한글 폰트 깨짐 확인 |
| PNG 검사 | 파일 크기 / 해상도 / 빈 화면 검증 |
| 파일명 | page_001.png, page_002.png 순차 저장 |
| 대응 | HTML과 PNG 1:1 대응 |

---

## 21. PNG 품질 체크리스트

품질 체크 12개:

1. 생성 성공
2. 해상도 일치
3. 파일 용량 0 아님
4. 빈 화면 아님
5. 한글 텍스트 렌더링 확인
6. safe area 침범 없음
7. 제목 / 본문 / CTA 잘림 없음
8. 페이지 번호 정상
9. 이미지 슬롯 깨짐 없음
10. 폰트 fallback 과다 사용 없음
11. output_profile 규격 준수
12. HTML 파일과 PNG 파일 개수 일치

주의:
V2에서 11/11로 표현한 경우가 있으나, v1.3 합의안의 체크리스트는 12개 항목이다.
최종 문서에서는 12개 체크리스트로 잠근다.

---

## 22. MVP 포함 범위

| 항목 | 포함 여부 |
|---|---|
| Prompt Layer / Prompt Pack 내부 구조 | 포함 |
| P1~P5 프롬프트 파일 | 포함 |
| frontmatter scope | 포함 |
| design_blocks.jsonl | 포함 |
| image_slots + required 필드 | 포함 |
| 모델별 토글 Claude/GPT/Gemini | 포함 |
| 룰 기반 자동 초안 | 포함 |
| 이미지 슬롯 매니저 drag&drop | 포함 |
| 채널 프로파일 구조 | 포함 |
| instagram_portrait 우선 구현 | 포함 |
| 프롬프트 설계 독립 단계 | 포함 |
| HTML/CSS → Playwright PNG 렌더 | 포함 |
| src/factory/prompts/ 모듈 골격 | 포함 |
| PNG 품질 체크리스트 자동화 | 포함 |
| cache.py / slicer.py / main.py 선별 이식 | 포함 |

---

## 23. MVP 제외 / Post-MVP 범위

| 항목 | 처리 |
|---|---|
| 이미지 API 자동 생성 | Post-MVP |
| SNS 자동게시 Threads / IG / X | Post-MVP |
| OAuth | Post-MVP |
| DB / Supabase | Post-MVP |
| 자동 삽입 고도화 | Post-MVP |
| 자체 변경이력 시스템 | 제외 |
| 대량 자동 생성 | Post-MVP |
| 3초 / 6초 자동 업로드 | Post-MVP |
| 벡터 DB | 당분간 제외 |
| 진짜 파인튜닝 | 제외 |
| Telegram 승인 알림 | Post-MVP |
| 캘린더 자동 배치 | Post-MVP |
| src/factory/ai/ 실제 구현 | Phase 2 |
| Framer식 풀 편집 UI | 제외 |

---

## 24. CLI 마이그레이션 결정

| 항목 | 결정 |
|---|---|
| 기존 V1 폴더 | 유지. archive / git tag로 마킹 |
| 새 폴더 | a4-page-flow-factory-v2 |
| 개발 방식 | 기존 V1 패치가 아니라 새 프로젝트 |
| v1.2 코드 | 직접 계승하지 않음 |
| 재사용 코드 | cache.py / slicer.py / main.py 선별 복사 |
| 첫 커밋 | 재사용 코드와 기본 scaffold 흡수 |
| anthropic 의존성 | MVP 제거 |
| block_types.yaml | 폐기 |
| enums.py | 단일 enum 정의로 신규 작성 |

---

## 25. MVP 단일 게이트

MVP의 단일 성공 게이트는 다음이다.

```
factory init
→ factory generate
→ factory copy import
→ factory sample
→ factory render
→ instagram_portrait PNG 1장 생성
→ PNG 품질 체크 통과
```

최소 성공 조건:

| 조건 | 기준 |
|---|---|
| 새 프로젝트 폴더 생성 | a4-page-flow-factory-v2 |
| CLI 실행 | help 출력 |
| init | 기본 폴더 / 시드 생성 |
| generate | P1~P5 프롬프트 파일 생성 |
| copy import | 외부 LLM 결과를 내부 구조로 import |
| sample | instagram_portrait HTML 1장 생성 |
| render | Playwright PNG 1장 생성 |
| quality_check | 12개 체크 통과 |

---

## 26. 1주차 5일 구현 계획

| Day | 작업 | 완료 조건 |
|---|---|---|
| Day 1 | 셋업 + V1 자산 이식 | 새 폴더, pyproject, CLI help, lint 통과 |
| Day 2 | enums / blocks schema + 자료실 시드 | blocks.jsonl / design_blocks.jsonl 검증 |
| Day 3 | P3 + P5 프롬프트 + 골든 픽스처 | P3/P5 산출물 생성 및 fixture 통과 |
| Day 4 | copy import + sample HTML | imported copy로 instagram_portrait HTML 1장 생성 |
| Day 5 | Playwright PNG 1장 + E2E | PNG 1장 생성 + 품질 체크 통과 |

---

## 27. 위험 신호

| 위험 신호 | 의미 | 대응 |
|---|---|---|
| Day 5까지 PNG 1장 미생성 | 인프라 함정 재발 | 범위 축소 |
| anthropic 의존성 추가 | MVP 기본 가정 위반 | 제거하고 prompt-only로 복귀 |
| 4번째 메타 문서 생성 | 문서 폭발 | 3개 문서 잠금으로 복귀 |
| output_profile을 한 번에 10개 구현 | 범위 과대 | instagram_portrait만 남김 |
| 이미지 API를 MVP에 추가 | 비용/복잡도 증가 | Post-MVP로 이동 |
| DB/Supabase 도입 | MVP 범위 초과 | 로컬 파일 기반 유지 |
| 자체 변경 이력 구현 | 불필요한 시스템화 | git 위임 |

---

## 28. 난이도 / 예상 시간 / 선행조건 표

| 작업 | 난이도 | 예상 시간 | 선행조건 |
|---|---|---|---|
| 새 폴더 셋업 | 낮음 | 0.5일 | 없음 |
| V1 자산 선별 이식 | 낮음 | 0.5일 | 새 폴더 |
| enums.py 작성 | 낮음 | 0.5일 | 스키마 결정 |
| blocks.jsonl 검증 | 낮음 | 0.5일 | enums.py |
| design_blocks.jsonl 검증 | 중간 | 0.5일 | enums.py |
| P1~P5 프롬프트 생성 | 중간 | 1일 | prompts 구조 |
| 모델별 토글 | 중간 | 0.5일 | P1/P3/P5 |
| 룰 기반 자동 초안 | 중간 | 1일 | block_type enum |
| copy import | 중간 | 1일 | P1/P2 구조 |
| sample HTML | 중간 | 1일 | imported copy |
| Playwright render | 중간 | 1일 | sample HTML |
| PNG 품질 체크 | 중간 | 0.5일 | render |
| 이미지 슬롯 매니저 | 중간~높음 | 1~2일 | image_slots schema |
| 채널 adapter 확장 | 중간 | 1~2일 | instagram_portrait 성공 |
| 실제 AI API 호출 | 높음 | 3일+ | Phase 2 |

---

## 29. 같은 세션 / 다른 세션 구분표

| 작업 | 세션 |
|---|---|
| 새 폴더 셋업 + pyproject + CLI help | 같은 세션 |
| V1 자산 이식 | 같은 세션 |
| lint / 기본 테스트 | 같은 세션 |
| enums.py + schema validator | 별도 세션 |
| blocks.jsonl / design_blocks.jsonl 시드 | 별도 세션 |
| P1~P5 프롬프트 생성 | 별도 세션 |
| copy import | 별도 세션 |
| sample HTML | 별도 세션 |
| Playwright PNG | 별도 세션 |
| 이미지 슬롯 매니저 | 별도 세션 |
| 채널 adapter 확장 | 별도 세션 |
| AI adapter 실제 구현 | Phase 2 별도 세션 |

---

## 30. 비용 발생 / 비용 없음 구분표

| 작업 | 비용 |
|---|---|
| 로컬 CLI 셋업 | 없음 |
| P1~P5 프롬프트 파일 생성 | 없음 |
| 외부 GPT/Claude/Gemini 웹 복사 사용 | 사용 계정 비용 범위 |
| HTML/CSS 생성 | 없음 |
| Playwright PNG 렌더 | 없음 |
| OCR Tesseract | 거의 없음 |
| 이미지 슬롯 프롬프트 생성 | 없음 |
| 이미지 생성툴 사용 | 사용 툴별 비용 |
| 텍스트 API 자동 호출 | Phase 2 비용 발생 |
| Vision API 자동 호출 | Phase 2 비용 발생 |
| 이미지 API 자동 생성 | Post-MVP 비용 큼 |
| SNS 자동게시 | API/운영비 발생 가능 |
| DB/Supabase | Post-MVP 비용 가능 |

---

## 31. 우선순위 Top 10

| 순위 | 작업 | 이유 |
|---|---|---|
| 1 | 새 폴더 셋업 | 기존 V1 패치 방지 |
| 2 | V1 자산 선별 이식 | 검증된 유틸만 가져오기 |
| 3 | CLI help / init | 최소 실행 가능 |
| 4 | enums.py | 단일 정의 원본 |
| 5 | blocks / design_blocks schema | 텍스트와 디자인 블록 분리 |
| 6 | P1~P5 prompt files | v1.3 핵심 |
| 7 | model toggle | Claude/GPT/Gemini 복사 흐름 |
| 8 | copy import | 반자동 워크플로우 연결 |
| 9 | sample HTML | 산출물 가시화 |
| 10 | Playwright PNG + 품질 체크 | MVP 성공 게이트 |

---

## 32. V3 이후 보류 항목

아래는 MVP 운영 후 결정한다.

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

## 33. 최종 하지 말아야 할 것

1. 기존 V1 폴더를 계속 패치하지 않는다.
2. v1.2의 M2-5 흐름으로 돌아가지 않는다.
3. API 자동 호출을 MVP에 넣지 않는다.
4. anthropic 의존성을 MVP에 넣지 않는다.
5. DB / Supabase를 MVP에 넣지 않는다.
6. 자동게시를 MVP에 넣지 않는다.
7. 이미지 생성 AI로 글자 박힌 카드뉴스를 만들지 않는다.
8. output_profile 10개를 한 번에 구현하지 않는다.
9. 문서를 4개 이상으로 늘리지 않는다.
10. 자체 변경 이력 시스템을 만들지 않는다.
11. Day 5 PNG 1장 게이트를 삭제하지 않는다.
12. P1~P5를 흐릿하게 만들지 않는다.
13. Pack이라는 단어를 사용자 UI에 노출하지 않는다.

---

*본 문서는 v1.3 합의안 §1~§33 사용자 작성 원문이다. 편집·합성·요약 금지.*
*통합 잠금본은 `A4_PAGE_FLOW_FACTORY_V1_3_REDESIGN.md` 참조.*
*충돌 시 본 원문이 통합본보다 우선한다.*
