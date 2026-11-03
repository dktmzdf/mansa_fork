<img src="docs/images/logo.svg" width="72" alt="MANSA 로고" />

# MANSA — 박규진(dktmzdf) 기여 정리

**Manual & Safety Assistant — 매뉴얼을 물으면 안전이 따라옵니다**

현대 AI 인사이트 캠퍼스 6팀 프로젝트 [HYUNDAI-AI-team6/mansa](https://github.com/HYUNDAI-AI-team6/mansa)를 포크한 저장소입니다.
이 문서는 팀 프로젝트에서 **제가 맡아 구현한 부분**을 정리합니다.
프로젝트 전체 소개·실행 방법·폴더 구조는 [README_Fork.md](README_Fork.md)(원본 README)를 참고해 주세요.

## 프로젝트 한 줄 소개

현장 엔지니어가 설비 매뉴얼과 안전 기준을 따로 찾지 않고, 질문 하나로 작업 절차·사양 수치·안전 요건을 출처와 함께 확인하도록 돕는 RAG 기반 질의응답 서비스입니다.
(Python 3.11 · Gradio · LangChain · ChromaDB · MySQL)

## 담당 파트

| 영역 | 내용 |
| --- | --- |
| 관리자 평가 | 테스트셋 → 응답 수집 → 채점 → 결과 저장·화면 표시까지 평가 기능 전체(백엔드·화면) |
| 고객센터 챗봇 | LLM tool calling 기반 MANSA 사용법 안내 챗봇, 안내 데이터 외부 파일화 |
| 사용자 문서 조회 | 목데이터 화면을 DB 문서 목록과 연결 |
| 태그 필터 검색 | 벡터 DB에서 태그로 필터링한 뒤 유사도 검색하는 API |
| 협업 규칙 | Python 개발 가이드·AGENTS.md(AI 코딩 도구 공통 규칙), gitignore, 서버 실행 배치 파일 |

## 기여 요약

| 항목 | 수치 |
| --- | --- |
| 커밋 (머지 제외) | 34개 · 130개 파일 · +7,298 / −1,036줄 |
| 작성한 이슈 | 8개 (모두 완료) |
| 작성한 PR | 12개 (모두 머지) |
| 코드 리뷰한 PR | 46개 |

기간: 2026-10-20 ~ 2026-10-30

---

## 1. 관리자 평가 기능

> 이슈 [#31](https://github.com/HYUNDAI-AI-team6/mansa/issues/31), [#78](https://github.com/HYUNDAI-AI-team6/mansa/issues/78), [#124](https://github.com/HYUNDAI-AI-team6/mansa/issues/124)
> PR [#46](https://github.com/HYUNDAI-AI-team6/mansa/pull/46), [#84](https://github.com/HYUNDAI-AI-team6/mansa/pull/84), [#159](https://github.com/HYUNDAI-AI-team6/mansa/pull/159)
> 코드: `mansa/features/evaluation/`, `mansa/ui/evaluation.py` · 문서: [docs/Evaluation.md](docs/Evaluation.md)

테스트셋 문항을 Q&A 파이프라인에 보내 답변을 받고, 채점해서 지표·실패 사례·지연·토큰을 화면과 파일로 보여 주는 관리자 기능입니다.

```
eval/testsets/tech_qa.json ─ 테스트셋 읽기·검증 (testset.py)
        ▼
문항별 답변 수집 (answerer.py → QAService)        진행률: 답변 n/N
        ▼
질문-답변 세트 저장 (storage.py)
        ▼
규칙 채점 + LLM-as-judge (scoring.py, judge.py)   진행률: 채점 n/N
        ▼
결과 JSON·CSV 저장 → 평가 화면 표시
```

**지표 5개**: 검색 적중률, 수치 정확도, 출처 정확도, 과잉 첨부율, 표현 일관성 + 평균 지연·평균 토큰

### 주요 구현

- **수치 정확도 채점 — 정규식 규칙 + LLM 보조**
  - 부호·숫자·단위(대소문자 포함)가 모두 같아야 정답 처리 (`MPa` ≠ `mPa`, `-1000 kg` ≠ `1000 kg`)
  - 천 단위 쉼표, 범위(`5-10 mm`), 분수, 복합 단위(`N·m`), 옴(Ω) 단위 처리
  - 규칙이 놓친 수치만 채점 LLM 판정을 쓰되, LLM이 제시한 근거 문장이 답변에 실제로 있고 정답 숫자를 포함할 때만 인정해 환각·단위 환산을 차단
  - 규칙과 LLM 판정이 엇갈린 경우 `LLM 통과` / `LLM 이견`으로 따로 기록
- **단계적 교체가 가능한 구조**
  - 응답기·채점기를 `Answerer` / `Judge` Protocol로 분리
  - Q&A 파이프라인이 완성되기 전에는 Codex CLI로 프로토타입을 만들고([#46](https://github.com/HYUNDAI-AI-team6/mansa/pull/46)), 팀 LLM 모듈이 나오자 교체([#84](https://github.com/HYUNDAI-AI-team6/mansa/pull/84)), 최종적으로 실제 `QAService`를 호출하는 `QaAnswerer`로 교체([#159](https://github.com/HYUNDAI-AI-team6/mansa/pull/159))
- **실제 RAG 지표 측정** ([#159](https://github.com/HYUNDAI-AI-team6/mansa/pull/159))
  - `RecordingRetriever`로 매뉴얼·안전 문서 검색 결과를 기록해 검색 적중률·출처 정확도·과잉 첨부율을 실제로 계산
  - `get_usage_metadata_callback()`으로 질문 분석·답변 생성·재생성 호출의 토큰을 합산
  - 정답 근거 비교 단위를 청크 ID에서 `파일명#p페이지`로 바꿔, 문서를 다시 올려도 테스트셋이 깨지지 않도록 함
  - 검색 적중률 기준을 "하나라도 검색됨"에서 "문항별 재현율 60% 이상"으로 강화
  - 평가 질문 50문항과 정답표 작성
- **오류 처리**
  - LLM 호출 실패(시간 초과, JSON 해석 실패, 검열 응답)가 나도 평가가 멈추지 않고 나머지 문항을 계속 진행, 실패는 "LLM 오류" 목록에 별도 기록
  - 평가는 앱 전체에서 동시에 하나만 실행
- **결과 저장**: 실행마다 `{run_id}_answers.json`, `_result.json`, `_result.csv`(엑셀용 UTF-8 BOM) 생성, 저장된 답변으로 재채점 가능
- **테스트**: 채점·응답기·테스트셋·저장·러너 단위 테스트 작성

## 2. 고객센터 챗봇

> 이슈 [#91](https://github.com/HYUNDAI-AI-team6/mansa/issues/91), [#146](https://github.com/HYUNDAI-AI-team6/mansa/issues/146)
> PR [#100](https://github.com/HYUNDAI-AI-team6/mansa/pull/100), [#162](https://github.com/HYUNDAI-AI-team6/mansa/pull/162), [#164](https://github.com/HYUNDAI-AI-team6/mansa/pull/164)
> 코드: `mansa/features/support/`, `mansa/data/support/guides.json`, `mansa/ui/support.py`

<img src="docs/images/support.png" width="640" alt="고객센터 화면" />

MANSA 사용법을 묻는 질문에 LLM이 알맞은 안내 주제를 골라 답하는 챗봇입니다.

- **Tool calling 기반 응답**
  - `core/llm.py`에 도구를 붙여 LLM을 호출하는 `choose_tool()` 추가 (시간 초과 재시도, `LlmError` 변환)
  - LLM이 `GetGuide(topic)` 도구로 안내 주제를 고르면 미리 작성한 안내 문구로 답변 → 사용법 안내에서 환각을 줄임
  - 기술 질문은 기술 Q&A로, 안내에 없는 질문은 관리자 연결로 유도
  - LLM 설정 없음·호출 실패·알 수 없는 주제일 때는 기본 안내 문구로 대체
- **안내 데이터를 코드 밖 JSON으로 분리** ([#162](https://github.com/HYUNDAI-AI-team6/mansa/pull/162))
  - 이전에는 주제 하나를 추가하려면 dict·`Literal`·`Field(description)` 세 곳을 함께 고쳐야 했음
  - 이제 `guides.json`만 수정하면 앱 시작 시 `pydantic.create_model`로 도구 스키마를 자동 생성
  - id 중복, 빈 설명, `steps`/`text` 동시 지정 등 잘못된 데이터는 앱 시작 단계에서 `ValueError`로 막음
  - 안내 주제를 5개 → 19개로 확장 (로그인, 되묻기, 답변 불가, 근거 확인, 문서 조회 등)
- **코드 리뷰 반영**: 툴 호출 내용이 로그에 그대로 찍히던 문제, 예상하지 못한 인자에서 크래시 나던 문제 수정

## 3. 사용자 문서 조회

> 이슈 [#127](https://github.com/HYUNDAI-AI-team6/mansa/issues/127) · PR [#130](https://github.com/HYUNDAI-AI-team6/mansa/pull/130) · 코드: `mansa/ui/doc_view.py`

- 목데이터 대신 DB `documents_info`를 읽도록 변경 (관리자 문서 관리의 `load_docs()` 재사용)
- 처리 상태가 `완료`인 문서만 사용자에게 노출
- 부서 선택 목록을 완료 문서의 부서로 구성, 메뉴 진입·필터 변경 시마다 최신 상태 반영
- 순환 import를 피하도록 구조 조정, 단위 테스트 작성 (DB 없이 가짜 함수로 대체)

## 4. 태그 필터 유사도 검색 API

> 이슈 [#104](https://github.com/HYUNDAI-AI-team6/mansa/issues/104) · PR [#123](https://github.com/HYUNDAI-AI-team6/mansa/pull/123) · 코드: `mansa/util/tag_search.py`

- ChromaDB에서 청크 태그로 먼저 필터링한 뒤 벡터 유사도 검색을 하는 API 추가

## 5. 협업 환경

> PR [#3](https://github.com/HYUNDAI-AI-team6/mansa/pull/3), [#12](https://github.com/HYUNDAI-AI-team6/mansa/pull/12), [#18](https://github.com/HYUNDAI-AI-team6/mansa/pull/18), [#51](https://github.com/HYUNDAI-AI-team6/mansa/pull/51)

- [Python 개발 가이드](docs/dev-stack-python.md)와 [AGENTS.md](AGENTS.md)(AI 코딩 도구 공통 작업 규칙) 작성 — 팀원 모두가 AI 도구를 쓰는 환경에서 비밀값 관리, 위험 명령 금지, 최소 수정, Git 방식 등을 통일
- 깨진 문서 링크 수정, `.gitignore`에 Claude·Codex 스킬 폴더와 평가 결과 폴더 추가
- Windows용 서버 실행 배치 파일(`Execute_Server.bat`) 추가

## 6. 코드 리뷰

팀원 PR **46개**를 리뷰했습니다. 검색(BM25·RRF·태그 순위), 답변 생성·되묻기, 문서 추출·청킹·임베딩, DB 연결·세션, 화면 연결 등 프로젝트 대부분의 기능 PR이 포함됩니다.

---

## 작성 이슈·PR 목록

| 구분 | 번호 | 제목 |
| --- | --- | --- |
| 이슈 | [#11](https://github.com/HYUNDAI-AI-team6/mansa/issues/11) | 깨진 문서 url 수정 및 gitIgnore 추가 |
| 이슈 | [#17](https://github.com/HYUNDAI-AI-team6/mansa/issues/17) | gitignore에 클로드 및 코덱스 스킬 폴더 추가 |
| 이슈 | [#31](https://github.com/HYUNDAI-AI-team6/mansa/issues/31) | 관리자 평가 페이지 구현 |
| 이슈 | [#78](https://github.com/HYUNDAI-AI-team6/mansa/issues/78) | 코덱스 호출을 구현된 LLM으로 교체 |
| 이슈 | [#91](https://github.com/HYUNDAI-AI-team6/mansa/issues/91) | 고객센터 LLM 구현 |
| 이슈 | [#124](https://github.com/HYUNDAI-AI-team6/mansa/issues/124) | 평가쪽 제대로 동작하지 않는 문제 수정 |
| 이슈 | [#127](https://github.com/HYUNDAI-AI-team6/mansa/issues/127) | 사용자의 문서 조회 페이지 작성 |
| 이슈 | [#146](https://github.com/HYUNDAI-AI-team6/mansa/issues/146) | 고객센터 FAQ 데이터 추가 |
| PR | [#3](https://github.com/HYUNDAI-AI-team6/mansa/pull/3) | 파이썬 컨벤션 및 AGENTS.md 올림 |
| PR | [#12](https://github.com/HYUNDAI-AI-team6/mansa/pull/12) | 깨진 문서 url 수정 및 gitignore 수정 |
| PR | [#18](https://github.com/HYUNDAI-AI-team6/mansa/pull/18) | gitIgnore 수정 |
| PR | [#46](https://github.com/HYUNDAI-AI-team6/mansa/pull/46) | 관리자 평가 페이지 구현 |
| PR | [#51](https://github.com/HYUNDAI-AI-team6/mansa/pull/51) | 서버 실행시키는 배치파일 추가 |
| PR | [#84](https://github.com/HYUNDAI-AI-team6/mansa/pull/84) | 코덱스 호출을 구현된 LLM으로 교체 |
| PR | [#100](https://github.com/HYUNDAI-AI-team6/mansa/pull/100) | 고객센터 LLM 구현 |
| PR | [#123](https://github.com/HYUNDAI-AI-team6/mansa/pull/123) | 태그 유사도 검색할수 있는 api 추가 |
| PR | [#130](https://github.com/HYUNDAI-AI-team6/mansa/pull/130) | 문서 조회 구현 |
| PR | [#159](https://github.com/HYUNDAI-AI-team6/mansa/pull/159) | 평가쪽 제대로 동작하지 않는 문제 수정 |
| PR | [#162](https://github.com/HYUNDAI-AI-team6/mansa/pull/162) | mansa 사용법 데이터 추가로 고객센터 대응 개선 |
| PR | [#164](https://github.com/HYUNDAI-AI-team6/mansa/pull/164) | `__init__.py` 옮김 |

## 사용 기술

Python 3.11 · uv · Gradio · LangChain (`langchain-openai`, tool calling) · Pydantic · ChromaDB · MySQL · unittest
