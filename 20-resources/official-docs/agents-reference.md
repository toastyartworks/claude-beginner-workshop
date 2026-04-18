# Claude Code 서브에이전트 레퍼런스

> 공식 문서 기반 정리 (최신화: 2026-04-18)
> 출처:
> - [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
> - [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices)
> - [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)

---

## 서브에이전트란?

서브에이전트 = **별도 context window**, **커스텀 시스템 프롬프트**, **독립 권한**을 갖고 특정 작업만 전담하는 AI 보조. 메인 대화를 쏟아지는 검색 결과·로그·파일 내용으로 더럽히지 않고, 요약만 돌려받는 게 핵심.

**서브에이전트가 해결해주는 것:**
- **Context 보호** — 탐색/구현이 메인 대화에 쌓이지 않음
- **제약 강제** — 도구 접근을 제한해서 안전하게
- **재사용** — user-level로 저장하면 모든 프로젝트에서 사용
- **전문화** — 도메인별 시스템 프롬프트
- **비용 제어** — 간단한 작업은 haiku로 라우팅

> 💡 **Agent Teams는 별개 기능.** 서브에이전트는 단일 세션 내에서 작동. 여러 에이전트가 병렬로 상호 통신해야 하면 [Agent Teams](https://code.claude.com/docs/en/agent-teams) 참고.

---

## Built-in 서브에이전트

Claude Code에는 자동으로 쓰이는 기본 서브에이전트들이 있음:

| Agent | Model | 도구 | 용도 |
|-------|-------|------|------|
| **Explore** | Haiku | 읽기 전용 | 코드베이스 탐색·파일 검색. `quick` / `medium` / `very thorough` 강도 지정 |
| **Plan** | 상속 | 읽기 전용 | Plan Mode에서 컨텍스트 수집 (중첩 방지용) |
| **general-purpose** | 상속 | 전체 | 탐색+수정 둘 다 필요한 복잡한 작업 |
| **statusline-setup** | Sonnet | — | `/statusline` 실행 시 자동 호출 |
| **Claude Code Guide** | Haiku | — | Claude Code 기능 질문 답변 |

---

## 파일 위치와 우선순위

서브에이전트는 YAML frontmatter가 붙은 Markdown 파일. 같은 이름이면 우선순위 높은 쪽이 이김.

| 위치 | 범위 | 우선순위 |
|------|------|---------|
| Managed settings | 조직 전체 | 1 (최상) |
| `--agents` CLI flag | 현재 세션 | 2 |
| `.claude/agents/<name>.md` | 프로젝트 | 3 |
| `~/.claude/agents/<name>.md` | 내 모든 프로젝트 | 4 |
| Plugin `agents/` | 플러그인 활성 시 | 5 (최하) |

- **프로젝트 서브에이전트**: 코드베이스별. git에 커밋해서 팀 공유.
- **User 서브에이전트**: 내 모든 프로젝트에서 사용.
- **`--add-dir`로 추가한 디렉토리는 스캔 안 됨** (파일 접근만 부여).
- 수동으로 파일을 만들었다면 세션 재시작 or `/agents`로 다시 로드.

---

## Frontmatter 전체 필드

`name`과 `description`만 필수. 나머지는 모두 선택.

```yaml
---
name: code-reviewer                  # (필수) 소문자+하이픈
description: 코드 리뷰 전문가.          # (필수) Claude가 위임 판단하는 기준
tools: Read, Grep, Glob, Bash        # 허용 목록 (생략 시 전체 상속)
disallowedTools: Write, Edit         # 제외 목록 (inherited/specified 이후 적용)
model: sonnet                        # sonnet, opus, haiku, 전체 ID, inherit
permissionMode: default              # default, acceptEdits, auto, dontAsk, bypassPermissions, plan
maxTurns: 10                         # 에이전트 턴 상한
skills: [api-conventions]            # 시작 시 context에 주입할 스킬
mcpServers:                          # 이 에이전트 전용 MCP 서버
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
hooks:                               # 에이전트 활성 중에만 동작하는 lifecycle hooks
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
memory: project                      # user, project, local (크로스 세션 학습)
background: true                     # 백그라운드 실행 기본값
effort: medium                       # low, medium, high, xhigh, max
isolation: worktree                  # git worktree 격리 실행
color: blue                          # red/blue/green/yellow/purple/orange/pink/cyan
initialPrompt: "첫 질문 ..."          # --agent로 메인 세션 실행 시 자동 제출되는 첫 턴
---

여기부터 시스템 프롬프트 역할의 Markdown body.
서브에이전트는 이 프롬프트 + 기본 환경 정보만 받음 (메인 Claude Code 시스템 프롬프트는 없음).
```

---

## 모델 선택 우선순위

`model` 필드 값은 다음 순서로 결정됨:

1. `CLAUDE_CODE_SUBAGENT_MODEL` 환경변수
2. 호출 시점의 `model` 파라미터
3. 에이전트 frontmatter의 `model`
4. 메인 대화의 모델

값은 `sonnet`/`opus`/`haiku` 같은 alias, `claude-opus-4-7` 같은 full ID, 또는 `inherit` (기본값).

---

## 도구 제한

### 기본 — allowlist / denylist

```yaml
# allowlist: 이 목록만 허용, MCP 포함 나머지는 차단
tools: Read, Grep, Glob, Bash
```

```yaml
# denylist: 메인에서 상속한 전체 도구에서 Write/Edit만 제외
disallowedTools: Write, Edit
```

둘 다 쓰면 `disallowedTools` 먼저 적용 → 남은 풀에서 `tools`가 걸러짐. 양쪽에 모두 있으면 제거됨.

### Agent 툴 제한 (main thread 전용)

`claude --agent`로 메인 스레드에서 돌 때만 Agent 툴로 서브에이전트를 spawn할 수 있음. 이 경우 spawn 가능한 대상 제한 가능:

```yaml
tools: Agent(worker, researcher), Read, Bash   # worker, researcher만 spawn 허용
tools: Agent, Read, Bash                        # 전체 허용
# tools에 Agent 자체가 없으면 spawn 불가
```

> ⚠️ **서브에이전트는 다른 서브에이전트를 spawn할 수 없음.** 그래서 서브에이전트 정의에서 `Agent(...)`는 효과 없음. 중첩이 필요하면 Skills나 메인에서 체이닝.

### 특정 서브에이전트 금지

`settings.json`의 `permissions.deny`:
```json
{ "permissions": { "deny": ["Agent(Explore)", "Agent(my-custom-agent)"] } }
```

---

## Permission Mode

```yaml
permissionMode: acceptEdits
```

| 모드 | 동작 |
|------|------|
| `default` | 모든 도구 사용 시 확인 |
| `acceptEdits` | 파일 편집/일반 파일시스템 명령 자동 허용 |
| `auto` | 분류기 모델이 검토, 위험한 것만 차단 |
| `dontAsk` | 자동 거부 (명시 허용된 도구만 동작) |
| `bypassPermissions` | 권한 프롬프트 스킵 |
| `plan` | 읽기 전용 |

**상속 규칙**:
- 부모가 `bypassPermissions`나 `acceptEdits`면 자식이 override 불가 (부모 우선).
- 부모가 `auto`면 자식은 `permissionMode`를 무시하고 무조건 auto 상속.

**bypassPermissions 안전장치** (이것조차 차단 안 됨):
- `.git/`, `.claude/` (단 `.claude/commands`, `.claude/agents`, `.claude/skills`는 허용), `.vscode/`, `.idea/`, `.husky/` 쓰기는 여전히 프롬프트.

---

## MCP 서버 스코핑

이 에이전트 전용 MCP 서버를 붙일 수 있음. 메인 대화에는 해당 도구 설명이 주입 안 되니 context 절약에도 유용.

```yaml
mcpServers:
  # 인라인: 이 서브에이전트 전용 (시작 시 연결, 종료 시 해제)
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  # 이름 참조: 이미 설정된 서버 재사용
  - github
```

인라인 정의는 `.mcp.json`과 동일 스키마 (`stdio`, `http`, `sse`, `ws`).

---

## Skills 프리로드

`skills` 필드는 스킬 **전체 내용을 시작 시 context에 주입**함. 단순히 `/스킬명`으로 호출 가능하게 만드는 게 아님.

```yaml
---
name: api-developer
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the preloaded skills.
```

> 서브에이전트는 부모에서 스킬을 상속 안 받음. 쓰려면 여기 명시.

---

## Persistent Memory (크로스 세션 학습)

```yaml
memory: project
```

| scope | 경로 | 언제 |
|-------|------|------|
| `user` | `~/.claude/agent-memory/<name>/` | 모든 프로젝트에서 공유되는 지식 |
| `project` | `.claude/agent-memory/<name>/` | **추천 기본값**. 프로젝트별, git 커밋해서 팀 공유 |
| `local` | `.claude/agent-memory-local/<name>/` | 프로젝트별, git 비공유 |

활성화되면:
- 메모리 디렉토리 읽기/쓰기 지시가 시스템 프롬프트에 자동 포함
- `MEMORY.md`의 앞 200줄 or 25KB가 시스템 프롬프트에 포함 (초과 시 큐레이션 지시)
- Read/Write/Edit 도구 자동 활성화

**팁**: 에이전트 body에 "시작 전 메모리 확인, 끝나고 학습 저장" 지시를 넣으면 자연스럽게 지식이 쌓임.

---

## Hooks

### frontmatter에 정의 (서브에이전트 활성 중에만)

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
```

> frontmatter hook은 **Agent tool/@-멘션으로 서브에이전트 스폰될 때만** 동작. `--agent`로 메인 세션이 될 때는 동작 안 함. 세션 전역 hook은 `settings.json`에.

### settings.json에서 lifecycle 감지

| Event | 매처 | 시점 |
|-------|------|------|
| `SubagentStart` | 에이전트 이름 | 서브에이전트 시작 |
| `SubagentStop` | 에이전트 이름 | 서브에이전트 완료 |

> **Plugin subagent 제약**: `hooks`, `mcpServers`, `permissionMode` **사용 불가** (보안). 필요하면 파일을 `.claude/agents/`로 복사.

---

## 호출 방법

1. **자동 위임** — Claude가 `description` 보고 판단. "use proactively" 넣으면 적극 위임.
2. **자연어** — `"code-reviewer 서브에이전트로 최근 변경 리뷰해줘"` (Claude가 최종 판단)
3. **@-멘션** — `@"code-reviewer (agent)"` → 해당 에이전트 **확정 호출**
4. **CLI flag** — `claude --agent code-reviewer` → 세션 전체를 그 에이전트로
5. **프로젝트 기본값** — `.claude/settings.json`의 `"agent": "code-reviewer"`

`--agent` 모드일 때는 서브에이전트 시스템 프롬프트가 **Claude Code 기본 시스템 프롬프트를 완전히 대체**. CLAUDE.md는 여전히 로드됨.

---

## Foreground vs Background

| | Foreground | Background |
|---|---|---|
| 실행 | 메인 대화 블로킹 | 동시 실행 |
| 권한 프롬프트 | 그대로 전달 | **실행 전 미리** 확인, 나머지는 auto-deny |
| `AskUserQuestion` | 정상 동작 | 실패 (계속은 됨) |

- Claude가 작업 성격에 따라 자동 선택
- `"백그라운드로 돌려"` 요청 or **Ctrl+B**로 전환
- 백그라운드 실패 시 같은 작업을 foreground로 재시도
- 전체 비활성화: `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`

---

## Worktree 격리

```yaml
isolation: worktree
```

- 임시 git worktree에 격리된 레포 복사본에서 실행
- 변경 없이 종료하면 **자동 정리**
- 변경 있으면 worktree 경로와 브랜치명이 결과에 포함

---

## 주요 사용 패턴

### 1. 고출력 작업 격리

테스트, 문서 fetch, 로그 처리처럼 출력이 많은 작업을 서브에이전트에 맡기면 verbose output이 메인에 안 들어옴.

```
서브에이전트로 테스트 스위트 돌리고 실패한 테스트와 에러만 보고해줘
```

### 2. 병렬 리서치

```
authentication, database, API 모듈을 각각 다른 서브에이전트로 병렬 조사해줘
```

> 주의: 너무 많은 서브에이전트의 결과가 한꺼번에 메인으로 돌아오면 context 소모 큼.

### 3. 체이닝

```
code-reviewer로 성능 이슈 찾고, optimizer로 고쳐줘
```

### 4. Writer/Reviewer

세션 A에서 구현 → 세션 B (fresh context)에서 리뷰. 코드 쓴 자기 자신에게 편향 안 됨.

---

## 메인 대화 vs 서브에이전트 판단

**메인 대화를 써라:**
- 반복적인 back-and-forth 필요
- 여러 단계가 같은 context 공유 (planning → 구현 → 테스트)
- 빠르고 타겟팅된 작은 변경
- latency 중요 (서브에이전트는 context 모으느라 초기 지연)

**서브에이전트를 써라:**
- verbose 출력을 메인에 두고 싶지 않을 때
- 도구 제한/권한을 강제하고 싶을 때
- 작업이 self-contained하고 요약만 돌려받으면 됨

> 재사용 워크플로우지만 메인 context에서 돌아야 하면 **Skills**가 더 맞음. 대화 안 이미 있는 정보에 대한 간단 질문은 `/btw`.

---

## Resume와 트랜스크립트

### Resume 메커니즘

각 서브에이전트 호출은 새 인스턴스 = fresh context. 이전 작업을 **이어가려면**: Claude에게 "그 서브에이전트 작업 계속해줘" 요청 → Claude가 `SendMessage(to: agent-id, message: ...)` 사용.

- Resume된 에이전트는 이전 대화 기록/툴 콜/추론 **전부 유지**한 채로 이어감
- `SendMessage`는 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 설정 필요
- 중단된 에이전트가 `SendMessage` 받으면 자동으로 background에서 재개

### 트랜스크립트 저장

```
~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl
```

- 메인 대화 압축과 독립적으로 보존
- `cleanupPeriodDays` 설정 따라 자동 정리 (기본 30일)

### 자동 압축

기본 ~95% 시점에 트리거. `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`로 더 빨리 (예: 50).

---

## 제약 정리

- **중첩 불가**: 서브에이전트는 서브에이전트를 spawn 못함. 1단계만.
- **Plugin 제약**: `hooks`, `mcpServers`, `permissionMode` 미지원.
- **작업 디렉토리**: 메인의 cwd에서 시작. 내부 `cd`는 툴 콜 간 유지 안 되고 메인에도 영향 없음.

---

## 예시

### 코드 리뷰어 (읽기 전용)

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- No duplicated code
- Proper error handling
- No exposed secrets
- Input validation implemented
- Good test coverage

Provide feedback by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)
```

### 디버거 (수정 포함)

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works

For each issue, provide:
- Root cause explanation
- Evidence
- Specific fix
- Testing approach
- Prevention recommendations

Focus on fixing the underlying issue, not the symptoms.
```

### 데이터 사이언티스트 (도메인 전문)

```markdown
---
name: data-scientist
description: Data analysis expert for SQL queries, BigQuery operations, and data insights. Use proactively for data analysis tasks.
tools: Bash, Read, Write
model: sonnet
---

You are a data scientist specializing in SQL and BigQuery analysis.

When invoked:
1. Understand the requirement
2. Write efficient SQL
3. Use bq CLI when appropriate
4. Analyze and summarize
5. Present findings clearly
```

### 읽기 전용 DB 쿼리 (Hook 검증)

```markdown
---
name: db-reader
description: Execute read-only database queries for analysis.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---

You are a database analyst with read-only access. Execute SELECT queries only.
```

```bash
# scripts/validate-readonly-query.sh
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)\b' > /dev/null; then
  echo "Blocked: SELECT only" >&2
  exit 2
fi
exit 0
```

---

## 효과적인 에이전트 설계 원칙

_Building Effective Agents (Anthropic Engineering)_에서 정리한 원칙:

### 1. 단순함이 먼저

가장 단순한 해결책부터 시작, 필요할 때만 복잡도 추가. 단일 LLM 호출로 충분하면 multi-step 시스템 만들지 말 것.

### 2. Workflow vs Agent 구분

| | Workflow | Agent |
|---|---|---|
| 정의 | 미리 정의된 코드 경로가 LLM/툴을 오케스트레이션 | LLM이 자신의 흐름을 동적으로 지시 |
| 쓸 때 | 잘 정의되고 예측 가능한 작업 | 유연성·모델 기반 판단 필요 |

### 3. 다섯 가지 workflow 패턴

| 패턴 | 쓸 때 | 예시 |
|------|-------|------|
| **Prompt Chaining** | 고정된 순차 하위 작업 | 마케팅 카피 생성 → 번역 |
| **Routing** | 입력 분류 후 전문가에게 | 고객 문의 → 팀별 라우팅 |
| **Parallelization** | 독립 병렬 작업 or 투표 | 여러 인스턴스로 코드 리뷰 |
| **Orchestrator-Workers** | 예측 불가한 하위 작업 | 동적 태스크 분할 다중 파일 변경 |
| **Evaluator-Optimizer** | 반복 개선 | 문학 번역 + 피드백 루프 |

### 4. 에이전트 3대 원칙

1. **Simplicity** — 아키텍처는 단순하게
2. **Transparency** — 계획 단계와 추론을 명시적으로 드러내기
3. **Tool Documentation** — 에이전트-컴퓨터 인터페이스(ACI) 설계에 HCI 수준의 노력 투자

### 5. 툴 설계 베스트 프랙티스

- **포맷 선택**: 텍스트에 자연스럽게 나오는 포맷. string-escaping, line counting 같은 overhead 피하기
- **문서화**: 예시, 엣지 케이스, 포맷 요구사항, 툴 간 경계 명시
- **테스트**: 다양한 입력으로 workbench 테스트
- **Poka-yoke**: 실수를 어렵게 만드는 인자 설계 (예: 상대 경로 대신 절대 경로 강제)

> Anthropic SWE-bench 에이전트는 전체 프롬프트보다 **툴 최적화에 더 많은 시간**을 썼다고 함.

---

## Claude Code Best Practices 체크리스트

1. **description이 핵심** — Claude가 위임 판단하는 유일한 기준. "Use proactively..." 같은 능동 문구
2. **Focused subagent** — 하나의 전문 작업에만 탁월하게
3. **최소 권한 도구** — 읽기만 필요하면 Write/Edit 제외
4. **버전 관리** — 프로젝트 서브에이전트는 git에 커밋해서 팀과 공유
5. **maxTurns** — 무한 루프 방지
6. **model 선택** — 간단한 작업은 haiku, 보안 리뷰 같은 복잡한 건 opus
7. **isolation: worktree** — 파일 수정 작업은 격리 실행 고려
8. **background 활용** — 독립 작업은 백그라운드로 병렬 처리
9. **verification 포함** — 에이전트 body에 "작업 후 테스트 실행/스크린샷 비교" 같은 검증 지시
10. **Context 예산** — 여러 서브에이전트 결과가 한꺼번에 돌아오면 메인 context 폭증 주의
