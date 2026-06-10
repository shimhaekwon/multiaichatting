# multiaichatting — 다자(多者) AI 협업·대화 설계 리포지토리

여러 AI(와 사람)가 함께 대화·협업하는 **제품군의 요구사항·설계 산출물**을 모은 리포지토리입니다.
하나의 제품이 아니라, 같은 DNA(멀티-AI 대화 + 귓속말/상호검토)를 공유하며 발전한 **3개 제품 라인**을 담고 있습니다.

## 제품군 (3 라인)

| 라인 | 제품 | 한 줄 | 형태 | AI 연결 | 상태 | 설계 폴더 |
|------|------|-------|------|---------|------|-----------|
| **VER-002** | **multichat** | 6종 클라우드 LLM **병렬 라운드** 협업 | 크롬 확장(MV3) | 브라우저 로그인 탭 DOM(무키) | 설계 완료 · 구현=사용자 | `210.설계산출물-multichat` |
| **VER-003** | **ai-sarangbang** | 로컬 AI들과의 라이브 **턴제** 채팅방 | 로컬 웹앱 | AgentDriver(Mock→Ollama→API) | **구현·라이브검증 완료** | `220.설계산출물-ai-sarangbang` |
| **VER-230** | **ai-chatting-cloud** | ai-sarangbang에 **무료 클라우드 LLM** 추가 | 로컬 웹앱 확장 | 무료 키(Gemini·OpenRouter)+Vite proxy | 설계 v1.1 · 구현 단계 | `230.설계산출물-ai-chatting-cloud` |

## 리포지토리 구조

```
multiaichatting/
├─ 100.START/                        요구사항 (버전별)
│   ├─ VER-001 / VER-002-multichat
│   ├─ VER-003-ai-sarangbang
│   └─ VER-230-ai-chatting-cloud
├─ 200.설계산출물-초안/                multichat 추상 설계 (201~260, 감사·대조용)
├─ 210.설계산출물-multichat/          multichat 구현적용판 (211~224, ⭐222 확정 우선)
├─ 220.설계산출물-ai-sarangbang/      ai-sarangbang 설계 정본 (221~228 + CHANGELOG)
├─ 230.설계산출물-ai-chatting-cloud/  클라우드 확장 설계 (231~232)
├─ 800.테스트/                        테스트 문서
├─ 900.AICOWORK/                      작업 처리내역 (세션 로그)
├─ docs/ · start/                     초기 요구사항·브레인스토밍
└─ README.md                          (이 문서)
```

---

## 1. multichat (VER-002) — 6종 클라우드 LLM 병렬 협업 · 크롬 확장

6종 LLM(**ChatGPT · Claude · Gemini · Perplexity · DeepSeek · Grok**)과 사용자가 함께 협업하는 **브라우저 세션 기반 멀티 LLM 협업 플랫폼**.

- **API 키 불필요** — 크롬 로그인 세션 + 각 사이트 탭 DOM 어댑터(`DomAdapterBot`)
- **라운드 기반 병렬** — R1(Blind 독립응답) → R2(상호검토, 자기 제외 YAML) → R3(최종답변), 동기화 배리어
- 라운드 콤보 {3,5,7,9} · 토큰 soft 지시문 {100…10000} · 진행 중 STOP/연장 · LLM swap · 비밀채팅(휘발) · MD 저장(IndexedDB)
- **베이스**: ChatHub **v1.45.7** 개조 (React 18 + TS + Vite + Jotai + Tailwind, MV3) — 구현 코드 `E:\workspace\multichat`
- ⚠️ **Phase 0 실증(2026-05-29)**: 1.45.7 내부 API 봇이 현재 사이트와 불일치(전부 실패) → 하이브리드 폐기, **6종 전부 DOM 어댑터로 통일**
- **구현 진입**: [`210.설계산출물-multichat/README.md`](210.설계산출물-multichat/README.md) → `211`(개요) 순, **충돌 시 `222-구현확정사항`이 우선**

## 2. ai-sarangbang (VER-003) — 로컬 턴제 AI 채팅방 · 구현 완료 ✅

AI 여러 명과 사람(나)이 함께 들어가는 **라이브 턴제 채팅방** — 90년대 PC통신(하이텔·나우누리) 감성의 "사랑방".

- **핵심 규칙 4**: ① 턴제 1회 발언 ② **floor 직렬화(랜덤 순서)** ③ 큐 소진까지 ④ 귓속말(휘발)
- **AgentDriver 추상화**(`speak(ctx,signal)`) — Mock → **Ollama**(localhost) → API 무변경 교체. **로컬 웹앱**(Vite+React+TS), Coordinator는 순수 TS 모듈(멀티유저 이식 대비)
- **227 (v1.1)**: 속도경쟁 floor → **랜덤 순서 직렬** + Roster 순번 배지 + 자동대화 **바퀴 로테이션·라운드 종료선** → 구현·라이브검증·정본 정합 완료
- **228 (v1.0)**: 진입 **로비**(런타임 Ollama 모델 선택 `/api/tags`·인원 가감·영속) + **Ollama 연결 보강**(주소 지정·타임아웃·진단·재연결) → 구현·라이브검증 완료
- **구현 코드**: `E:\workspace\ai-sarangbang` (독립 git repo)
- **상세**: [`220.설계산출물-ai-sarangbang/README.md`](220.설계산출물-ai-sarangbang/README.md) · 변경이력 [`CHANGELOG.md`](220.설계산출물-ai-sarangbang/CHANGELOG.md)

## 3. ai-chatting-cloud (VER-230) — 무료 클라우드 LLM 확장

ai-sarangbang(로컬 Ollama 턴제 채팅)에 **클라우드 LLM을 무료·합법으로 참여자 추가**하는 확장 설계.

- **결론**: "API 키 없이 채팅 계정으로"는 기술 차단(httpOnly·CORS·봇탐지) + 3사 약관 위반 + 계정 정지 위험 + v1 폐기 방식 회귀로 **불가** → **무료 API 키**(Gemini 무료 티어 + OpenRouter 무료 모델) + **Vite proxy/.env 키 보관**으로 비용 0 목적 달성
- **통합 모델 선택 UI**(로컬+클라우드 한 목록, value=`provider::model`) · **Ollama-off 클라우드-only 입장** · **키 클라이언트 노출 0** · core 순수성 유지 · 회귀 0
- **상세**: [`230.설계산출물-ai-chatting-cloud/README.md`](230.설계산출물-ai-chatting-cloud/README.md) · 타당성 검토 정본 = `100.START/VER-230-ai-chatting-cloud/10-검토결과-및-방향.md`

---

## 공통 DNA / 계보

```
multichat (VER-002, 크롬확장·병렬)
   │  데이터모델 · context빌더 · MD포맷 · 상태머신 · 귓속말(비밀채팅) 만 순수 이식
   │  (DOM·chrome·스크래핑·throttle 대응은 제외)
   ▼
ai-sarangbang (VER-003, 로컬웹앱·턴제)  ──확장──▶  ai-chatting-cloud (VER-230, 무료 클라우드 키)
```

## 진행 상태 (전체)

- [x] **multichat**: 요구사항·설계 완료 · Phase 0 실증(DOM 전환) · 구현=사용자(boundary)
- [x] **ai-sarangbang**: 설계 + **구현·라이브검증 완료** (227 랜덤직렬 floor · 228 진입 로비)
- [ ] **ai-chatting-cloud**: 설계 v1.1 완료 · 구현 단계(무료 키 + Vite proxy)

## 개발 원칙

- **Step by step** (단계별 검증 후 진행) · **No guess** (합의되지 않은 추정 구현 금지)
- **빠짐없이 / 틀림없이 / 다름없이 / 더함없이**
- **설계 자기완결성** — 구현자는 해당 제품의 설계 폴더만으로 구현 가능. 문서 충돌 시 각 폴더의 **확정/정본 문서가 우선**
- 재사용 가능한 설계 템플릿 자산화

## 참고

- multichat 베이스: [ChatHub](https://chathub.gg) (크롬 확장, v1.45.7) · 구현 `E:\workspace\multichat`
- ai-sarangbang 구현: `E:\workspace\ai-sarangbang`
- 각 제품 상세는 위 설계 폴더의 `README.md` 참조
