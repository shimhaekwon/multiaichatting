# 220. 설계산출물 — ai-sarangbang (Design Artifacts)

> **제품 한 줄**: AI 여러 명과 사람(나)이 함께 들어가는 **라이브 턴제 채팅방** — 90년대 PC통신(하이텔·나우누리) 감성의 "사랑방". 한 명씩 순서대로 발언 + 끼어들기(바지인) + 귓속말.
>
> **버전**: v1(multichat) 대비 **신제품 VER-003** (`ai-sarangbang`). 구현 코드는 `E:\workspace\ai-sarangbang`. 설계 출처: `100.START/VER-003-ai-sarangbang`.
>
> 본 폴더는 **자기완결적** — 설계·구현에 `200.설계산출물-초안`/`210.설계산출물-multichat`를 볼 필요 없음(210은 carry-forward 대조·감사용 참고일 뿐).

## 우선순위 (문서 간 충돌 시)

> **규칙(핵심 4규칙)이 충돌하면 [221] §3 이 정본.** 그 외 항목은 각 문서가 자기 영역의 정본(coordinator=222, 데이터모델=223, 아키텍처/스택/단계=224).

## 읽는 순서

1. **`221-제품개요-및-핵심규칙.md`** — 제품 정의·범위(P0/P1/P2)·**핵심 규칙 4개**(턴제·floor 직렬화·큐 소진·귓속말)·코딩 지침 (먼저 읽기, 규칙 정본)
2. `222-coordinator-상태머신.md` — 제품의 **심장**. floor 루프 상태머신·race-free 의사코드·바지인 abort·whisper 채널·AgentDriver seam
3. `223-데이터모델-로그포맷.md` — 타입(Room·Participant·Message·Intent·Whisper)·저장 스키마·MD 포맷·whisper 휘발 불변식
4. `224-아키텍처-스택-단계.md` — 3레이어(UI/Coordinator/AgentDriver)·스택 결정(Vite+React+TS)·단계별 완료기준·폴더 구조·**carry-forward 매핑**·멀티유저 이식 경로
5. `225-UI-UX-설계.html` — 90s PC통신 감성 UI/UX(디자인 토큰·레이아웃·컴포넌트 매핑·상태 시각화·귓속말·접근성) + **인터랙티브 목업**(브라우저로 열면 동작). [221] O1(90s 감성 구체 스펙) 확정

## 문서 목록

| 문서 | 영역 | 핵심 | 상태 |
|------|------|------|------|
| `221-제품개요-및-핵심규칙.md` | 제품 개요·규칙 | 4규칙(R1~R4) 정밀 정의·범위·모호함 해소(D1~D5) | 정본(규칙 §3 최우선) |
| `222-coordinator-상태머신.md` | Coordinator | 단일 floor 루프·race-free·abort 가드·whisper·driver seam | 정본 |
| `223-데이터모델-로그포맷.md` | 데이터 모델 | 타입·저장 스키마·MD 포맷·whisper 미저장 | 정본 |
| `224-아키텍처-스택-단계.md` | 아키텍처·스택·단계 | 3레이어·Vite+React+TS·P0/P1/P2 완료기준·carry-forward·멀티유저 경로 | 정본 |
| `225-UI-UX-설계.html` | UI/UX | 90s 디자인 토큰·레이아웃·컴포넌트 매핑·상태 시각화·귓속말 UI·인터랙티브 목업 | 정본(O1 확정) |

## v1(multichat) 대비 — 무엇이 다른가

| 축 | v1 (multichat, VER-002) | 신제품 (ai-sarangbang, VER-003) |
|----|-------------------------|--------------------------------|
| 대화 방식 | 라운드제 **병렬**(전원 동시 응답) | 라이브 **턴제**(한 명씩 floor 직렬화) |
| AI 연결 | 브라우저 탭 DOM 스크래핑(무키) | **AgentDriver** 추상화(Mock→Ollama→API) |
| 형태 | 크롬 확장(MV3) | **로컬 웹앱**(확장 아님) |
| 공유 DNA | — | 멀티-AI 대화 + 귓속말(v1 비밀채팅 계승) |
| 구현 위치 | `E:\workspace\multichat` | **`E:\workspace\ai-sarangbang`** |

> **carry-forward**: v1의 데이터모델·context빌더·MD포맷·상태머신·whisper 불변식만 **순수 이식**. DOM·chrome·스크래핑·throttle 대응은 **제외**. 상세 매핑표 = [224] §5.

## 미해결 (폴더 전역)

- 미확정 결정(D1~D5) = [221] §4/§7, coordinator 잔여(C2~C5) = [222] §10, 아키텍처 잔여(A1~A4) = [224] 미해결, UI 잔여(U1~U5) = [225] §8 참조.
