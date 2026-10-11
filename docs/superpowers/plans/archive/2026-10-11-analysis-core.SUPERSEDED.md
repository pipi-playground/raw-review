# [폐기] 분석 코어 Implementation Plan — SUPERSEDED

> **이 플랜은 폐기되었습니다. 구현하지 마세요.**
>
> 문자열 딕셔너리 `SearchSpec`과 문서 전체 문자열 `RuleFilter` 설계는 상품 일반화와 언급 단위 매칭을 표현하지 못해 폐기되었습니다.
>
> 대체:
> - 설계: `docs/superpowers/specs/2026-10-11-matching-engine-design.md`
> - 구현 플랜: `docs/superpowers/plans/2026-10-11-matching-engine.md` (1a 매칭 엔진)
> - 구현 플랜: `docs/superpowers/plans/2026-10-11-report-and-orchestration.md` (1b 리포트 + run_search)
>
> 원본 전체 내용은 git 히스토리에서 확인할 수 있습니다(커밋 `857a262`, `5cc24d6`).
>
> 자동화 도구·검색이 옛 계약(RuleFilter, FakeClassifier, 문자열 SearchSpec)을 참조하지 않도록 본문은 제거했습니다.
