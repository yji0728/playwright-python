# 구현 로드맵 (Playwright Python 기반 웹 데이터 수집 서비스)

본 로드맵은 MVP 릴리스를 목표로 한 단계적 실행 계획과 마일스톤, 산출물, 수용 기준을 포함합니다. 기간은 팀 규모/인프라에 따라 조정할 수 있습니다(주 단위 예시).

## 원칙
- 작은 단위로 가시적 가치를 빠르게 제공(주 단위 인크리먼트)
- 재현성과 관찰성을 우선(Trace/HAR/로그/메트릭)
- 보안과 준법 내에서 성장(Allowlist/Rate limit/비밀 금고)
- 테스트 자동화와 CI/CD 내재화

## 아키텍처 스택(권장)
- 프론트엔드: React(Next.js), TypeScript, Tailwind/Chakra, WebSocket
- 백엔드: FastAPI, Pydantic, Uvicorn, PostgreSQL, Redis/RabbitMQ, S3 호환 스토리지
- 워커: Python + Playwright(Chromium/Firefox/WebKit), asyncio 기반
- 관찰성: OpenTelemetry, JSON 구조 로그, Prometheus/Grafana
- 인증/권한: OIDC(OAuth2) + JWT, RBAC

---

## 마일스톤 개요

### M0. 프로젝트 시동(1주)
- 저장소 구조 초안: `api/`, `worker/`, `web/`, `infra/`, `docs/`
- 기본 도커 이미지(Playwright 브라우저 포함) 빌드
- 공통 라이브러리 버전/포맷(Black/isort/ruff, Pre-commit) 도입
- CI 기초: Lint + 단위 테스트 실행
- 수용 기준: CI가 초록, 컨테이너 로컬 실행 가능

### M1. 백엔드 API 스켈레톤(1-2주)
- FastAPI 프로젝트 구성, 기본 엔드포인트 헬스체크
- Job/Run/Result 개체 스키마(Pydantic)
- Postgres 마이그레이션(알렘빅) 초기 테이블 생성
- JWT 베이스 인증 훅(미니멈)
- 수용 기준: /health, /jobs CRUD(메모리 or DB), 기본 인증 통과

### M2. 워커 런타임 v1 + 큐(2주)
- Redis/RabbitMQ 중 택1로 큐 통합
- Worker: Job 소비 → Playwright 실행 → 결과 JSON 한 묶음 저장
- 브라우저 풀/컨텍스트 재사용 최소 기능(스토리지 분리)
- 수용 기준: 간단한 URL 목록 크롤 Job이 성공(Trace/스크린샷 옵션)

### M3. 추출 스키마 & 동적 로딩(2주)
- CSS/Locator 기반 추출기, post-process(정규식/치환/트림)
- wait/paginate/scroll 규칙 DSL 1.0
- 오류 모델/재시도 정책 기본(네트워크/타임아웃/429)
- 수용 기준: 무한 스크롤/페이징 사이트에서 필드 추출 성공률 기준치 달성

### M4. UI v1(2주)
- 작업 생성 마법사(소스/스키마/옵션) + 미리보기
- 실행 모니터(로그 스트림, 썸네일, Trace/HAR 링크)
- 결과 목록/다운로드(JSON/CSV)
- 수용 기준: 비개발자도 UI로 Job 생성→실행→결과 다운로드 가능

### M5. 보안/운영 강화(2주)
- RBAC 역할(Owner/Editor/Viewer/Admin)
- 도메인 허용 리스트, Rate limiting
- 관찰성: 메트릭/트레이스 대시보드, 경보 룰 초안
- 수용 기준: 권한/정책 우회 테스트 통과, 주요 메트릭 대시보드 제공

### M6. 스케줄/알림 & 안정화(2주)
- 크론/주기 실행, Webhook/Email 알림
- 변경 감지(dedupeKey 기반) 옵션
- 부하/회복력 테스트, 비용 최적화(아티팩트 TTL)
- 수용 기준: 예약 잡 성공률/알림 정확도 기준 충족, 비용 보고서 초안

---

## 세부 작업 분해(예시 백로그)

- API
  - [ ] Pydantic 모델: Job/Run/Result/Template/Credential
  - [ ] 엔드포인트: POST/GET /jobs, /jobs/{id}/run, /runs/{id}, /runs/{id}/results
  - [ ] WebSocket: /runs/{id}/events
  - [ ] 인증/권한: JWT 미들웨어, RBAC 데코레이터
  - [ ] 스토리지: Postgres 리파지토리, S3 아티팩트 업로더

- Worker
  - [ ] 큐 컨슈머(ack/retry/dead-letter)
  - [ ] Playwright 런처(브라우저 풀, 컨텍스트 옵션, 프록시)
  - [ ] 동적 로딩 DSL(wait/paginate/scroll)
  - [ ] 추출기(engine + post-process)
  - [ ] 아티팩트(Trace/HAR/스크린샷) 수집
  - [ ] 관찰성(구조 로그, 메트릭, traceId=runId)

- Web(UI)
  - [ ] 프로젝트 스캐폴드(Next.js)
  - [ ] 작업 마법사 + 요소 픽커(브라우저 미리보기 iframe/WebSocket 핸드셰이크)
  - [ ] 실행 모니터(로그 스트림/타임라인/이미지 썸네일)
  - [ ] 결과 그리드/다운로드
  - [ ] 설정 화면(RBAC/Allowlist/Rate limit)

- 보안/운영
  - [ ] OIDC 연동, 비밀 금고(예: Azure Key Vault)
  - [ ] 도메인 허용/Rate limit 정책 엔진
  - [ ] 알림 커넥터(Webhook/Email)
  - [ ] 대시보드(Grafana): 성공률, 평균 실행 시간, 95p, 실패 원인 분포

---

## 데이터베이스 및 스토리지 설계(요약)

- Postgres: jobs, runs, results(meta index), templates, credentials
- S3: traces/{runId}.zip, har/{runId}.har, screenshots/{runId}/step-*.png, results/{runId}.jsonl
- 인덱스: jobId/runId 중심, 결과는 JSONB 키 인덱스 선택적 적용

---

## 수용 기준(대표)
- Job 생성→큐→워커 실행→결과 저장→UI 다운로드 경로가 E2E로 통과
- 3개 이상 동적 로딩 패턴(무한 스크롤/페이지 버튼/탭 전환)에서 성공률 ≥ 목표
- 중요 오류(NAVIGATION_ERROR/SELECTOR_TIMEOUT/429) 메시지와 해결 가이드가 UI에 노출
- Trace/HAR/스크린샷으로 문제 재현 가능

---

## 리스크와 완화
- 안티봇 차단: 동시성/지터/헤더 전략, 정책 위반 도메인 차단
- 세션 만료/2FA: storageState 재사용 + 사용자 개입 플로우(핸드오프)
- 비용 과다(아티팩트): TTL/압축/샘플링
- 법적 준수: robots/ToS 옵션, 조직 정책/감사 로그

---

## 추정 타임라인(예시, 10~12주)
- 주 1: M0
- 주 2~3: M1
- 주 4~5: M2
- 주 6~7: M3
- 주 8~9: M4
- 주 10: M5
- 주 11~12: M6 & 하드닝

---

## 다음 단계(실행 우선순위)
1) API 스켈레톤과 DB 마이그레이션 초기 세팅
2) 워커 런타임 v1 + 큐 연동으로 첫 성공 케이스 달성
3) UI 마법사 v1과 로그 스트림 연결
4) 보안/권한 최소 요건 충족(RBAC, Allowlist)
5) 성능/관찰성 대시보드로 운영 준비
