# 내가 원하는 것.
## 사용자와 여러 LLM간 다자 대화
- 크롬 브라우져 에 구글 계정으로 로그인 후 사용 가능 LLM 대상으로 진행
- 여러 LLM 목록을 보여주고, 사용자가 선택하여 전체 또는 부분 LLM 선택 가능.
- request/response cylcle 횟수를 제한하여 3회까지 진행가능(default) - 9회까지 변경 가능(1회씩, 채팅 중에도 변경 가능)
 - 완전 자유 대화는 아니며, 사용자는 chatting 진행 중 제한할 수 있음.
 - 채팅 중 선택한 LLM을 제외 처리 가능
 - 채팅 중 선택한 LLM을 추가 처리 가능
- 1차 고객이 request, LLM들이 Response 
    -  모든 response가 완료될 때까지 Request 불가.
- 2차 request 발생 시 1차 response를 LLM명과 response내용을 포함하여 전달
- 3차 request 발생 시 2차 response를 LLM명과 response내용을 포함하여 전달
- 지정한 회차까지 Response까지 전달받음.
- 1차 Request 전달할 때, 몇 회차까지 response를 받는지 LLM들에게 공유함.
- 전체 내용을 Md 로 저장하거나, LLM 별로 md 파일 저장하는 기능 필요.
- UI/UX 는 chat hub (크롬 확장 플러그인)을  참고함.
- 