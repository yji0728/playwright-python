# 웹 데이터 수집 웹 인터페이스 스펙 (Playwright Python 기반)

## 목적과 범위

- 목적: 사용자가 원하는 웹 자료(문서/목록/가격/이벤트 등)를 지정된 규칙으로 수집·가공하여 웹 UI에서 탐색/다운로드/자동화 관리가 가능하도록 하는 서비스.
- 범위: 합법적이고 허용된 웹 리소스에 한해, 비로그인/로그인 페이지, 무한 스크롤/SPA 포함 동적 콘텐츠 처리. 결과는 구조화(JSON/CSV), 미디어(이미지/스크린샷), 추적(Trace/HAR) 형태로 제공.
- 기술 기반: 이 리포지토리의 Python Playwright 라이브러리와 트레이싱/라우팅/컨텍스트 API를 활용. 웹 서비스는 Python(FastAPI 권장) + 작업 실행 워커(Playwright) + 큐/스토리지로 구성.

성공 기준
- 사용자가 UI에서 작업(Job)을 생성하고 상태/결과를 확인할 수 있다.
- 주요 사이트의 동적 로딩과 페이징/스크롤, 로그인 흐름을 구성 가능하다.
- 실패/재시도/타임아웃/안티봇 대응 정책을 설정 가능하다.
- 결과는 재현 가능하도록 Trace/HAR/스크린샷을 선택 저장할 수 있다.

---

## 사용자 페르소나와 주요 시나리오

- 데이터 분석가: 키워드/도메인별 크롤링 규칙을 저장하고 주기적으로 실행, 결과를 CSV/Parquet로 내보냄.
- 리서처/PM: 일회성 수집을 UI에서 빠르게 설정, 결과를 검토하며 규칙을 미세조정.
- 운영자(Admin): 작업 풀/리소스/접속 권한/도메인 허용 리스트 관리.

주요 시나리오
1) 작업 생성: URL/쿼리/선택자/추출 스키마 지정 → 미리보기 → 실행 → 결과 다운로드
2) 목록 크롤: 페이징/무한 스크롤 규칙과 항목 선택자 지정 → 항목 상세까지 파이프라인 추출
3) 로그인 필요: 자격 증명 금고 참조 + 저장소 상태(storageState) 재사용 → 승인 세션으로 수집
4) 실패 관리: 네트워크/셀렉터 실패 시 재시도·백오프 → 사용자에게 의미 있는 오류 메시지 제공
5) 자동화/예약: cron/주기 실행, 변경 감지(diff) 시 알림(Webhook/Email)

---

## 기능 요구사항

필수
- 작업 생성/조회/취소/재시작/예약
- 소스 구성: URL 리스트, 사이트 템플릿, 파라미터화된 쿼리
- 추출 스키마: CSS/Role/Locator 기반과 post-process(JS 표현식/정규식)
- 동적 로딩 지원: 대기 전략(waitForSelector/networkidle/요소 수 변동), 스크롤/클릭 페이징
- 로그인 흐름: 폼 로그인, 2단계(사용자 개입 또는 범용 핸드오프), storageState/쿠키 재사용
- 결과 저장/다운로드(JSON/CSV), 스크린샷/Trace/HAR 옵션
- 작업 상태와 로그 스트림(웹소켓) 제공
- 도메인 허용 리스트, Rate limiting, 요청 헤더/프록시/UA 설정

선택
- 레시피 마켓(공유 가능한 템플릿)
- 알림: Webhook/Email/Teams
- 중복 필터링/변경 감지
- 번역/정규화 파이프라인
- 외부 CAPTCHA 솔버 연동(선택, 준법 내)

---

## 비기능 요구사항

- 성능: 단일 워커 기준 평균 TTFJ(Time to first JSON) < 10s(사이트/정책에 따라 상이), 동시 N(가변) 스케줄링
- 확장성: 수평 확장(워커/큐), 브라우저 컨텍스트 재사용으로 오버헤드 최소화
- 신뢰성: 재시도 지수 백오프, 작업 타임아웃, 멱등적 재시작
- 보안/준법: RBAC, 도메인 허용, 비밀 관리, 로깅에서 PII 마스킹, 사이트 약관/robots 준수 설정
- 접근성: UI WCAG 2.1 AA 준수 지향
- 관찰성: 구조적 로그, 트레이스/메트릭, 분산 트레이싱(작업 ID 기준)

---

## 아키텍처 개요

- 프론트엔드: SPA(React/Next.js 권장) + WebSocket 상태 스트림
- 백엔드 API: FastAPI
  - REST 엔드포인트 제공, 작업 생성/조회/결과 다운로드/알림 등록
- 워커(크롤러): Playwright Python 실행 유닛
  - 브라우저 풀 관리(Chromium/Firefox/WebKit), 컨텍스트 재사용, 트레이싱/스크린샷
- 큐: Redis/RabbitMQ/SQS 중 택1
- 스토리지: Postgres(메타/스케줄/결과 인덱스) + S3 호환 객체 저장소(Trace/HAR/이미지/원본)
- 비밀 금고: Azure Key Vault/AWS Secrets Manager/HashiCorp Vault 중 택1
- 관찰성: OpenTelemetry + 로그/메트릭 수집(예: Prometheus/Grafana)

데이터 흐름
1) 사용자 → API: 작업 생성
2) API → 큐: Job 메시지
3) 워커: 메시지 수신 → Playwright 실행 → 후처리 → 결과 저장
4) API/UI: 상태/결과 조회, 실시간 업데이트

---

## 데이터 모델(요약)

### Job
- id, name, status(queued/running/succeeded/failed/canceled/expired)
- source: type(urls|template|query), config(json)
- extract_schema: fields[{name, selector|locator, attr|text, transform?}]
- options: browser, headless, locale/timezone/geolocation, proxy, retries, timeout, trace, screenshot
- schedule: cron?, startAt?, expireAt?
- owner, createdAt, updatedAt, lastRunAt

### Run
- id, jobId, attempt, status, startedAt, endedAt, error?
- metrics: navTime, selectorWaits, retries, bytes, requests
- artifacts: traceUrl, harUrl, screenshots[...], logsUrl

### Result
- id, jobId, runId, itemIndex, payload(json), rawHtml?, mediaUrls[...], dedupeKey?

### SourceTemplate
- id, name, domain, steps(scroll/paginate/click), loginFlow?, robotsPolicy

### CredentialRef
- id, type(form/basic/bearer), secretRef, storageStateUrl?

---

## API 스펙(REST)

공통
- 요청/응답 Content-Type: application/json
- 인증: Bearer JWT
- 에러 형식: { error: { code, message, details? } }

### POST /jobs
- body: { name, source, extract_schema, options, schedule? }
- 201: { id, status, createdAt }
- 400: 검증 오류, 403: 권한, 409: 제한 위반(도메인 블록 등)

### GET /jobs?status=&owner=&q=&limit=&cursor=
- 200: { items:[{...}], nextCursor? }

### GET /jobs/{id}
- 200: Job + 최근 실행/통계

### POST /jobs/{id}/run
- 즉시 실행 트리거, 202: { runId, status: "queued" }

### GET /jobs/{id}/runs
- 실행 히스토리 목록

### GET /runs/{runId}
- 실행 상세, 메트릭/아티팩트 링크 포함

### GET /runs/{runId}/results?limit=&cursor=&format=json|csv
- 200: JSON 스트리밍 또는 CSV 다운로드 링크

### WS /runs/{runId}/events
- status/log/progress 이벤트 스트림

### POST /credentials
- 자격 증명 참조 생성(비밀 금고 연동 키만 저장)

### GET /templates
- 템플릿 목록/다운로드

---

## 추출 스키마(예)

필드 단위 정의
- selector: CSS/locator 규칙. 예: "article .title"
- mode: text|attr|html|eval
- attr: mode=attr일 때 대상 속성
- eval: 선택된 요소에 대한 JS 표현식(안전 sandbox)
- multiple: 다건 추출 여부
- required: 미검출시 실패 처리 여부
- post: 정규식/치환/트림 등

예시
```json
{
  "fields": [
    { "name": "title", "selector": "h1", "mode": "text", "required": true },
    { "name": "price", "selector": ".price", "mode": "text", "post": {"regex": "\\d+[\\,\\.]?\\d*"} },
    { "name": "images", "selector": "img", "mode": "attr", "attr": "src", "multiple": true }
  ]
}
```

---

## Playwright 활용 전략

- 브라우저 풀: per-worker에서 브라우저 타입별 1개 인스턴스 유지, 컨텍스트 재사용(스토리지 분리)
- 대기/안정화: route.waitFor, locator.waitFor, networkidle, requestfinished 메트릭 기반 다음 스텝 진행
- 무한 스크롤: evaluate로 viewport 높이·요소 수 추적, 스크롤 반복+중복 방지, 상한선/타임박스
- 페이징: next-selector(버튼) 클릭+가시성 대기, page.url 변화/항목 수 증가 확인
- 로그인: storageState 파일 관리, 만료 시 재로그인 플로우 수행
- 네트워크 제어: 헤더/UA/Accept-Language, geolocation/locale/timezone 설정, 프록시/HTTP 인증
- 안티봇 배려: 과도한 동시성/짧은 텀 클릭 방지, 지터 지연, 마우스/키보드 이벤트 인젝션 간소화
- 관찰성: tracing.start/stop(zip 보관), context.tracing, page.screenshot 단계별 캡처, HAR 기록
- 안전: robots 정책 확인 옵션, 도메인 allowlist, ToS 준수 가드

---

## 크롤링 워크플로

1) 생성: Job 저장 → 큐에 메시지
2) 준비: 워커가 브라우저 컨텍스트 생성, 프록시/UA/스토리지 적용
3) 탐색: 소스(템플릿/URL/쿼리) 단계 실행(로그인→목록→상세)
4) 추출: 스키마에 따라 필드 추출, 후처리/정규화
5) 저장: 결과 배치 업로드, 아티팩트 저장(Trace/HAR/이미지)
6) 완료: 상태 업데이트, 알림(Webhook/Email)
7) 재시도: 일시 오류(네트워크/타임아웃/가시성 실패) → 지수 백오프, 최대 횟수 N

경계 사례
- 요소 미검출: required=true → 필드 에러, 정책에 따라 레코드 스킵/실패
- 로그인 만료: 재로그인 1회 시도 후 실패
- 무한 스크롤 한계: 아이템 증가 없음 → 조기 종료
- 중복: dedupeKey 기반 저장소 중복 제거

---

## UI 화면 및 컴포넌트

- 대시보드: 최근 Job/Run 상태 카드, 성공률/에러 분포
- 작업 생성 마법사
  - 소스 단계: URL 리스트/템플릿/쿼리
  - 로그인/세션: 자격 증명 참조 선택, 세션 테스트
  - 추출 스키마: 시각 선택(요소 픽커)/미리보기
  - 실행 옵션: 브라우저·headless·locale·프록시·타임아웃·트레이스/스크린샷
  - 검증/미리보기: 샘플 결과/스크린샷 표시
- 실행 모니터: 로그 스트림, 스텝 타임라인, 스크린샷 썸네일, Trace/HAR 다운로드
- 결과 목록: 필터/검색/컬럼 토글, CSV/JSON 내보내기, 변경 감지 표시
- 스케줄/알림: cron/주기, Webhook/이메일 구성
- 설정: 도메인 허용 리스트, Rate Limit, RBAC 역할/권한

---

## 에러·예외 체계

카테고리
- NAVIGATION_ERROR, SELECTOR_TIMEOUT, AUTH_FAILED, BLOCKED_BY_ROBOTS, RATE_LIMITED, NETWORK_ERROR, SCRIPT_EVAL_ERROR, STORAGESTATE_INVALID, UNKNOWN

HTTP 매핑
- 400 검증, 401/403 인증·권한, 409 정책 충돌, 422 처리 불가, 429 과다 요청, 500 내부

재시도 정책
- 네트워크/타임아웃/429 → 백오프 재시도
- 인증/정책 위반 → 즉시 실패

사용자 메시지
- 기술 메시지를 인간 친화적으로 매핑, 해결 팁 포함

---

## 보안·권한

- 인증: OAuth2/OIDC→JWT
- 권한: RBAC(Owner, Editor, Viewer, Admin)
- 입력 검증: URL/셀렉터/스크립트 길이·패턴 제한, 샌드박스된 eval만 허용
- 비밀: 자격 정보는 금고 참조만 저장, 평문 금지
- 네트워크: 아웃바운드 도메인 허용 리스트, 조직 프록시/아이피 명시
- 남용 방지: 사용자·도메인별 Rate limiting/쿼터
- 로그: PII 마스킹, 결과 데이터 별도 보관 정책

---

## 로그/모니터링

- 로그: JSON 구조 {ts, level, jobId, runId, step, msg, err?}
- 메트릭: 실행 시간, 실패율, 페이지 로드/요청 수, 바이트, 재시도 횟수
- 추적: OpenTelemetry traceId=runId, 스팬=스텝 단위
- 알림: 임계치 초과/연속 실패/큐 적체 경보

---

## 테스트 전략

- 단위: 추출기(post-process, 정규식), 설정 검증
- 통합: 템플릿별 최소 happy path + 1~2 경계(요소 미검출, 타임아웃)
- E2E: Playwright UI 테스트(작업 생성→실행→결과 확인), 워커 모의/실체 혼합
- 성능: 동시 실행 N에서 평균/95% 타임 측정
- 보안: 입력 페이로드 fuzz, 인증/권한 우회 방지 테스트

---

## 배포/운영

- 컨테이너: API/워커 분리, Playwright 브라우저 설치 포함 이미지
- 구성: DEV/STAGE/PROD, 브라우저 채널/버전 고정, 폰트/지역화 패키지 포함
- CI/CD: 테스트→보안 스캔→릴리즈→점진 배포, 롤백 버튼
- 스토리지: S3(아티팩트)/DB(메타) 백업/보존 정책
- 비용: Trace/HAR는 선택 저장, TTL/압축으로 비용 관리

---

## 확장/로드맵(요약)

- 플러그인: 사이트별 로그인/추출 어댑터 인터페이스
- 커넥터: BigQuery/Snowflake/Elastic 내보내기
- GraphQL API 추가, 실시간 구독
- 다국어 UI, 멀티 테넌시/조직
- ML 추천: 셀렉터 제안/항목 자동 식별(중장기)

---

## MVP 범위와 다음 단계

MVP
- 작업 생성/실행/결과 확인(단일 워커)
- URL/템플릿 소스, 스키마 기반 단순 추출
- 무한 스크롤/페이징 기본, 로그인(storageState) 지원
- JSON/CSV 다운로드, 스크린샷·Trace 옵션
- RBAC 기본, 도메인 허용·Rate limit
- 관찰성: 구조 로그 + 간단 메트릭

다음 단계
- 예약/알림, 변경 감지
- 템플릿 저장소/공유
- 성능 최적화(브라우저 풀, 컨텍스트 재사용 정책 고도화)
- 고급 보안/준법(감사용 감사 로그, DPIA 지원)

---

## 간단한 계약 요약

입력
- 소스: URL/템플릿/쿼리
- 옵션: 브라우저·시간·위치·프록시·타임아웃·재시도
- 추출 스키마: 필드별 선택자/모드/후처리

출력
- 결과: JSON/CSV
- 아티팩트: 스크린샷/Trace/HAR
- 메타: 실행 메트릭/로그

오류 모드
- 내비게이션/셀렉터/인증/정책/네트워크/429
- 정책에 따른 재시도/실패/스킵

성공 기준
- 지정된 스키마에 따라 안정적으로 결과를 생성하고, 실패 시 원인을 파악 가능하며 재현(Trace/HAR) 가능
