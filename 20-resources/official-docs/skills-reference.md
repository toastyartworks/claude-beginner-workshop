# Claude Code 스킬(슬래시 커맨드) 레퍼런스

> Source: Claude Code 공식 문서 기반 정리 (2026-04-18 갱신)

## 파일 위치

| 우선순위 | 위치 | 범위 |
|---------|------|------|
| 1 (최상) | Enterprise (managed settings) | 조직 전체 |
| 2 | `~/.claude/skills/<name>/SKILL.md` (Personal) | 모든 프로젝트 |
| 3 | `.claude/skills/<name>/SKILL.md` (Project) | 현재 프로젝트 |
| 별도 | `<plugin>/skills/<name>/SKILL.md` (Plugin) | 네임스페이스됨 |

동일한 이름이 여러 레벨에 있으면 **상위 우선순위가 승**. 플러그인 스킬은 `plugin-name:skill-name` 네임스페이스를 사용하므로 충돌 불가.

**레거시** (여전히 동작): `.claude/commands/<name>.md`

### 레거시 commands/ vs 새 skills/ 차이

| | `commands/` (레거시) | `skills/` (현재) |
|---|---|---|
| 파일 구조 | 단일 `.md` 파일 | 디렉토리 (SKILL.md + 보조 파일) |
| 우선순위 | skills보다 낮음 | 우선 적용 |
| 번들 리소스 | 미지원 | 보조 파일 번들 가능 |
| 자동 호출 | 지원 | 지원 |
| 디렉토리 중첩 | 불가 | 가능 (하위 디렉토리) |

동일 이름의 `skills/`와 `commands/`가 있으면 **skills가 우선**.

### 모노레포 중첩 디렉토리 탐색

모노레포 환경에서 하위 디렉토리의 `.claude/skills/`도 자동 발견됨. 예:

```
monorepo/
├── .claude/skills/       # 루트 스킬
├── packages/
│   └── api/
│       └── .claude/skills/   # api 패키지 스킬 (자동 발견)
└── apps/
    └── web/
        └── .claude/skills/   # web 앱 스킬 (자동 발견)
```

### --add-dir 동작 (스킬은 예외)

`--add-dir`은 기본적으로 **파일 접근 권한만** 부여하고 설정은 로드하지 않지만, **스킬은 예외**. 추가된 디렉토리의 `.claude/skills/`는 자동 로드됨.

반면 서브에이전트, 커맨드, 아웃풋 스타일은 `--add-dir`에서 로드되지 않음.

추가된 디렉토리의 CLAUDE.md는 **기본적으로 로드되지 않음**. 환경변수 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`로 활성화 가능.

### 라이브 변경 감지

Claude Code는 스킬 디렉토리 변경을 감지. `~/.claude/skills/`, 프로젝트 `.claude/skills/`, `--add-dir` 내 `.claude/skills/` 에서 스킬을 **추가·수정·삭제하면 세션 재시작 없이 즉시 반영**됨.

**예외**: 세션 시작 시점에 존재하지 않던 새 top-level `skills/` 디렉토리는 재시작해야 watch 대상으로 등록됨.

---

## 3단계 로딩 모델

스킬은 3단계로 점진적 로드됨:

| 단계 | 로드 시점 | 내용 | 컨텍스트 영향 |
|------|----------|------|-------------|
| 1. 메타데이터 | **항상** (세션 시작) | name, description, frontmatter | description만 컨텍스트에 상주 |
| 2. SKILL.md | **호출 시** | 메인 지시사항 전체 | 호출된 시점에 로드 |
| 3. 번들 리소스 | **필요 시** | reference.md, scripts/ 등 | SKILL.md에서 참조할 때만 |

**핵심**: description은 항상 컨텍스트에 올라가므로, 자동 호출 판단의 유일한 기준이 됨.

---

## 스킬 콘텐츠 라이프사이클

호출된 스킬의 SKILL.md는 **세션 내내 컨텍스트에 상주**. 이후 턴마다 다시 읽지 않음.

- 한 번 호출되면 첫 응답 이후에도 계속 영향 유지
- 작업 전체에 적용되어야 할 지침은 "한 번만 하는 단계"가 아니라 **상시 지시사항**으로 작성할 것

### auto-compaction 동작

컨텍스트가 요약될 때, Claude Code는 호출됐던 스킬을 다음 규칙으로 re-attach:
- 각 스킬의 **최근 호출본**만 유지, **첫 5,000토큰**만 보존
- 전체 re-attach 예산 **25,000토큰** (가장 최근에 호출한 스킬부터 채움)
- 한 세션에 스킬을 너무 많이 호출하면 오래된 스킬은 compaction 이후 완전히 drop 가능

### 스킬이 효과 없어 보일 때

콘텐츠는 대부분 아직 존재. 모델이 다른 도구/접근을 선택한 것:
- description과 지침을 강화해서 모델이 계속 선호하도록
- deterministic 하게 강제하려면 [hooks](https://code.claude.com/docs/en/hooks) 사용
- 스킬이 크거나 이후 다른 스킬을 많이 호출했다면, compaction 후 **재호출**해서 전체 콘텐츠 복원

---

## 스킬 콘텐츠 타입

SKILL.md는 자유 형식이지만, **어떻게 호출될지**를 기준으로 내용을 잡으면 편함.

### Reference 콘텐츠

배경 지식을 현재 작업에 적용. 컨벤션, 패턴, 스타일 가이드, 도메인 지식.
- 인라인 실행 (대화 컨텍스트와 함께 사용)
- 보통 `user-invocable: false` — Claude만 호출

```yaml
---
name: api-conventions
description: 이 코드베이스의 API 설계 패턴
user-invocable: false
---

API 엔드포인트 작성 시:
- RESTful 명명 규칙 사용
- 일관된 에러 포맷 반환
- Request validation 포함
```

### Task 콘텐츠

특정 작업에 대한 단계별 지시사항. 배포, 커밋, 코드 생성 등.
- 보통 `/skill-name`으로 사용자가 직접 호출
- `disable-model-invocation: true`로 Claude 자동 호출 방지
- 무거운 작업은 `context: fork`로 격리

```yaml
---
name: deploy
description: 프로덕션 배포
context: fork
disable-model-invocation: true
---

1. 테스트 실행
2. 애플리케이션 빌드
3. 배포 대상으로 push
4. 배포 성공 확인
```

---

## Frontmatter 전체 필드

```yaml
---
name: my-command          # 선택. 소문자, 숫자, 하이픈만. 최대 64자. 생략 시 디렉토리명 사용
description: 설명          # Claude 자동 호출 판단 기준. 구체적으로 작성
when_to_use: |            # 선택. description 보완. 트리거 구문이나 예시 요청
  Use when the user says "deploy" or asks about deployment status.
argument-hint: [인자설명]   # 자동완성 시 힌트
allowed-tools: Read, Write, Bash(git *)  # 허용 도구 (공백 또는 줄바꿈 구분)
model: sonnet              # sonnet, opus, haiku, inherit, 또는 전체 모델 ID
effort: high               # low, medium, high, xhigh, max (모델별 가용 레벨 상이)
context: fork              # fork면 격리된 서브에이전트로 실행
agent: Explore             # context: fork일 때 서브에이전트 타입
disable-model-invocation: true   # Claude 자동 호출 차단 (사용자만 호출 가능)
user-invocable: false            # 사용자 메뉴에서 숨김 (Claude만 호출 가능)
paths:                           # 선택. 글로브 패턴 매칭 파일 작업 시에만 자동 호출
  - "src/**/*.{tsx,jsx}"
  - "components/**/*"
shell: bash                      # bash(기본) 또는 powershell. 인라인 !`cmd` 실행 쉘
hooks:                           # 스킬 스코프 lifecycle hooks
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
---
```

모든 필드는 선택. `description`만 권장 (Claude가 언제 스킬을 쓸지 판단하는 기준).

### when_to_use 필드

- `description`을 보완. 트리거 구문, 예시 요청 등 긴 컨텍스트 제공용
- **description + when_to_use 합계 1,536자로 잘림** (스킬 listing에서)
- 핵심 use case는 앞쪽에 배치 (뒤쪽이 truncate될 수 있음)

### paths 필드 — 조건부 활성화

- 글로브 패턴 매칭되는 파일 작업 시에만 Claude가 자동 호출
- 컨텍스트 절약: 비관련 파일 작업 중에는 스킬 로드 안 됨
- 문법: [path-specific rules](https://code.claude.com/docs/en/memory)와 동일

### effort 필드 — 모델 effort 오버라이드

- 스킬 실행 동안 세션의 effort level 오버라이드
- 옵션: `low`, `medium`, `high`, `xhigh`, `max` (모델별로 지원 레벨 다름)
- 환경변수 `CLAUDE_CODE_EFFORT_LEVEL`이 frontmatter보다 우선

### shell 필드

- `` !`command` `` 와 ` ```! ` 블록 실행 쉘 선택
- `bash` (기본) 또는 `powershell`
- PowerShell 사용 시 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` 필요

### description 작성 가이드

description은 자동 호출의 핵심. **"pushy description"** 원칙 — Claude가 해당 스킬을 써야 하는 상황을 적극적으로 명시.

**좋은 예시** (구체적, 적극적):
```yaml
description: >
  Use this skill to deploy to production. TRIGGER when: user says "deploy",
  "ship it", "push to prod", or asks about deployment status.
  DO NOT TRIGGER when: user is just discussing deployment concepts.
```

```yaml
description: >
  Code reviewer. Use after ANY code changes to review quality,
  security, and best practices. Always run before committing.
```

**나쁜 예시** (모호, 소극적):
```yaml
description: A helper for deployments    # 너무 모호
description: Reviews code                # 언제 쓸지 불분명
```

**작성 팁**:
- **TRIGGER when**: 어떤 상황에서 호출해야 하는지 명시
- **DO NOT TRIGGER when**: 비슷하지만 호출하면 안 되는 상황 구분
- 구체적인 키워드, 패턴, 조건을 나열

### description 캐릭터 예산

- **전체 예산**: 컨텍스트 윈도우의 **1%**, fallback 기본 **8,000자**
- **개별 스킬 캡**: `description` + `when_to_use` 합계 **1,536자**로 잘림 (전체 예산과 무관하게 항상 적용)
- 환경변수 `SLASH_COMMAND_TOOL_CHAR_BUDGET`로 전체 예산 상향 조정 가능
- 스킬이 많으면 개별 description 간결하게, **핵심 use case는 앞쪽 배치** (뒤쪽이 잘리면 Claude가 매칭 못할 수 있음)

### disable-model-invocation의 정확한 동작

`disable-model-invocation: true` 설정 시:
- description이 **컨텍스트에 올라가지 않음** (예산도 소모하지 않음)
- Claude가 해당 스킬의 존재 자체를 인식하지 못함
- 오직 사용자가 `/커맨드명`으로 직접 호출할 때만 동작

### hooks 필드 상세

스킬 단위로 lifecycle hooks를 정의할 수 있음. 해당 스킬이 실행되는 동안에만 활성화됨.

```yaml
hooks:
  PreToolUse:              # 도구 실행 전
    - matcher: "Bash"      # 특정 도구에만 적용
      hooks:
        - type: command
          command: "./scripts/validate.sh"
  PostToolUse:             # 도구 실행 후
    - matcher: "Write"
      hooks:
        - type: command
          command: "./scripts/lint.sh"
```

- 스킬 hooks는 글로벌 hooks와 병합됨 (스킬 hooks가 추가로 적용)
- `PreToolUse`, `PostToolUse` 등 동일한 hook 이벤트 지원

### 호출 제어 매트릭스

| 설정 | 사용자 호출 | Claude 호출 | description 컨텍스트 로드 |
|------|-----------|------------|------------------------|
| 기본값 (플래그 없음) | O | O | O |
| `disable-model-invocation: true` | O | X | X |
| `user-invocable: false` | X | O | O |

---

## 변수 (Variables)

| 변수 | 설명 | 예시 |
|------|------|------|
| `$ARGUMENTS` | 호출 시 전달된 모든 인자 | `/cmd foo bar` → `foo bar` |
| `$ARGUMENTS[N]` | N번째 인자 (0-based) | `$ARGUMENTS[0]` = 첫 번째 |
| `$N` | `$ARGUMENTS[N]` 축약 | `$0` = 첫 번째 |
| `$CURRENT_DATE` | 오늘 날짜 | `2026-03-20` |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID | 세션별 고유 파일 생성 시 |
| `${CLAUDE_SKILL_DIR}` | 스킬 파일이 위치한 디렉토리 | 번들 스크립트 참조 시 |

**참고**: `$ARGUMENTS`가 프롬프트에 없으면 자동으로 끝에 `ARGUMENTS: <입력값>` 추가됨

---

## allowed-tools 유효 도구 목록

### 기본 도구
`Read`, `Write`, `Edit`, `MultiEdit`, `Bash`, `Glob`, `Grep`, `WebFetch`, `WebSearch`

### 스코프 제한 (패턴)
- `Bash(git *)` — git 명령만 허용
- `Bash(npm *)` — npm 명령만 허용
- `Skill(deploy)` — 특정 스킬만 허용
- `Agent(worker, researcher)` — 특정 서브에이전트만 허용

### MCP 도구
설정된 MCP 서버의 도구도 사용 가능

---

## 빌트인 스킬 목록

Claude Code에 기본 내장된 스킬들:

| 스킬 | 설명 |
|------|------|
| `/batch` | 여러 파일/작업을 일괄 처리 |
| `/claude-api` | Claude API/Anthropic SDK 사용 코드 작성 지원 |
| `/debug` | 에러, 테스트 실패 등 디버깅 |
| `/loop` | 프롬프트나 슬래시 커맨드를 반복 실행 (예: `/loop 5m /foo`) |
| `/simplify` | 변경된 코드의 재사용성, 품질, 효율성 리뷰 후 개선 |

빌트인 스킬은 사용자 정의 스킬과 동일한 방식으로 호출 가능.

### Skill tool로 호출 가능한 빌트인 커맨드

일부 빌트인 커맨드는 Claude가 Skill tool을 통해 호출 가능:
- `/init` — CLAUDE.md 초기화
- `/review` — PR 리뷰
- `/security-review` — 보안 리뷰

그 외 빌트인 커맨드 (`/compact` 등)는 Skill tool로 호출 불가 (사용자만 입력).

---

## 고급 기능

### 동적 컨텍스트 주입 (`!` 명령)

실행 결과를 프롬프트에 주입 (전처리):

```markdown
## PR 컨텍스트
- Diff: !`gh pr diff`
- 변경 파일: !`gh pr diff --name-only`

위 내용을 요약해주세요.
```

멀티라인 명령은 ` ```! ` 로 여는 fenced code block 사용:

````markdown
```!
node --version
npm --version
git status --short
```
````

**전역 비활성화**: `settings.json`에 `"disableSkillShellExecution": true` 추가 시, 사용자/프로젝트/플러그인/추가 디렉토리 스킬의 인라인 쉘 실행이 차단됨 (각 명령은 `[shell command execution disabled by policy]`로 대체). 번들 스킬은 영향 없음. [managed settings](https://code.claude.com/docs/en/settings)에서 설정하면 사용자가 override 불가.

### Extended Thinking

프롬프트에 `ultrathink` 키워드를 포함하면 활성화.

### 격리 실행 (context: fork)

```yaml
---
context: fork
agent: Explore
---
```
- 독립 context window에서 실행
- 메인 대화 기록 접근 불가
- 결과만 요약되어 반환
- 명시적 작업 지시가 있어야 동작

---

## 스킬 디렉토리 구조

```
my-skill/
├── SKILL.md              # 필수 — 메인 지시사항
├── reference.md          # 선택 — 상세 문서
├── examples.md           # 선택 — 사용 예시
├── template.md           # 선택 — 템플릿
└── scripts/
    ├── helper.py         # 선택 — 유틸리티
    └── validate.sh
```

`SKILL.md`만 필수. 보조 파일은 `SKILL.md`에서 참조해야 Claude가 로드함.
SKILL.md는 500줄 이하 권장.

---

## 권한으로 스킬 제어

`disable-model-invocation` 외에 permissions으로 Claude의 스킬 호출을 제한 가능.

### 전체 차단

```text
# /permissions deny rules
Skill
```

### 특정 스킬 허용/차단

```text
# 허용
Skill(commit)
Skill(review-pr *)

# 차단
Skill(deploy *)
```

- `Skill(name)` — 정확한 이름 매칭
- `Skill(name *)` — 이름 prefix + 임의 인자 매칭

**참고**: `user-invocable: false`는 `/` 메뉴 표시만 제어하고 Skill tool 호출은 막지 않음. Claude 호출 자체를 막으려면 `disable-model-invocation: true` 사용.

---

## 트러블슈팅

### 스킬이 호출 안 됨

1. description에 사용자가 자연스럽게 말할 키워드가 포함됐는지 확인
2. `What skills are available?` 로 스킬이 리스트에 나오는지 확인
3. 요청 표현을 description에 더 가깝게 바꿔보기
4. user-invocable 스킬이면 `/skill-name`으로 직접 호출 시도

### 스킬이 너무 자주 호출됨

1. description을 더 구체적으로
2. 수동 호출만 원하면 `disable-model-invocation: true` 추가

### description이 잘림

스킬이 많으면 description이 캐릭터 예산에 맞춰 잘림 → Claude가 매칭할 키워드가 사라질 수 있음:
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` 환경변수로 예산 상향
- description과 when_to_use의 핵심 use case를 **앞쪽에 배치** (1,536자 개별 캡이 항상 적용되므로)

---

## 설계 팁

1. **description을 구체적으로** — "pushy description" 원칙. TRIGGER/DO NOT TRIGGER 패턴 사용
2. **핵심 use case는 앞쪽에** — 1,536자 캡에서 잘리면 Claude가 매칭 못 함
3. **allowed-tools는 최소 권한** — 필요한 도구만 명시
4. **`!` 명령으로 컨텍스트 주입** — Claude가 실행하는 게 아니라 전처리
5. **disable-model-invocation** — 위험한 작업(배포 등)에는 자동 호출 차단 (컨텍스트에서도 제거됨)
6. **context: fork** — 무거운 리서치 작업은 격리 실행
7. **paths로 조건부 활성화** — 특정 파일 패턴 작업 시에만 호출되는 스킬은 컨텍스트 절약
8. **hooks로 안전장치** — 스킬 스코프에서만 동작하는 검증 스크립트 연결
9. **Reference vs Task 구분** — 배경 지식은 `user-invocable: false`, 실행 작업은 `disable-model-invocation: true`

---

## 공식 문서

- [Skills](https://code.claude.com/docs/en/skills)
- [Commands reference](https://code.claude.com/docs/en/commands)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [Plugins](https://code.claude.com/docs/en/plugins)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Memory](https://code.claude.com/docs/en/memory)
