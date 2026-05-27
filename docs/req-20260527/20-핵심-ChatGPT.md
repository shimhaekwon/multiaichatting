# Multi-LLM Cowork System — 핵심 고객 요구사항 v2.0

# 1. 시스템 목적

본 시스템은 Chrome 브라우저 로그인 세션을 활용하여
복수의 LLM(ChatGPT, Claude, Gemini)을 동시에 orchestration 하는
로컬 전용 Multi-LLM 협업 시스템이다.

API 기반 호출은 허용하지 않는다.

---

# 2. 시스템 범위

## 포함 범위
- Chrome Extension 기반 UI
- 로그인 세션 탐색
- 다중 LLM 제어
- 라운드 orchestration
- 컨텍스트 전달
- 응답 저장
- Markdown export
- 히스토리 조회

## 제외 범위
- 모바일 지원
- 서버 저장
- 사용자 계정 시스템
- 암호화
- 멀티 유저
- API Key 연동

---

# 3. LLM 연결 요구사항

## 지원 대상
- ChatGPT
- Claude
- Gemini

## 연결 방식
- Chrome 로그인 세션 사용
- API 사용 금지
- 각 서비스 로그인 상태 자동 탐색

## 실패 처리
1. 로그인 유도
2. 사용 가능한 LLM 부재 시 안내 출력

---

# 4. LLM 선택 요구사항

## 선택 규칙
- 최소 2개 이상 선택 필수
- 단독 실행 금지

## 상태 유지
- 선택 상태 세션 유지

## 동적 변경
- 라운드 종료 후 변경 가능
- 라운드 진행 중 변경 금지

---

# 5. 라운드 제어 요구사항

## 기본 규칙
- 기본 3회
- 최대 9회

## 회차 구조

### Round 1
입력:
- 사용자 요청만 전달

제약:
- 타 LLM 응답 공유 금지

### Round 2
입력:
- 타 LLM의 Round1 응답 전체

목적:
- 상호 검토
- 피드백 생성

### Round 3+
입력:
- 이전 라운드 결과

목적:
- 수렴
- 보완
- 최종 답변 생성

---

# 6. 동기화 규칙

다음 라운드는:
- 모든 선택 LLM 응답 완료 후 시작

예외:
- timeout
- retry 초과
- 사용자 종료

---

# 7. 사용자 개입 요구사항

## 허용
- 조기 종료
- 회차 연장
- 특정 LLM 재질의
- 다음 요청 사전 입력

## 금지
- 응답 생성 중 개입
- 라운드 중 재질의
- 라운드 중 LLM 변경

---

# 8. Timeout 정책

## 기본값
- 5분

## 사용자 설정
- timeout 분 단위 설정
- retry 횟수 3~9회

## 전달 정보
각 request에는:
- 현재 회차
- 최대 회차
- timeout 정보
포함 필요

---

# 9. 저장 요구사항

## 자동 저장
- 라운드 완료 시 IndexedDB 저장

## Export
- 전체 Markdown
- LLM별 개별 Markdown

## 검색
검색 기준:
- 년월일시
- LLM
- Round

---

# 10. 응답 정책

## 출력 형식
- 구조화 강제

## 컨텍스트 제한
- response context 1500 token 제한
- 추가 설계 검토 필요

## 출력 스타일
- 장황한 출력 회피
- 핵심 중심 응답

---

# 11. 기술 요구사항

## 플랫폼
- Chrome 최신 안정버전

## Extension
- Manifest V3

## OS
- Windows
- macOS
- Linux

## 저장
- Local only

---

# 12. 개발 원칙

## 필수 원칙
- step by step
- no guess

## 품질 원칙
- 빠짐없이
- 틀림없이
- 다름없이
- 더함없이

## 산출물 원칙
- 재사용 가능한 설계 템플릿 자산화

