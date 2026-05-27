# [핵심] LLM 다중 라운드 토론 시스템 상세 설계서 (v2.0)

전문 개발자가 아키텍처 수립 및 핵심 모듈을 구현할 수 있도록 상세화된 기술 요구사항 명세서입니다.

## 1. 아키텍처 및 기술 스택
* **Base Source:** `E:\\workspace\\chathub` 개선 및 확장
* **Platform:** Chrome Extension Manifest V3
* **Storage:** 브라우저 내장 IndexedDB (로컬 전용, 외부 서버 통신 없음)
* **Target OS:** Windows, macOS, Linux 크롬 데스크톱 환경

## 2. 핵심 모듈별 요구사항

### 2.1 LLM Session & Authentication Manager
* **탐색 매커니즘:** Claude, ChatGPT, Gemini 웹 서비스의 크롬 쿠키 및 세션 정보 자동 탐색 (API Key 미사용)
* **예외 처리 워크플로우:** 탐색 실패 시 -> 크롬 로그인 유도 UI 노출 -> 유효 계정 부재 시 default LLM 계정 생성 요청 가이드라인 안내
* **상태 유지:** 사용자가 선택한 LLM 목록(최소 2개 이상 필수)은 세션 간 영구 유지

### 2.2 Round Control Engine (토론 프로세스 제어)
* **라운드 범위:** 기본 3회 ~ 최대 9회 (사용자가 라운드 도중 1회씩 증감 가능, 마지막 회차에서도 연장 가능)
* **동기화 제어:** 선택된 모든 LLM의 Response가 완전히 수신(동기화)된 후 다음 라운드 진입 가능
* **라운드별 상태 머신 (State Machine):**
  * **Round 1:** Blind 모드. 각 LLM에 유저 원본 Prompt 투입 (타 LLM 응답 차단)
  * **Round 2:** 1라운드에 수신된 모든 LLM의 응답 텍스트 전체를 결합하여 각 LLM의 Context에 주입 후 상호 검토 유도
  * **Round 3 ~ N:** 최종 답변 유도 또는 사용자 연장에 따른 반복 제어
* **제어 명령:** 사용자 판단에 의한 조기 종료 및 회차 완료 후 연장 기능 구현

### 2.3 User Interaction & Interruption Policy
* **개입 차단:** Request 전송 후 모든 선택 LLM의 Response가 완료될 때까지 입력창 및 제어 UI 비활성화
* **라운드 간 개입 (Inter-round State):** 라운드와 라운드 사이 대기 상태에서만 아래 기능 허용
  * 참여 LLM 추가/제외
  * 특정 LLM 대상 1:1 재질의 (해당 결과는 다음 라운드 컨텍스트에 포함)
* **사전 입력 버퍼:** LLM이 연산 중인 상태에서 다음 회차에 전송할 request 미리 입력 가능 (단, 전송 버튼은 비활성화)
* **MVP 제외 항목:** 선응답 LLM과의 개별 '몰래 채팅' 기능은 MVP 이후로 배제 (구현 금지)

### 2.4 Timeout & Robustness Manager
* **타임아웃 제어:** 기본 5분, 분 단위로 사용자 커스텀 설정 가능
* **재시도 메커니즘:** LLM 응답 실패 시 3~9회 범위 내에서 사용자가 설정한 횟수만큼 재시도 프로세스 수행
* **프롬프트 메타데이터 주입:** 각 라운드 Request 시 프롬프트 내에 `[제한시간] + [현재차수/제한차수]` 정보를 메타데이터 형태로 강제 삽입하여 LLM에 전달

### 2.5 Data Persistence & Storage Subsystem
* **자동 저장:** 각 라운드가 완료될 때마다 IndexedDB에 즉시 트랜잭션 처리
* **검색 엔진:** 년월일시, 참여 LLM, 라운드 번호를 복합 인덱스로 하는 검색 기능 구현
* **File Export:** 토론 완료 후 전체 컨텍스트 통합 Markdown 파일 생성 기능 및 LLM별 독립적 Markdown 파일 개별 저장 기능 구현

### 2.6 Response Formatting & Guardrails
* **구조화 출력 강제:** LLM 프롬프트 인젝션을 통해 출력 형식을 강제 지정하고, 장황한 출력을 회피하도록 페이로드 설계
