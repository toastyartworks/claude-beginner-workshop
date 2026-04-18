# Claude Code 서브에이전트 레퍼런스

> Source: Claude Code 공식 문서 기반 정리 (2026-03-20)

## 파일 위치

| 위치 | 범위 |
|------|------|
| `.claude/agents/<name>.md` | 프로젝트 |
| `~/.claude/agents/<name>.md` | 모든 프로젝트 |

---

## Frontmatter 전체 필드

```yaml
---
name: code-reviewer
description: 코드 리뷰 전문가. 코드 변경 후 자동 실행.
tools: Read, Grep, Glob, Bash       # 사용 가능 도구 (생략 시 전체 상속)
disallowedTools: Write, Edit         # 제외할 도구
model: sonnet                        # sonnet, opus, haiku, inherit
permissionMode: default              # default, acceptEdits, dontAsk, bypassPermissions, plan
maxTurns: 10                         # 최대 에이전트 턴 수
skills: [deploy, review]             # 로드할 스킬 목록
memory: user                         # user, project, local (크로스 세션 학습)
background: true                     # 백그라운드 실행
effort: medium                       # low, medium, high, max
isolation: worktree                  # git worktree 격리
---

여기부터 시스템 프롬프트 역할의 Markdown body
```

---

## Agent Teams vs Subagents 구분

이 둘은 **완전히 다른 기능**:

| | Agent Teams | Subagents |
|---|---|---|
| 정의 | `.claude/agents/`에 정의한 커스텀 에이전트 | Claude가 내부적으로 spawning하는 작업 단위 |
| 호출 | `@에이전트명`, `--agent` flag | Claude가 Agent tool로 자동 생성 |
| 설정 | frontmatter로 세부 설정 | 호출 시 파라미터로 지정 |
| 목적 | 전문 역할 정의 (코드리뷰어, 디버거 등) | 병렬 작업 분배 |
| 영속성 | 파일로 정의, 재사용 가능 | 세션 내 일회성 |

---

## 스킬 vs 에이전트 차이

| | 스킬 (commands) | 에이전트 (agents) |
|---|---|---|
| 실행 방식 | 메인 대화에 주입 | 독립 context window |
| 호출 | `/커맨드명` (사용자) | Claude 자동 위임 또는 `@에이전트명` |
| 상태 | 메인 대화에 누적 | 완료 후 결과만 반환 |
| 병렬 | 불가 | 여러 에이전트 동시 실행 가능 |
| 용도 | 반복 프롬프트 템플릿 | 전문 역할 위임, 격리된 작업 |

---

## 에이전트 호출 방법

1. **자동 위임** — Claude가 description 기반으로 판단
2. **@-멘션** — `@"code-reviewer (agent)" 인증 모듈 살펴봐`
3. **CLI flag** — `claude --agent code-reviewer`
4. **설정 파일** — `.claude/settings.json`에 `"agent": "code-reviewer"`

---

## Plugin Subagent 제약

Plugin에서 제공하는 서브에이전트는 다음 기능을 **사용할 수 없음**:

- `hooks` — lifecycle hooks 미지원
- `mcpServers` — MCP 서버 스코핑 미지원
- `permissionMode` — 권한 모드 설정 미지원

Plugin subagent는 도구 접근과 기본 설정만 가능. 보안상 제한.

---

## Agent Tool 제약 Syntax

스킬의 `allowed-tools`에서 Agent tool을 제한할 때:

```yaml
allowed-tools: Agent(worker, researcher)   # worker, researcher만 허용
```

- 괄호 안에 허용할 에이전트 이름을 쉼표로 나열
- 지정하지 않은 에이전트는 호출 불가
- `Agent`만 쓰면 모든 에이전트 허용

---

## Subagent 중첩 제한

**서브에이전트는 서브에이전트를 spawn할 수 없음.** 1단계 깊이만 허용.

```
메인 Claude → 서브에이전트 A   ✅ 가능
메인 Claude → 서브에이전트 B   ✅ 가능
서브에이전트 A → 서브에이전트 C  ❌ 불가
```

---

## Resume 메커니즘 (v2.1.77+)

서브에이전트와의 통신 방식이 변경됨:

| | 이전 | v2.1.77+ |
|---|---|---|
| 방식 | `Agent resume` | `SendMessage` |
| 동작 | 에이전트를 다시 시작 | 기존 에이전트에 메시지 전송 |
| 컨텍스트 | 리셋됨 | **보존됨** (전체 컨텍스트 유지) |

```
# SendMessage로 기존 에이전트에 추가 지시
SendMessage(to: "agent-id", message: "추가 작업 해줘")
```

---

## Transcript 저장 위치

서브에이전트의 대화 기록은 다음 경로에 저장됨:

```
~/.claude/projects/{project-hash}/{sessionId}/subagents/
```

디버깅이나 감사 시 이 경로에서 서브에이전트의 전체 대화 기록을 확인할 수 있음.

---

## Background Subagent 동작

`background: true` 설정 시:

- 에이전트가 **백그라운드에서 비동기 실행**
- 필요한 권한을 **실행 전에 미리 확인** (백그라운드에서 권한 프롬프트 불가)
- 완료 시 자동으로 메인 대화에 알림
- foreground에서 재시작 가능 (백그라운드 결과를 이어받아 계속)
- `sleep`이나 polling 불필요 — 완료 시 알림이 옴

---

## Worktree 격리 상세

`isolation: worktree` 설정 시:

- 임시 git worktree에서 격리된 레포 복사본으로 작업
- **Cleanup 조건**: 에이전트가 변경 없이 종료하면 자동 정리
- 변경이 있으면 worktree 경로와 브랜치명이 결과에 포함
- `WorktreeCreate` / `WorktreeRemove` 훅이 트리거됨

```yaml
---
isolation: worktree
---
```

---

## Persistent Memory 저장 경로

`memory` 필드별 실제 저장 위치:

| memory 값 | 저장 경로 | 범위 |
|-----------|----------|------|
| `user` | `~/.claude/user-memory/` | 사용자 전체 (모든 프로젝트) |
| `project` | `~/.claude/projects/{project-hash}/memory/` | 프로젝트별 |
| `local` | `~/.claude/local/memory/` | 로컬 머신만 |

- `user`: 사용자 선호도, 코딩 스타일 등 범용 학습
- `project`: 프로젝트 고유 컨텍스트 (아키텍처, 컨벤션)
- `local`: git에 올리면 안 되는 로컬 환경 정보

---

## Permission Mode 상세

```yaml
permissionMode: bypassPermissions
```

| 모드 | 동작 |
|------|------|
| `default` | 모든 도구 사용 시 사용자 확인 |
| `acceptEdits` | 파일 편집은 자동 허용, 나머지는 확인 |
| `dontAsk` | 대부분 자동 허용, 위험한 작업만 확인 |
| `bypassPermissions` | 모든 권한 자동 허용 |
| `plan` | 읽기 전용. 파일 수정 불가 |

**bypassPermissions 보호 규칙**:
- `.git/` 디렉토리 직접 수정은 여전히 차단
- 시스템 파일 수정 차단
- 보안상 최소한의 안전장치는 유지됨

---

## 에이전트 예시

### 코드 리뷰어 (읽기 전용)

```markdown
---
name: code-reviewer
description: Expert code reviewer. Use after code changes.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code clarity and readability
- No duplicated code
- Proper error handling
- No exposed secrets
- Input validation
- Test coverage
```

### 디버거 (수정 포함)

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works
```

### 읽기 전용 DB 쿼리 (Hook 포함)

```markdown
---
name: db-reader
description: Execute read-only database queries for analysis and reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access...
```

---

## 주요 기능

### Persistent Memory (크로스 세션 학습)
```yaml
memory: user    # user / project / local
```

### 권한 모드
```yaml
permissionMode: bypassPermissions   # 권한 프롬프트 스킵
```

### MCP 서버 스코핑
```yaml
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
```

### 도구 제한
```yaml
tools: Read, Grep, Glob, Bash        # 허용 목록
# 또는
disallowedTools: Write, Edit          # 제외 목록
```

---

## 설계 팁

1. **description이 핵심** — Claude가 자동 위임 판단하는 유일한 기준. "pushy description" 원칙 적용
2. **tools는 최소 권한** — 읽기만 필요하면 Write/Edit 제외
3. **maxTurns 설정** — 무한 루프 방지
4. **model 선택** — 간단한 작업은 haiku/sonnet, 복잡한 건 opus
5. **isolation: worktree** — 파일 수정이 있는 작업은 격리 실행 고려
6. **서브에이전트는 중첩 불가** — 1단계 깊이만 허용, 설계 시 고려
7. **background 활용** — 독립적인 작업은 백그라운드로 돌려 병렬 처리
