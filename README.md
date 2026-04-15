# my-claude-skills

개인용 Claude Code 스킬 저장소. 여러 프로젝트·여러 컴퓨터에서 같은 스킬을 심볼릭 링크 기반으로 공유한다. 외부 CLI 의존 없이 `npx` 로 직접 실행되는 얇은 Node.js 스크립트(`bin/my-skills.mjs`) 하나로 설치/업데이트를 관리한다.

## 동작 원리

1. `npx github:jamin12/my-claude-skills install ...` 실행
2. `bin/my-skills.mjs` 가 `~/.my-claude-skills` 에 repo 를 clone (또는 기존 clone 을 `git pull`)
3. `~/.my-claude-skills/skills/<name>` → `~/.claude/skills/<name>` (또는 `./.claude/skills/<name>`) 심볼릭 링크 생성
4. 이후 스킬 수정은 `~/.my-claude-skills` 에서만 하고 `git push`
5. 다른 컴퓨터에서는 `npx github:jamin12/my-claude-skills update` 한 줄로 모든 설치가 즉시 최신 상태

Node.js 외 다른 런타임·CLI 가 필요하지 않다. 심볼릭 링크라서 `update` 후 재설치도 필요 없다.

## 설치

### 새 컴퓨터 세팅 (글로벌)

```bash
# 전부 설치
npx -y github:jamin12/my-claude-skills install

# 일부만 설치
npx -y github:jamin12/my-claude-skills install spec-setup cmux-worktree
```

`~/.claude/skills/` 아래에 심볼릭 링크가 생성된다. 원본은 `~/.my-claude-skills/skills/` 에 있다.

### Kotlin/Spring 프로젝트에 설치

프로젝트 루트에서 `--project` 플래그로 로컬 설치:

```bash
cd <your-project>
npx -y github:jamin12/my-claude-skills install --project \
  domain-model jpa-entity rest-controller usecase \
  setup-gradle mapper testing architecture code-convention ai-memory-plan
```

`./.claude/skills/` 아래에 심볼릭 링크가 생성된다.

## 명령어

```bash
# 업데이트 (모든 컴퓨터 공통)
npx -y github:jamin12/my-claude-skills update

# 설치된 스킬 확인
npx -y github:jamin12/my-claude-skills list
npx -y github:jamin12/my-claude-skills list --project

# 제거
npx -y github:jamin12/my-claude-skills remove spec-setup
npx -y github:jamin12/my-claude-skills remove --project usecase
```

`update` 는 `~/.my-claude-skills` 를 `git pull` 한 번 하는 것. 심볼릭 링크가 새 파일 내용을 자동으로 가리킨다.

## 스킬 목록

### 범용 (어떤 프로젝트에도 적용)

| 스킬 | 설명 |
|------|------|
| `spec-setup` | CLAUDE.md + `docs/specs/` 기반 맥락 문서 체계를 새 프로젝트에 이식 |
| `cmux-worktree` | git worktree 생성/삭제 + cmux(iTerm2) 새 탭 + Claude 세션 마이그레이션 |

### Kotlin / Spring Boot / 헥사고날 아키텍처 전용

| 스킬 | 설명 |
|------|------|
| `architecture` | 헥사고날(Ports & Adapters) 레이어 구조와 의존성 방향 |
| `domain-model` | 도메인 모델·VO 작성 규칙 (private constructor, factory, 불변성) |
| `usecase` | UseCase 패턴 (OutPort 의존, Result DTO, Command/Spec/Query/Result 파일 분리) |
| `rest-controller` | REST 컨트롤러·Request/Response DTO·매퍼 규칙 |
| `jpa-entity` | JPA Entity / Repository / Persistence Adapter 규칙 |
| `mapper` | MapStruct vs 확장함수, REST/JPA/Application 매퍼 패턴 |
| `testing` | Kotest BehaviorSpec, OutPort 모킹 전략 |
| `setup-gradle` | Gradle Convention Plugin 빌드 구조 |
| `code-convention` | 필드 네이밍(자기 도메인 vs 외부 도메인), Boolean 접두사 금지, ktlint |
| `ai-memory-plan` | `docs/plans/` 계획서·맥락노트·체크리스트 3종 문서 체계 |

## 규칙

- **수정은 `~/.my-claude-skills` 에서만 한다.** 설치된 각 프로젝트의 `.claude/skills/` 는 심볼릭 링크이므로, 그 경로에서 편집하면 원본도 바뀐다 (편의성). 단 편집 후 반드시 `~/.my-claude-skills` 에서 `git add/commit/push` 해야 다른 컴퓨터에 전파된다.
- **새 스킬 추가** 시 `skills/<name>/SKILL.md` 구조를 따르고 이 README 의 스킬 목록에도 한 줄 추가한다.
- **스킬 작성 포맷**은 `skills/code-convention` 같은 기존 스킬을 참고한다. YAML frontmatter 에 `name` + `description` 필드가 있어야 한다.

## 제약

- Node.js 18 이상 필요 (npx 포함)
- Windows 의 심볼릭 링크 생성은 관리자 권한 또는 Developer Mode 필요
