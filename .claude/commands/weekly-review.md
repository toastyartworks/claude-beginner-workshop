---
description: 이번 주 Daily Note를 모아 주간 회고 작성
allowed-tools: Read, Write, Bash, Glob, AskUserQuestion
---

이번 주의 Daily Note들을 읽고 주간 회고를 작성해주세요.

**오늘 날짜**: $CURRENT_DATE

**수행할 작업:**

1. 이번 주 월요일~일요일 날짜 범위 계산
2. `30-personal/daily/` 폴더에서 해당 주의 Daily Note 파일들 찾기 (Glob 사용)
   - 파일이 하나도 없으면: "이번 주 Daily Note가 없어. /daily-note로 먼저 기록을 시작해볼까?" 라고 안내하고 종료
3. 찾은 파일들 전부 읽기
4. 내용을 분석해서 주간 회고 초안 작성

**주간 회고 템플릿:**

```markdown
# 주간 회고 — YYYY년 M월 N째 주 (MM-DD ~ MM-DD)

## 이번 주 요약
(Daily Note들에서 파악한 주요 활동 3~5줄 요약)

## 잘한 것
-

## 아쉬운 것
-

## 다음 주 포커스
-
```

5. 초안을 사용자에게 보여주고, 수정할 내용이 있는지 물어보기
6. `30-personal/weekly/` 폴더가 없으면 생성
7. `30-personal/weekly/YYYY-WXX.md` 형식으로 저장 (ISO 주차 사용)

**출력 형식:**
- "주간 회고 저장 완료: 30-personal/weekly/YYYY-WXX.md"
