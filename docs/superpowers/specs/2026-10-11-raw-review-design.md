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
| LLM | OpenAI API(구조화 출력, JSON schema). 포트/어댑터 구조로 교체 가능 |
| 웹 | Next.js(App Router, TypeScript) |
| 앱(후순위) | Expo(React Native), 웹과 같은 REST API 사용 |
| 테스트 | pytest, testcontainers, import-linter, Vitest, Playwright(E2E) |

레포 구조(모노레포):

```
raw-review/
  apps/
    api/          # FastAPI + arq 워커
    web/          # Next.js
    mobile/       # (후순위) Expo
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

- **Category**: 관리자가 등록하지 않는다. 검색 시점에 판별하고, 처음 등장한 카테고리는 DB에 캐시한다.
- **AttributeSchema**: 카테고리별로 매칭에 쓰는 속성을 정의한다. 각 속성은 `hard`(반드시 일치)와 `soft`(가중치)로 나뉜다.
  - 휴대폰케이스: 대상기기(hard), 형태(hard: 투명/범퍼/카드/가죽 등)
  - 청소기: 형태(hard: 무선스틱/로봇/유선), 모델명(hard)
  - 매트리스: 사이즈(soft), 경도(soft)
  - 키보드: 배열(hard), 스위치(soft)
- **SearchSpec**: 사용자 질의를 파싱한 결과. 예를 들어 "아이폰17 투명케이스"는 다음과 같이 파싱된다.

  ```
  category: 휴대폰케이스
  hard_attributes: {대상기기: 아이폰17, 형태: 투명}
  soft_attributes: {}
  exclusion_variants: [17 Pro, 17 Pro Max, 17 Air, 16, 가죽]
  aliases: {아이폰17: [아이폰 17, 아17, iphone17, iphone 17]}
  ```

- **CategoryResolver**(도메인 서비스, LLM 포트 사용): 질의를 받아 `SearchSpec`을 만든다.
  - 해당 카테고리의 `AttributeSchema` 캐시가 있으면 재사용한다.
  - 처음 보는 카테고리면 LLM이 스키마를 생성하고 캐시에 저장한다.
  - 판별 신뢰도가 임계값 미만이면 선택지를 담은 `AmbiguousQuery`를 반환해 사용자에게 되묻는다. 예: "키보드" → 기계식/멤브레인/태블릿용
- **Brand / ProductLine**: 카테고리 단위로 관리하는 별칭 사전. 예: 슈피겐 = spigen. 별칭은 LLM 추출 결과를 사전으로 정규화할 때 쓴다.

### 4.2 collection: 수집

- **Source**: 수집 대상 사이트. 두 종류가 있다.
  - `CommunitySource`: 디시인사이드(갤러리 단위), 클리앙, 뽐뿌, 퀘이사존, 루리웹, FM코리아, 쿨앤조이, 아카라이브, 어미새
  - `ShoppingReviewSource`: 네이버 쇼핑 리뷰. 별점이 있으며, 단점 수집은 저별점 리뷰를 우선한다.
  - 어미새는 핫딜 모음 사이트일 가능성이 있어 구현 전에 사이트 구조를 확인한다. 원 출처 링크를 따라가는 방식이 될 수 있다.
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

글 하나마다 다음 파이프라인을 거친다.

1. **RuleFilter**(순수 도메인, LLM 없음)
   - 제목과 본문에 `exclusion_variants`만 나오고 대상 모델은 나오지 않으면 즉시 탈락시킨다.
   - hard 속성의 핵심어와 별칭이 하나도 없으면 탈락시킨다.
   - 광고 신호 점수를 매긴다: 협찬/제공/체험단/원고료 문구, 제휴·단축 링크, 쿠폰코드, 쇼핑 리뷰의 포인트 리뷰 문구, 복붙형 리뷰(동일 문장 반복).
   - 결과는 `PASS` / `REJECT(reason)` / `UNCERTAIN` 중 하나다.
2. **LlmJudge**(LLM 포트): `PASS`와 `UNCERTAIN`만 처리한다. 본문과 댓글 일부, SearchSpec을 넣고 JSON schema 출력을 받는다.
   ```
   relevance: {matches: bool, matched_attributes, reason}
   ad: {is_ad: bool, confidence, evidence}
   opinions: [{brand, product_line, polarity: con|pro, aspect, quote, source_part: post|comment}]
   ```
   - 댓글에 "광고네", "뒷광고" 같은 반응이 있으면 광고 근거로 쓴다.
   - 댓글에서 나온 의견도 추출하되, 같은 모델/형태에 대한 의견일 때만 인정한다.
3. **결과**: `RelevanceVerdict`, `AdVerdict`, `Opinion[]`
   - `Opinion.quote`는 원문 표현 그대로 저장하되 최대 길이를 정해 자른다(예: 120자).
   - 장점은 `aspect` 태그로만 집계하고 인용은 선택 사항이다.
   - 단점은 인용이 필수다.
- **LlmClassifier 포트**: 기본 어댑터는 `OpenAiClassifier`이고, 테스트에서는 `FakeClassifier`를 쓴다. 응답이 schema 검증에 실패하면 1회 재시도하고, 또 실패하면 그 글을 `UNANALYZED`로 표시한다.
- **AspectNormalizer**: "누래짐", "황변", "변색"처럼 같은 뜻의 aspect를 카테고리 단위로 정규화한다. 처음에는 LLM 출력을 쓰고, 정규화 사전을 캐시해 점차 일관성을 높인다.

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

## 5. 데이터 흐름

1. 클라이언트가 `POST /searches {query}`를 호출한다.
2. `CategoryResolver`가 `SearchSpec`을 만든다. 애매하면 `409 + 선택지`를 반환한다.
3. 같은 SearchSpec의 유효한 `ProductReport`가 있으면 `200 + 리포트`를 반환한다.
4. 없으면 `CrawlJob`을 만들어 큐에 넣고 `202 + jobId`를 반환한다.
5. 워커가 활성 상태인 소스의 `CollectTask`를 병렬로 실행한다. 각 task는 검색어 생성, 검색, 글+댓글 수집, PII 제거, `RawPost` 저장 순서로 진행한다.
6. 수집된 글마다 analysis 파이프라인(RuleFilter → LlmJudge)을 실행한다.
7. 모든 task가 끝나면(부분 실패 포함) `ReportBuilder`가 리포트를 만들고 job을 완료 처리한다.
8. 클라이언트는 `GET /searches/{jobId}`를 폴링한다. 앱 단계에서 SSE나 푸시를 검토한다.

## 6. API

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
- **E2E**: Next.js + Playwright. 백엔드는 fake 소스 모드로 띄운다.
- **경계 테스트**: import-linter 계약을 CI에서 검사한다.

## 10. 로드맵

각 단계는 별도 구현 플랜으로 진행한다.

1. **분석 코어**: catalog(SearchSpec, CategoryResolver), analysis(RuleFilter, LlmJudge, AspectNormalizer), review(ReportBuilder), OpenAI 어댑터, 골든셋 평가 하네스
2. **수집**: Fetcher/RateLimiter/SourceHealth, CrawlJob 상태 관리, 소스 어댑터. 순서는 디시·클리앙·뽐뿌 → 퀘이사존·루리웹·FM코리아·쿨앤조이·아카라이브·어미새 → 네이버 쇼핑(정책 확인 후)
3. **워커 + 검색 API + 캐시 + 관리자 API**
4. **Next.js 웹**: 검색 화면, 리포트 화면, 관리자 화면, 최소 인증
5. **앱(Expo)**

## 11. 열린 질문

- 어미새의 사이트 구조와 수집 방식 (2단계 시작 전 확인)
- 네이버 쇼핑 리뷰의 접근 정책 (2단계 후반 전 확인)
- 리포트 TTL 기본값(7일)이 카테고리별로 달라야 하는지 (운영하면서 조정)
