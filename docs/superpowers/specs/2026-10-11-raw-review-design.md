# raw-review 설계 문서

- 작성일: 2026-10-11
- 상태: 리뷰 대기
- 개발 방식: TDD + DDD

## 1. 목적

구매 전 제품을 비교할 때 광고·협찬·블로그 포스팅을 배제하고, 커뮤니티와 쇼핑몰 실사용 후기에서 **날것의 단점**을 정확하게 취합한다. 장점은 단순하게 요약한다.

예: "아이폰17 투명케이스" 검색 시 신지모루/다이소/슈피겐 등 브랜드별로 다음을 보여준다.

- 단점: `변색 14건 — "2주 쓰니까 누렇게 떠서 버림" (디시 아이폰갤, 링크)`
- 장점: `가격 저렴(22)`, `얇음(9)`

### 1.1 성공 기준

1. 요청한 모델/형태와 다른 제품의 글이 결과에 섞이지 않는다. 아이폰17을 검색하면 17 Pro/Air/16 글이 제외되고, 투명케이스를 검색하면 가죽케이스 글이 제외된다. 골든셋 기준 관련성 precision ≥ 0.95.
2. 광고·협찬·체험단 글이 결과에서 제외된다. 골든셋 기준 광고 판별 recall ≥ 0.9.
3. 모든 단점은 원문 표현을 보존한 인용과 출처 링크를 가진다.
4. 관리자 화면에서 소스별 수집 성공/실패를 확인하고, 실패한 소스만 재시도할 수 있다.

### 1.2 범위 밖 (현재 단계)

- 다중 사용자, 회원가입, 공개 운영 (경계만 확보)
- 네이버 블로그, 광고성 포스팅 수집
- 가격 비교, 구매 링크

## 2. 운영 형태와 제약

- **개인용(비영리, 단일 사용자)을 우선**하되, 공개 서비스로 확장할 수 있는 구조로 만든다.
- 공개 운영으로 전환하기 전에 저작권·약관·개인정보 법률 검토를 먼저 받는다.
- 작성자 식별정보(닉네임, ID, 프로필, IP 표기)는 수집 단계에서 제거하고 저장하지 않는다.
- 화면에는 요약, 짧은 인용, 원문 링크만 노출한다.
- 수집은 소스별 rate limit을 지키고, 사이트가 접근을 차단하면 해당 소스를 일시 비활성화한 뒤 관리자 화면에 실패로 보고한다.

## 3. 기술 스택

| 영역 | 선택 |
|---|---|
| 백엔드 API | Python 3.12, FastAPI, Pydantic v2 |
| 워커/큐 | arq + Redis |
| DB | PostgreSQL, SQLAlchemy 2.x, Alembic |
| 수집 | httpx, Playwright(JS 렌더링이 필요한 페이지) |
| HTML 파싱 | selectolax 또는 BeautifulSoup |
| LLM(생성형) | 2-tier. 질의 이해 = 고성능 모델, 대량 글 추출 = 저비용 모델. OpenAI API(JSON schema 구조화 출력), 포트/어댑터로 교체 가능. 모델명은 설정값 |
| 결정 모델(향후) | Laya 같은 System-1 모델(상태+형식화 질문 → 확률, 생성 없음)을 광고 판별·언급 일치·aspect 분류 같은 제한된 결정에 도입 검토. **현재는 미도입**(생성형 LLM + 룰로 처리). 도입 시 보정·사람검토 임계값을 골든셋으로 검증 |
| 웹 앱 | Next.js(App Router, TypeScript), PWA(설치형 웹 앱, 반응형) |
| BFF | Next.js 서버(Route Handlers, Server Components) |
| API 계약 | FastAPI OpenAPI 스키마 → openapi-typescript로 TS 타입 생성 |
| 테스트 | pytest, testcontainers, import-linter, Vitest, MSW, Playwright(E2E) |

별도 네이티브 앱은 만들지 않는다. 모바일에서는 PWA로 설치해 사용한다.

레포 구조(모노레포):

```
raw-review/
  apps/
    api/          # FastAPI + arq 워커 (내부 API)
    web/          # Next.js 웹 앱 + BFF
  packages/
    api-types/    # OpenAPI에서 생성한 TS 타입
  docs/
```

## 4. 아키텍처

**모듈러 모놀리스 + 비동기 워커** 구조를 쓴다. Python 패키지 하나 안에 bounded context별 하위 패키지를 둔다. context 간 의존은 각 context의 공개 인터페이스(application 서비스, 도메인 이벤트)로만 허용하고, import-linter로 강제한다.

각 context는 헥사고날 구조를 따른다.

```
<context>/
  domain/         # 엔티티, 값 객체, 도메인 서비스. 외부 의존 없음
  application/    # 유스케이스, 포트(인터페이스)
  adapters/       # DB, HTTP, LLM, 수집기 구현
```

### 4.1 catalog: 검색 의도와 카테고리

이 context의 구조화 속성 모델과 언급 단위 매칭은 복잡도가 커서 **별도 하위프로젝트(매칭 엔진)로 분리**한다. 상세 설계는 `docs/superpowers/specs/2026-10-11-matching-engine-design.md`를 따른다. 아래는 요지다.

- **Category**: 관리자가 등록하지 않는다. 검색 시점에 판별하고, 처음 등장한 카테고리는 DB에 캐시한다.
- **AttributeSchema**: 카테고리 독립적인 구조화 속성 정의. 문자열 딕셔너리가 아니라 속성마다 매칭 규칙을 명시한다.
  - 각 `AttributeDef`: `name`, `hard|soft`, `match_mode`(EXACT·ENUM·SYNONYM·NUMERIC·RANGE·SUBSTRING·FREETEXT), `canonical_values`, `aliases`, `exclusions`(속성 단위 제외값), `unit`(숫자/범위용), `normalization`(정규화 규칙).
  - `category_aliases`로 카테고리명 별칭을 둔다. 예: 양말 = 삭스 = socks, 자전거용 장갑 = 라이딩 글러브.
  - 예: 휴대폰케이스(대상기기=EXACT, 형태=ENUM), 울양말(소재=ENUM, 종류=ENUM), 부츠(종류=ENUM, 계절=ENUM), 운동화(사이즈=RANGE+unit mm).
- **SearchSpec**: 사용자 질의를 파싱한 구조화 결과. 전역 `exclusion_variants`를 버리고 제외를 **속성 단위**로 둔다.
  ```
  category: 휴대폰케이스
  constraints:
    - {name: 대상기기, hard: true, match_mode: EXACT,
       wanted: [아이폰17], aliases: {아이폰17: [iphone17, 아17]},
       exclusions: [17 Pro, 17 Pro Max, 17 Air]}
    - {name: 형태, hard: true, match_mode: ENUM,
       wanted: [투명], aliases: {투명: [클리어]}, exclusions: [가죽]}
  ```
- **CategoryResolver**(생성형 LLM 고tier 포트 사용): 질의 → 구조화 `SearchSpec`.
  - 상품명 분해 규칙과 `category_aliases`로 "메리노삭스"→양말+소재:울, "라이딩 글러브"→자전거용 장갑처럼 연결한다.
  - 해당 카테고리 `AttributeSchema` 캐시가 있으면 재사용한다. 없으면 LLM이 스키마를 생성해 캐시한다.
  - 신뢰도가 임계값 미만이면 `AmbiguousQuery`로 되묻는다. 예: "키보드" → 기계식/멤브레인/태블릿용.
- **Brand / ProductLine**: 카테고리 단위 별칭 사전. 예: 슈피겐 = spigen.

### 4.2 collection: 수집

- **Source**: 수집 대상 사이트. 두 종류가 있다.
  - `CommunitySource`: 디시인사이드(갤러리 단위), 클리앙, 뽐뿌, 퀘이사존, 루리웹, FM코리아, 쿨앤조이, 아카라이브
  - `ShoppingReviewSource`: 네이버 쇼핑 리뷰. 별점이 있으며, 단점 수집은 저별점 리뷰를 우선한다.
  - 네이버 쇼핑은 약관과 접근 정책을 확인한 뒤 진행하며, 우선순위가 가장 낮다.
- **SourceAdapter**(포트): 소스마다 하나씩 구현한다.
  - `build_queries(spec) -> list[str]`: SearchSpec의 별칭과 핵심어로 사이트 검색어를 만든다.
  - `search(query) -> list[PostRef]`
  - `fetch_post(ref) -> RawPost`: 본문과 댓글을 함께 가져온다.
  - 파싱 로직은 순수 함수 `parse_list(html)`, `parse_post(html)`로 분리해서 fixture 테스트를 할 수 있게 한다.
- **Fetcher**(포트): HTML을 가져오는 일만 담당한다.
  - `HttpFetcher`: httpx 기반 기본 구현
  - `BrowserFetcher`: Playwright 기반. JS 렌더링이 필요한 페이지에 쓴다.
  - 소스별로 사용할 Fetcher를 설정으로 지정한다.
- **RateLimiter**: 소스별 토큰버킷으로 요청 간격과 동시성을 제한한다.
- **CrawlJob / CollectTask**: 검색 1회가 `CrawlJob` 1개이고, 그 안에 소스별 `CollectTask`가 있다. 각 task는 상태(`pending`/`running`/`succeeded`/`partial`/`failed`), 수집 건수, 실패 사유, 시작·종료 시각을 가진다.
- **FailureReason**(값 객체): `HTTP_4XX`, `HTTP_5XX`, `BLOCKED`, `CAPTCHA`, `PARSE_ERROR`, `TIMEOUT`, `ZERO_RESULTS`
- **RawPost / RawComment**: 정규화된 원문 데이터. 필드는 소스, URL, 제목, 본문, 작성일, 조회수/추천수(있으면), 별점(쇼핑), 댓글 목록. 작성자 식별정보는 `PiiScrubber`가 저장 전에 제거한다.
- **SourceHealth**: 소스별 최근 성공률, 마지막 성공 시각, 활성/일시비활성 상태. 연속 실패나 `BLOCKED`/`CAPTCHA`가 나오면 일정 시간 비활성화한다.
- **HtmlSnapshot**: `PARSE_ERROR`가 나면 원본 HTML을 저장한다. 관리자 화면에서 테스트 fixture로 내보낼 수 있다.

### 4.3 analysis: 판별과 의견 추출

문서 전체를 문자열로 검사하지 않고 **상품 언급 단위**로 처리한다. 상세는 매칭 엔진 하위프로젝트 spec을 따른다. 요지:

1. **CoarseGate**(순수 룰, LLM 없음): 글에 카테고리/속성 키워드가 하나도 없으면 LLM 호출 없이 즉시 탈락시킨다. 비용 1차 방어.
2. **MentionExtractor**(생성형 LLM 저tier 포트): 본문·댓글에서 `ProductMention[]`을 추출한다. 각 언급은 `{원문조각, 브랜드, 속성값(name→값), 그 언급에 붙은 의견(polarity·aspect·quote·source_part)}`.
3. **MentionMatcher**(순수 룰, 결정적): 각 `ProductMention`을 `SearchSpec.constraints`와 `match_mode`로 비교한다.
   - 하드 속성이 모두 일치하고, 속성 단위 `exclusions`에 걸리지 않는 언급만 대상으로 채택한다.
   - **제외는 언급 단위다.** "면양말은 보풀, 울양말은 따뜻"에서 면양말 언급만 버리고 울양말 의견은 살린다. "17 Pro는 안 맞고 17 투명케이스는 변색"도 Pro 언급만 버린다.
   - 퍼지 영역(FREETEXT·SYNONYM·애매 속성)은 현재 생성형 LLM 판정으로 처리한다. 향후 Laya 같은 결정 모델로 교체 검토.
4. **광고 판별**: 룰 광고 신호 점수와 LLM 광고 판정을 OR로 합친다. 댓글의 "광고네/뒷광고"도 신호로 쓴다.
5. **결과**: 채택된 언급의 의견만 `Opinion[]`으로 남긴다.
   - `Opinion.quote`는 원문 표현 그대로 저장하되 최대 길이를 자른다(예: 120자). 단점은 인용 필수, 장점은 `aspect` 태그 집계.
- **포트**: 생성형 LLM은 포트 뒤에 두고, 어댑터는 `OpenAi*`(고tier 질의이해 / 저tier 추출), 테스트는 fake. 응답이 schema 검증에 실패하면 1회 재시도 후 그 글을 `UNANALYZED`로 둔다.
- **AspectNormalizer**: "누래짐/황변/변색"처럼 같은 뜻 aspect를 카테고리 단위로 정규화한다.

### 4.4 review: 리포트

- **ProductReport**: SearchSpec 하나에 대한 브랜드별 집계.
  - 브랜드마다: 단점 aspect별 언급 수, 대표 인용 3~5개(출처, 링크, 작성일), 장점 태그별 언급 수, 분석에 쓴 글 수, 광고로 제외된 글 수
  - 소스별 수집 상태 요약
  - 생성 시각, TTL(기본 7일)
- **ReportBuilder**(도메인 서비스): `Opinion[]`을 받아 `ProductReport`를 만든다. 순수 함수로 구현한다.
- 단점 정렬은 언급 수 → 최근성 → 추천수 순서로 한다.

### 4.5 identity: 인증

- 현재는 단일 계정과 JWT만 쓴다.
- `User`와 `Role(user/admin)` 모델 경계는 미리 만들어 두어 공개 전환 시 확장한다.

### 4.6 BFF (Backend For Frontend)

브라우저는 Next.js 서버(BFF)와만 통신한다. FastAPI는 내부 API로, 외부에 직접 노출하지 않는다.

- **역할**
  - 인증: 로그인 시 FastAPI가 발급한 JWT를 BFF가 httpOnly·Secure·SameSite 쿠키로 보관한다. 브라우저 JS는 토큰에 접근하지 않는다.
  - 화면 단위 응답 조립: 화면에 필요한 형태로 FastAPI 응답을 합치고 줄인다. 예: 리포트 화면은 리포트와 소스 상태를 한 번에 받는다.
  - 폴링 중계: 브라우저는 BFF를 폴링하고, BFF가 FastAPI job 상태를 조회한다.
  - 관리자 경로 보호: `/admin` 화면과 BFF 관리자 엔드포인트에서 역할을 검사한다. FastAPI에서도 한 번 더 검사한다.
  - 오류 변환: FastAPI 오류를 화면에서 쓰는 오류 코드와 메시지로 변환한다.
- **도메인 로직은 두지 않는다.** BFF는 조립, 변환, 인증 중계만 맡는다. 판별, 집계 같은 규칙은 모두 FastAPI에 있다.
- **계약**: FastAPI OpenAPI 스키마에서 `packages/api-types`를 생성한다. BFF는 생성된 타입으로만 FastAPI를 호출한다. 스키마가 바뀌면 CI에서 타입 생성 결과의 diff로 감지한다.
- **배포**: BFF와 FastAPI는 같은 내부 네트워크에 둔다. FastAPI는 BFF에서 오는 요청만 받는다(내부 네트워크 + 서비스 토큰).

## 5. 데이터 흐름

1. 브라우저가 BFF의 `POST /api/searches {query}`를 호출하고, BFF가 FastAPI의 `POST /searches`로 전달한다. 이후 단계의 응답도 BFF를 거쳐 브라우저로 간다.
2. `CategoryResolver`가 `SearchSpec`을 만든다. 애매하면 `409 + 선택지`를 반환한다.
3. 같은 SearchSpec의 유효한 `ProductReport`가 있으면 `200 + 리포트`를 반환한다.
4. 없으면 `CrawlJob`을 만들어 큐에 넣고 `202 + jobId`를 반환한다.
5. 워커가 활성 상태인 소스의 `CollectTask`를 병렬로 실행한다. 각 task는 검색어 생성, 검색, 글+댓글 수집, PII 제거, `RawPost` 저장 순서로 진행한다.
6. 수집된 글마다 analysis 파이프라인(RuleFilter → LlmJudge)을 실행한다.
7. 모든 task가 끝나면(부분 실패 포함) `ReportBuilder`가 리포트를 만들고 job을 완료 처리한다.
8. 브라우저는 BFF의 `GET /api/searches/{jobId}`를 폴링한다. 이후 SSE 또는 웹 푸시(PWA)를 검토한다.

## 6. API

### 6.1 BFF 엔드포인트 (브라우저용)

| Method | Path | 설명 |
|---|---|---|
| POST | `/api/auth/login` | 로그인, httpOnly 쿠키 설정 |
| POST | `/api/auth/logout` | 쿠키 삭제 |
| POST | `/api/searches` | 검색 |
| GET | `/api/searches/{jobId}` | job 상태, 완료 시 리포트 화면 데이터 |
| GET | `/api/admin/*` | 관리자 화면 데이터 (아래 내부 API를 화면 단위로 조립) |
| POST | `/api/admin/jobs/{jobId}/retry` | 실패 task 재시도 |

### 6.2 FastAPI 내부 API (BFF 전용)

| Method | Path | 설명 |
|---|---|---|
| POST | `/searches` | 검색. 200(캐시) / 202(job 생성) / 409(애매) |
| GET | `/searches/{jobId}` | job 상태와 완료 시 리포트 |
| GET | `/reports/{reportId}` | 리포트 조회 |
| GET | `/admin/sources` | 소스별 health |
| PATCH | `/admin/sources/{id}` | 소스 활성/비활성 |
| GET | `/admin/jobs` | job 목록(필터: 상태, 소스) |
| GET | `/admin/jobs/{jobId}` | 소스별 task 상세, 실패 사유 |
| POST | `/admin/jobs/{jobId}/retry` | 실패한 task만 재시도(소스 지정 가능) |
| GET | `/admin/snapshots/{id}` | 파싱 실패 HTML 스냅샷 다운로드 |

## 7. 관리자 화면

- **소스 대시보드**: 소스별 최근 24시간/7일 성공률, 마지막 성공 시각, 활성 상태, 최근 실패 사유 분포
- **job 목록과 상세**: 소스별 성공/부분/실패, 수집 건수, 실패 사유, 소요 시간
- **재시도**: 실패한 소스만 골라 재시도
- **스냅샷**: 파싱 실패 HTML을 내려받아 fixture로 추가

## 8. 에러 처리

- 소스 하나가 실패해도 job은 계속 진행한다. 리포트에 실패한 소스를 표시한다.
- 네트워크 오류와 5xx는 지수 백오프로 최대 3회 재시도한다.
- `BLOCKED`/`CAPTCHA`는 재시도하지 않는다. 해당 소스를 일시 비활성화하고 관리자 화면에 보고한다.
- `PARSE_ERROR`가 나면 HTML 스냅샷을 저장하고 해당 글을 건너뛴다.
- LLM 오류: 타임아웃과 rate limit은 재시도한다. schema 검증 실패는 1회 재시도 후 `UNANALYZED`로 처리한다.
- LLM 비용 상한: job당 최대 분석 글 수와 토큰 상한을 설정으로 둔다.

## 9. 테스트 전략

- **도메인 단위테스트(TDD 주력)**: SearchSpec 매칭, RuleFilter, 광고 신호 점수, AspectNormalizer, ReportBuilder 집계, CrawlJob 상태 전이, SourceHealth 비활성화 규칙. 외부 의존 없이 빠르게 돌린다.
- **파서 fixture 테스트**: 소스별로 실제 HTML을 `tests/fixtures/<source>/`에 저장하고 `parse_list`/`parse_post` 결과를 검증한다. 어댑터는 fixture부터 만들고 시작한다.
- **LLM 포트 테스트**: 단위테스트는 `FakeClassifier`를 쓴다. OpenAI 어댑터는 요청 생성과 응답 파싱만 계약 테스트한다.
- **골든셋 평가**: 라벨링한 실제 글 100개 이상(여러 카테고리, 광고/비광고, 모델 일치/불일치 포함)으로 관련성, 광고 판별, 단점 추출의 precision/recall을 측정한다. 수동으로 또는 nightly로 실행하며, 1.1의 임계값을 밑돌면 실패로 처리한다.
- **통합테스트**: FastAPI + testcontainers(PostgreSQL, Redis). 수집은 fixture를 돌려주는 `FakeFetcher`로 대체한다.
- **BFF 테스트**: Vitest + MSW로 FastAPI 응답을 모킹하고, 응답 조립, 오류 변환, 쿠키 처리, 관리자 권한 검사를 검증한다.
- **계약 테스트**: FastAPI OpenAPI 스키마와 `packages/api-types` 생성 결과가 일치하는지 CI에서 검사한다.
- **E2E**: Playwright로 브라우저 → BFF → FastAPI 전체 경로를 검증한다. FastAPI는 fake 소스 모드로 띄운다.
- **경계 테스트**: import-linter 계약을 CI에서 검사한다.

## 10. 로드맵

각 단계는 별도 구현 플랜으로 진행한다. 1단계는 다시 하위프로젝트로 분해해 각자 spec을 가진다.

1. **분석 코어**
   1a. **매칭 엔진**(하위프로젝트, 자체 spec `2026-10-11-matching-engine-design.md`): 구조화 `AttributeSchema`/`SearchSpec`, match_mode 매처, `MentionExtractor`+`MentionMatcher`, CategoryResolver(2-tier LLM), AspectNormalizer, 카테고리별 골든셋. 가장 많은 테스트가 필요한 핵심.
   1b. **리포트**: `ReportBuilder`(브랜드별 집계), 채택 언급 → `ProductReport`.
2. **수집**: Fetcher/RateLimiter/SourceHealth, CrawlJob 상태 관리, 소스 어댑터. 순서는 디시·클리앙·뽐뿌 → 퀘이사존·루리웹·FM코리아·쿨앤조이·아카라이브 → 네이버 쇼핑(정책 확인 후). 실제 라벨로 골든셋 100건+ 확장, LLM precision/recall 게이트 가동.
3. **워커 + 내부 검색 API + 캐시 + 관리자 API**, OpenAPI 타입 생성 파이프라인
4. **Next.js 웹 앱 + BFF**: 인증 중계, 검색 화면, 리포트 화면, 관리자 화면
5. **PWA**: 설치 지원(manifest, service worker), 모바일 반응형 다듬기, 웹 푸시 검토
6. **(향후 최적화)** Laya 같은 결정 모델 도입: 광고 판별·언급 일치·aspect 분류를 System-1로. 보정·사람검토 임계값을 골든셋으로 검증 후 전환.

## 11. 열린 질문

- 네이버 쇼핑 리뷰의 접근 정책 (2단계 후반 전 확인)
- 리포트 TTL 기본값(7일)이 카테고리별로 달라야 하는지 (운영하면서 조정)
