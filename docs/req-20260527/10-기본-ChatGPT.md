# Multi-LLM Cowork System — 고객 요구사항 개요 v2.0

## 목적
사용자가 여러 LLM(ChatGPT, Claude, Gemini)을 동시에 참여시켜,
라운드 기반 협업형 질의응답을 수행하는 Chrome Extension 기반 로컬 전용 시스템 구축.

---

# 핵심 개념

## 1. API 없는 LLM 협업
- Chrome 로그인 세션 기반 사용
- API Key 미사용
- 사용자의 실제 브라우저 로그인 세션 활용

지원 대상:
- ChatGPT
- Claude
- Gemini

---

## 2. 다중 LLM 동시 참여
사용자는 최소 2개 이상의 LLM 선택 가능.

지원 기능:
- 전체 선택
- 일부 선택
- 세션 유지
- 채팅 중 LLM 추가/제외

---

## 3. 라운드 기반 협업
기본 3회 라운드 진행.

### Round 1
- 각 LLM 독립 응답
- 타 LLM 응답 미공유

### Round 2
- 다른 LLM 응답 검토
- 상호 피드백

### Round 3
- 최종 답변 생성

최대 9회까지 연장 가능.

---

## 4. 사용자 중심 제어
사용자가:
- 조기 종료 가능
- 회차 연장 가능
- 특정 LLM 재질의 가능
- 다음 요청 미리 작성 가능

최종 판단자는 사용자.

---

## 5. 로컬 저장
서버 저장 없음.

저장 방식:
- IndexedDB 자동 저장
- Markdown 파일 저장
- 이력 검색 지원

---

## 6. 기술 조건
- Chrome Extension (Manifest V3)
- Windows/macOS/Linux 지원
- 모바일 미지원
- 단일 사용자 전용

---

## 7. 개발 원칙
- step by step
- no guess
- 빠짐없이 / 틀림없이 / 다름없이 / 더함없이
- 재사용 가능한 설계 자산화

---

# 참고 시스템
- ChatHub Chrome Extension
- 기존 코드베이스:
  `E:\workspace\chathub`

