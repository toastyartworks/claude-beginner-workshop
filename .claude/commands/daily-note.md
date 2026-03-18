---
description: 오늘의 Daily Note 생성 또는 열기
allowed-tools: Read, Write, Bash, Glob, AskUserQuestion
---

오늘 날짜의 Daily Note를 만들어주세요.

**오늘 날짜**: $CURRENT_DATE

**수행할 작업:**

1. `30-personal/daily/` 폴더가 없으면 생성
2. `30-personal/daily/$CURRENT_DATE.md` 파일 확인
   - **이미 있으면**: 파일 내용을 읽어서 보여주기
   - **없으면**: 아래 템플릿으로 새 파일 생성

3. 파일 생성 후, 사용자에게 물어보기:
   - "오늘 할 일이 있어? 적어줄게."
   - 사용자가 답하면 `## 오늘 할 일` 섹션에 체크리스트로 추가

**템플릿:**

```markdown
# $CURRENT_DATE

## 오늘 할 일
- [ ]

## 메모


## 감사한 것

```

**출력 형식:**
- 새로 만들었으면: "Daily Note 생성 완료: 30-personal/daily/$CURRENT_DATE.md"
- 이미 있으면: "오늘 Daily Note가 이미 있어. 내용 보여줄게."
