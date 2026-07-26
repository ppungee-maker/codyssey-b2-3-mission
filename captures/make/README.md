# Make 시나리오 — 구축·실행 기록 (완료)

이 디렉터리는 **작업 지시서가 아니라 결과 기록**이다. 시나리오는 실제로 만들어져 돌았다.

| 파일 | 내용 |
|---|---|
| [`scenario-blueprint.json`](scenario-blueprint.json) | 시나리오 구조 전문 — 모듈 8개, 연결, 필드 매핑. **API 키·토큰은 `<REDACTED>` 로 마스킹** |
| [`run-history.json`](run-history.json) | 실행 이력 — 시각·성공여부(`status` 1=성공/3=오류)·오퍼레이션 수·소요시간 |

성공 실행 = **8 오퍼레이션 / 약 4.6초**. 한 번 돌면 Notion 에 행 3개가 생긴다.
자동화가 실제로 만든 텍스트와 규격 대조는 [`../../outputs/05-automation-run-2026-07-26.md`](../../outputs/05-automation-run-2026-07-26.md).

---

## 구축 방식 — 전 과정 API

브라우저에서 모듈을 끌어다 놓지 않았다. Make API 로 청사진(blueprint)을 업로드해 만들었다.

```
POST /api/v2/hooks       웹훅 생성 → 트리거 주소 확보
POST /api/v2/scenarios   blueprint(JSON) 업로드
PATCH /api/v2/scenarios/{id}   스케줄 = immediately (웹훅 도착 즉시 실행)
POST /api/v2/scenarios/{id}/start   활성화
```

그래서 이 워크플로우는 **파일로 재현된다** — `scenario-blueprint.json` 을 Make 의
`Import Blueprint` 로 올리면 같은 구조가 그대로 생긴다. 재현 절차는 루트 README §7-3.

## 구조

| # | 모듈 | 역할 |
|---|---|---|
| 1 | `Webhooks > Custom webhook` | `{"topic": "..."}` 수신 |
| 2 | `Tools > Set variable` (`대표이미지`) | 대표 이미지 URL 을 **Router 밖에서 1회만** 정의 → 3갈래 공유(비용 3배 방지) |
| 3 | `Router` | 인스타그램 / 블로그 / X 3분기 |
| 4·6·8 | `HTTP` → Gemini API | 갈래별 최종 프롬프트(루트 README §5) + `{{1.topic}}` |
| 5·7·9 | `HTTP` → Notion REST API | 속성 매핑 후 행 1개 생성 |

Notion 을 앱 모듈이 아니라 `HTTP` 로 붙인 이유 = Notion 앱 모듈은 OAuth 브라우저 로그인이
필요한데, 이 구성은 전부 API 로 돌리기로 했기 때문. 저장 결과는 동일하다.

---

## 실제로 걸린 함정 4개 (전부 실측)

같은 걸 만들 사람이 시간을 안 버리도록 남긴다.

**1. Make API 가 403 `error code: 1010` 을 낸다 — 무료 플랜 제한이 아니다.**
Cloudflare 가 기본 User-Agent(스크립트 라이브러리의 것)를 브라우저 시그니처로 차단한다.
평범한 브라우저 User-Agent 헤더를 붙이면 무료 플랜에서도 API 가 정상 동작한다.

**2. AI 응답의 줄바꿈이 JSON 을 깨뜨린다.**
Notion 이 `invalid_json` 400 을 낸다. JSON 문자열 안에는 실제 줄바꿈 문자를 넣을 수 없기 때문.
→ `replace(<본문>; newline; "\n")` 으로 이스케이프.

**3. Make 의 `replace()` 는 정규식 리터럴(`/\n/g`)을 조용히 무시한다.**
오류도 안 나고 치환도 안 된다 — 원문이 그대로 꽂혀 2번 증상으로 되돌아간다.
내장 키워드 `newline` / `space` / `emptystring` 을 써야 한다. 변형 6종을 돌려 확인했다.

**4. Gemini 무료 티어 한도는 모델마다 다르고, 최신 별칭일수록 좁다.**

| 모델 | 무료 일일 한도 |
|---|---|
| `gemini-flash-latest` (= `gemini-3.6-flash`) | **20건** |
| `gemini-2.5-flash` | 20건 |
| **`gemini-flash-lite-latest`** (= `gemini-3.5-flash-lite`) | **500건** |

디버깅으로 20건은 순식간에 마른다. 쿼터 버킷은 **모델별로 분리**돼 있어 모델을 바꾸면 바로 풀린다.
그래서 운영 모델은 Flash Lite 계열로 잡았다.

---

## 선택 — 화면 캡처

증빙은 위 JSON 두 개로 이미 성립한다(구조·실행시각·성공여부·소요시간이 전부 들어 있다).
시각 자료가 추가로 필요하면 Make 에서 시나리오를 열어 `01-scenario.png`(전체 구조),
`02-run-history.png`(실행 히스토리)로 저장해 이 디렉터리에 넣는다.

> ⚠️ **캡처에 API 키·Notion 토큰·웹훅 주소가 보이면 반드시 가린다.** 이 저장소는 public 이다.
> 웹훅 주소는 인증 없는 실행 트리거라, 노출되면 남이 시나리오를 돌려 무료 오퍼레이션을 태울 수 있다.
