# 230.설계산출물 — ai-chatting-cloud (VER-230)

ai-sarangbang(로컬 Ollama 턴제 채팅)에 **클라우드 LLM을 무료·합법으로 참여자 추가**하는 확장의 설계 산출물.

## 한 줄 결론

요청한 "API 키 없이 채팅 계정으로"는 **기술 차단(httpOnly·CORS·봇탐지) + 3사 약관 위반 + 계정 정지 위험 + v1 폐기 방식 회귀**로 불가 →
**무료 API 키(Gemini 무료 티어 + OpenRouter 무료 모델)** + **Vite proxy/.env 키 보관**으로 실질 목적(비용 0)을 달성.

## 문서 인덱스

| 문서 | 내용 |
|---|---|
| `100.START/VER-230-ai-chatting-cloud/user-request.txt` | 원시 요청 |
| `100.START/VER-230-ai-chatting-cloud/10-검토결과-및-방향.md` | **타당성 검토 정본**(근거·약관·대안·결정 로그) |
| `100.START/VER-230-ai-chatting-cloud/20-요구사항.md` | FR/NFR·수용 기준 |
| `230.…/231-설계-아키텍처-데이터-보안-UI.md` | CloudDriver·provider·프록시·키 보안·데이터모델·로비 |
| `230.…/232-구현계획.md` | 슬라이스 S1~S6·게이트·회귀·라이브 실증·롤백 |

## 상태

- [x] 타당성 검토 (사용자 검토요청 응답)
- [x] 방향 확정 (무료 키 + proxy/.env, 설계 먼저)
- [x] 요구사항·설계 산출물
- [ ] 구현 (사용자 OK 후 S1~S6)
- [ ] 라이브 실증·push

## 핵심 제약

- 공식 API만(세션 재사용/스크래핑 0) · core 순수성 유지 · 키 클라이언트 노출 0 · 회귀 0 · prod 백엔드는 follow-up.

## 버전 이력

- v1.0 (2026-06-08): 검토·요구사항·설계 초안. 구현 미착수.
- v1.1 (2026-06-08): **통합 모델 선택 UI**(로컬+클라우드 한 목록, value=`provider::model`)·**Ollama-off 클라우드-only 입장** 반영. **uw-critic 2회 적대검토**(231 §7 1차·§8 2차) 반영 — 배지 헬스 프로브 정정(OpenRouter `/api/v1/key`)·스키마 bump 테스트 마이그레이션·`gone` 판정·모델 ID '확정' 철회. 구현 미착수.
