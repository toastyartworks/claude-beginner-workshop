---
description: 오늘 하루 회고 작성
allowed-tools: Read, Edit, Write, Bash, Glob, AskUserQuestion
---

오늘의 Daily Note를 읽고 하루 회고를 도와주세요.

**오늘 날짜**: $CURRENT_DATE

**수행할 작업:**

1. `30-personal/daily/$CURRENT_DATE.md` 파일 읽기
   - 없으면: "오늘 Daily Note가 없어. /daily-note로 먼저 만들어볼까?" 라고 안내하고 종료

2. Daily Note 내용을 보여주고, 사용자에게 질문하기:
   - "오늘 뭘 했어?"
   - 사용자 답변을 듣고: "내일 이어갈 게 있어?"

3. 사용자 답변을 바탕으로 Daily Note에 회고 섹션 추가:

```markdown

## 회고
### 오늘 한 것
- (사용자 답변 정리)

### 내일 이어갈 것
- (사용자 답변 정리)
```

4. 저장 후 확인: "회고 추가 완료."

**주의사항:**
- 기존 Daily Note 내용은 절대 삭제하지 않기
- 회고 섹션이 이미 있으면 덮어쓰지 말고, 내용을 업데이트할지 물어보기
