아래 제공하는 유튜브 대본(스크립트) 내용을 기반으로, PPT 슬라이드 덱을 제작해 줘.

[발표 및 스크립트 정보]
- 제목: Claude Code, 도대체 어떻게 작동하는 걸까?
- 러닝타임 / 타겟: ~7분 (개발자 및 AI 도구에 관심 있는 분)
- 슬라이드 구성: 총 14개 슬라이드 (30초 단위 핵심 전환)
- 전반적인 Tone & Style: 개발자 대상의 Modern, Tech-Editorial, 깔끔하고 가독성 높은 다크/스틸 톤

[전체 대본 내용]
Title: Claude Code, 도대체 어떻게 작동하는 걸까?
Duration: ~7분 (~1,000 words)
Style: Casual-Friendly, Educational
Target: 개발자 / AI 도구에 관심 있는 분
Slides: 14개

---

[HOOK - 0:00~0:10]
[슬라이드 1: 터미널 화면 + Claude Code 로고]
"터미널에 명령 하나 치면, AI가 알아서 코드를 읽고, 수정하고, 테스트까지 돌려준다면? 이게 바로 Claude Code입니다."

---

[INTRO - 0:10~0:40]
[슬라이드 2: 타이틀 - "Claude Code 작동 원리"]
"안녕하세요! 오늘은 Claude Code가 내부적으로 어떻게 작동하는지, 핵심만 쏙쏙 뽑아서 쉽게 설명드릴게요. 어렵지 않습니다. 딱 3가지만 기억하시면 됩니다."

---

[MAIN 1: Agentic Loop & Harness - 0:40~2:30]
[슬라이드 3: AI 모델과 Harness의 개념]
"첫 번째는 Agentic Loop와 Harness 구조입니다. Claude AI 모델은 추론(Thinking)을 담당하고, Harness는 터미널/파일 읽기 등의 실행(Action)을 담당하는 구조입니다."

[슬라이드 4: Agentic Loop 3단계 순환구조]
"이 동작은 [1. 모델 추론 -> 2. Tool 실행 -> 3. 결과 피드백] 3단계 루프를 계속 반복하면서 사용자의 목표를 달성해 나갑니다."

[슬라이드 5: 주요 내장 도구 (Tools)]
"Harness가 제공하는 핵심 도구에는 Bash 실행, Read/Edit 파일 수정, Web Search, MCP 연결 등이 있습니다."

---

[MAIN 2: Context Management & Memory - 2:30~4:30]
[슬라이드 6: 컨텍스트와 토큰 누적]
"두 번째는 Context 관리입니다. 읽은 파일, 실행 결과, 대화 기록이 모두 컨텍스트에 쌓여 AI의 기억 공간을 구성합니다."

[슬라이드 7: 컨텍스트 압축 (Compaction Pipeline)]
"컨텍스트가 커져 한계에 다다르면 5단계 Compaction 파이프라인이 작동해 중요 정보만 요약하여 맥락을 유지합니다."

[슬라이드 8: 영속적 메모리 (CLAUDE.md)]
"세션이 끝나도 지침을 유지하기 위해 프로젝트 루트의 CLAUDE.md와 Auto Memory를 활용해 투명한 기억장치를 만듭니다."

---

[MAIN 3: Safety & Permission Framework - 4:30~6:00]
[슬라이드 9: 7가지 권한 모드]
"세 번째는 안전/권한 프레임워크입니다. plan, default, bypassPermissions 등 7가지 모드로 에이전트의 자율 동작 범위를 통제합니다."

[슬라이드 10: YOLO Classifier를 통한 피로도 감소]
"매번 승인 팝업이 뜨면 피곤하겠죠? YOLO ML 분류기가 안전한 작업을 예측해 인간의 확인 피로를 최소화합니다."

[슬라이드 11: 격리된 실행 환경 (Git Worktree)]
"코드베이스 오염을 막기 위해 백그라운드에서 임시 Git Worktree 사이드체인을 생성해 격리된 공간에서 검증을 완료합니다."

---

[CONCLUSION - 6:00~7:00]
[슬라이드 12: 핵심 요약 (3가지 축)]
"정리하자면, 1) Agentic Loop 실행, 2) 효율적 Context/Memory 관리, 3) 촘촘한 Safety/Permission이 Claude Code의 핵심입니다."

[슬라이드 13: 미래 코딩 에이전트의 방향성]
"복잡한 추론 체인보다 에이전트가 안전하고 자유롭게 움직일 수 있는 인프라 Harness OS 설계가 앞으로의 핵심입니다."

[슬라이드 14: Q&A 및 종결]
"감사합니다! Q&A 시간입니다."

---

위 대본의 핵심 주제와 14개 슬라이드 구성을 반영해서, 시각적으로 뛰어난 프레젠테이션 슬라이드를 생성해 줘!