# Claude Code 고급 기능 가이드

> MCP, Skills, Subagents, Commands, Hooks를 활용한 실전 워크플로우

---

## 목차

- [개요: Claude Code 아키텍처](#개요-claude-code-아키텍처)
- [1. MCP (Model Context Protocol)](#1-mcp-model-context-protocol)
- [2. Skills](#2-skills)
- [3. Subagents](#3-subagents)
- [4. Commands (슬래시 커맨드)](#4-commands-슬래시-커맨드)
- [5. Hooks](#5-hooks)
- [6. 실전 워크플로우 예시](#6-실전-워크플로우-예시)

---

## 개요: Claude Code 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│  │Commands │  │ Skills  │  │Subagents│  │  Hooks  │            │
│  │ /plan   │  │playwright│ │ oracle  │  │on-commit│            │
│  │ /review │  │git-master│ │ explore │  │on-save  │            │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘            │
├─────────────────────────────────────────────────────────────────┤
│                    MCP (Model Context Protocol)                  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐        │
│  │ Bash │ │ Read │ │ Edit │ │ Grep │ │ Web  │ │Custom│        │
│  │      │ │ Write│ │      │ │ Glob │ │Fetch │ │Tools │        │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘        │
└─────────────────────────────────────────────────────────────────┘
```

**핵심 개념**:
- **MCP**: Claude가 외부 도구와 통신하는 표준 프로토콜
- **Skills**: 특정 작업에 특화된 지침 세트
- **Subagents**: 전문화된 하위 AI 에이전트들
- **Commands**: 빠른 작업 실행을 위한 슬래시 명령어
- **Hooks**: 특정 이벤트에 자동 실행되는 트리거

---

## 1. MCP (Model Context Protocol)

### 1.1 MCP란?

MCP는 AI가 외부 도구/서비스와 통신하는 **표준화된 프로토콜**입니다.

```
┌─────────┐         MCP          ┌─────────────┐
│ Claude  │ ◄──────────────────► │ MCP Server  │
│         │   JSON-RPC 통신       │ (도구 제공)  │
└─────────┘                      └─────────────┘
```

### 1.2 기본 제공 MCP 도구들

| 도구 | 용도 | 예시 |
|------|------|------|
| `bash` | 터미널 명령 실행 | `npm install`, `git status` |
| `read` | 파일 읽기 | 코드 파일, 설정 파일 |
| `write` | 파일 생성/덮어쓰기 | 새 파일 작성 |
| `edit` | 파일 부분 수정 | 특정 라인 수정 |
| `glob` | 파일 패턴 검색 | `**/*.ts` 파일 찾기 |
| `grep` | 내용 검색 | 코드에서 패턴 찾기 |
| `webfetch` | 웹 페이지 가져오기 | 문서, API 응답 |
| `todowrite` | 작업 목록 관리 | 진행 상황 추적 |

### 1.3 실사용 예시

#### 파일 검색 및 수정 흐름

```
사용자: "인증 관련 코드 찾아서 JWT 만료 시간을 1시간으로 바꿔줘"

Claude 내부 동작:
1. grep("JWT|token|auth", include="*.ts")  → 관련 파일 찾기
2. read("/src/auth/jwt.config.ts")         → 파일 내용 확인
3. edit(oldString="expiresIn: '24h'", newString="expiresIn: '1h'")  → 수정
4. bash("npm run test:auth")               → 테스트 실행
```

#### 프로젝트 구조 파악

```
사용자: "이 프로젝트 구조 설명해줘"

Claude 내부 동작:
1. glob("**/*", path="./")                 → 전체 파일 목록
2. read("package.json")                    → 의존성 확인
3. read("tsconfig.json")                   → 설정 확인
4. read("src/index.ts")                    → 엔트리포인트 확인
```

### 1.4 커스텀 MCP 서버 연결

**설정 파일** (`~/.claude/mcp_servers.json`):

```json
{
  "mcpServers": {
    "database": {
      "command": "node",
      "args": ["/path/to/db-mcp-server.js"],
      "env": {
        "DB_CONNECTION_STRING": "postgresql://..."
      }
    },
    "slack": {
      "command": "npx",
      "args": ["@anthropic/mcp-server-slack"],
      "env": {
        "SLACK_TOKEN": "xoxb-..."
      }
    },
    "github": {
      "command": "npx",
      "args": ["@anthropic/mcp-server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    }
  }
}
```

**활용 예시**:

```
사용자: "최근 이슈 중에 버그 관련된 거 찾아줘"

Claude (GitHub MCP 사용):
→ github.list_issues(labels=["bug"], state="open")
→ 결과 요약해서 전달
```

```
사용자: "이 변경사항 #frontend 채널에 공유해줘"

Claude (Slack MCP 사용):
→ slack.post_message(channel="#frontend", text="...")
```

### 1.5 MCP 서버 직접 만들기

**간단한 MCP 서버 예시** (Node.js):

```typescript
// my-mcp-server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "my-tools",
  version: "1.0.0"
}, {
  capabilities: { tools: {} }
});

// 도구 정의
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "get_user_info",
    description: "사내 시스템에서 사용자 정보 조회",
    inputSchema: {
      type: "object",
      properties: {
        userId: { type: "string", description: "사용자 ID" }
      },
      required: ["userId"]
    }
  }]
}));

// 도구 실행
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "get_user_info") {
    const { userId } = request.params.arguments;
    // 실제 로직...
    return { content: [{ type: "text", text: JSON.stringify(userInfo) }] };
  }
});

// 서버 시작
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 2. Skills

### 2.1 Skills란?

Skills는 **특정 작업에 특화된 지침 세트**입니다. 
해당 작업이 감지되면 자동으로 로드되어 Claude의 동작을 가이드합니다.

```
일반 Claude:     범용적인 응답
Skill 로드 후:   해당 분야 전문가처럼 동작
```

### 2.2 기본 제공 Skills

| Skill | 트리거 | 용도 |
|-------|--------|------|
| `playwright` | 브라우저 관련 작업 | 웹 자동화, 스크린샷, 테스트 |
| `git-master` | git 관련 작업 | 커밋, rebase, 히스토리 검색 |
| `frontend-ui-ux` | UI/UX 작업 | 컴포넌트 디자인, 스타일링 |
| `mermaid-tools` | 다이어그램 작업 | Mermaid 다이어그램 생성 |

### 2.3 실사용 예시

#### Playwright Skill

```
사용자: "로그인 페이지 스크린샷 찍어줘"

[Playwright Skill 자동 로드]

Claude:
1. 브라우저 실행
2. 로그인 페이지 접속
3. 스크린샷 캡처
4. 결과 전달
```

```
사용자: "회원가입 플로우 E2E 테스트 만들어줘"

[Playwright Skill 자동 로드]

Claude:
1. Playwright 테스트 구조 생성
2. 각 단계별 테스트 코드 작성
3. assertion 추가
4. 실행 및 결과 확인
```

#### Git-Master Skill

```
사용자: "이 버그 누가 만들었어? 언제 추가된 거야?"

[Git-Master Skill 자동 로드]

Claude:
1. git blame 실행
2. git log -S로 변경 이력 추적
3. 관련 커밋 찾기
4. 작성자와 날짜 정보 전달
```

```
사용자: "최근 5개 커밋 squash 해줘"

[Git-Master Skill 자동 로드]

Claude:
1. git log로 커밋 확인
2. git rebase -i 실행
3. squash 수행
4. 결과 확인
```

### 2.4 커스텀 Skill 만들기

**Skill 파일 위치**: `~/.claude/skills/my-skill.md`

```markdown
# My Custom Skill

## Trigger
이 skill은 다음 상황에서 활성화됩니다:
- 사용자가 "결제" 또는 "payment" 언급
- 결제 관련 파일 작업 시

## Context
우리 결제 시스템 특징:
- PG사: Stripe 사용
- 통화: KRW, USD 지원
- 결제 방식: 카드, 계좌이체

## Rules
1. 결제 금액은 항상 정수로 처리 (소수점 ❌)
2. 모든 결제 로직에 idempotency key 필수
3. 실패 시 반드시 재시도 로직 포함

## Patterns
결제 처리 시 다음 패턴 따르기:
\`\`\`typescript
// 결제 요청
const payment = await stripe.paymentIntents.create({
  amount: amountInCents,
  currency: 'krw',
  idempotencyKey: generateIdempotencyKey(),
});
\`\`\`

## Don'ts
- 절대 카드 번호를 로그에 남기지 않음
- PCI DSS 규정 위반 코드 생성 금지
```

---

## 3. Subagents

### 3.1 Subagents란?

Subagents는 **특정 역할에 전문화된 하위 AI 에이전트**입니다.
메인 Claude가 작업을 위임하고 결과를 받아옵니다.

```
┌─────────────────────────────────────┐
│          Main Claude                │
│  ┌─────┐  ┌─────┐  ┌─────┐        │
│  │Task1│  │Task2│  │Task3│        │
│  └──┬──┘  └──┬──┘  └──┬──┘        │
└─────┼────────┼────────┼───────────┘
      ▼        ▼        ▼
   Oracle   Explore  Librarian
  (분석)    (검색)    (문서)
```

### 3.2 주요 Subagents

| Agent | 역할 | 사용 시점 | 비용 |
|-------|------|----------|------|
| **Oracle** | 아키텍처, 디버깅 전문가 | 복잡한 문제, 설계 결정 | 높음 |
| **Explore** | 코드베이스 검색 | 파일/패턴 찾기 | 낮음 |
| **Librarian** | 외부 문서/레퍼런스 검색 | 라이브러리 사용법 | 낮음 |
| **Frontend-Engineer** | UI/UX 전문가 | 컴포넌트 개발 | 중간 |
| **Document-Writer** | 문서 작성 전문가 | README, API docs | 낮음 |

### 3.3 실사용 예시

#### Oracle - 복잡한 디버깅

```
사용자: "이 에러가 3일째 안 잡혀... 도와줘"

Claude: "Oracle에게 상담 요청합니다"

→ sisyphus_task(agent="oracle", prompt="
   에러 상황: [설명]
   시도한 것: [목록]
   코드: [관련 코드]
   
   근본 원인을 분석하고 해결책을 제시해주세요.
")

Oracle 응답:
"이 문제의 근본 원인은 race condition입니다.
 
 분석:
 1. 라인 42에서 비동기 호출이...
 2. 라인 58에서 상태 업데이트가...
 
 해결책:
 1. mutex 도입
 2. 또는 상태 관리를 이렇게 변경..."
```

#### Explore - 코드베이스 검색

```
사용자: "인증 관련 코드 다 찾아줘"

Claude: 
→ sisyphus_task(
    agent="explore", 
    prompt="Find all authentication-related code: 
            - Login/logout handlers
            - JWT/session management
            - Auth middleware
            - Permission checks",
    run_in_background=true
  )

[백그라운드에서 검색 진행]
[다른 작업 계속 가능]

Explore 응답:
"인증 관련 파일 목록:
 - src/auth/login.ts (로그인 핸들러)
 - src/middleware/auth.ts (인증 미들웨어)
 - src/utils/jwt.ts (JWT 유틸)
 ..."
```

#### Librarian - 외부 문서 검색

```
사용자: "Prisma에서 soft delete 어떻게 해?"

Claude:
→ sisyphus_task(
    agent="librarian",
    prompt="Find Prisma soft delete implementation:
            - Official docs
            - Best practices
            - Middleware approach",
    run_in_background=true
  )

Librarian 응답:
"Prisma soft delete 구현 방법:

 공식 문서에 따르면:
 1. Middleware 사용 방식 (권장)
 2. $extends 사용 방식 (Prisma 4.16+)
 
 예시 코드:
 \`\`\`typescript
 prisma.$use(async (params, next) => {
   if (params.action === 'delete') {
     params.action = 'update';
     params.args.data = { deletedAt: new Date() };
   }
   return next(params);
 });
 \`\`\`"
```

#### 병렬 Subagent 실행

```
사용자: "이 기능 구현하기 전에 조사 좀 해줘"

Claude:
→ [병렬 실행]
   sisyphus_task(agent="explore", prompt="기존 유사 기능 찾기...")
   sisyphus_task(agent="librarian", prompt="관련 라이브러리 문서...")
   sisyphus_task(agent="oracle", prompt="아키텍처 검토...")

[3개 동시 실행, 결과 종합]

Claude: "조사 결과를 종합하면:
 1. 기존 코드에서 참고할 부분: ...
 2. 라이브러리 권장 방식: ...
 3. 아키텍처 관점 고려사항: ..."
```

### 3.4 Subagent 호출 패턴

```typescript
// 동기 호출 (결과 기다림)
sisyphus_task(
  agent: "oracle",
  prompt: "...",
  run_in_background: false
)

// 비동기 호출 (백그라운드 실행)
const taskId = sisyphus_task(
  agent: "explore",
  prompt: "...",
  run_in_background: true
)
// 나중에 결과 수집
background_output(task_id: taskId)

// 이전 세션 이어서 (컨텍스트 유지)
sisyphus_task(
  resume: "ses_abc123",
  prompt: "추가로 이것도 확인해줘..."
)
```

---

## 4. Commands (슬래시 커맨드)

### 4.1 Commands란?

Commands는 **복잡한 워크플로우를 한 줄로 실행**하는 단축키입니다.

```
일반 요청:    "계획 세우고, 검토하고, 실행해줘" (여러 단계)
Command:     /plan "기능 구현"  (한 줄)
```

### 4.2 주요 Commands

| Command | 용도 | 설명 |
|---------|------|------|
| `/plan` | 계획 수립 | Prometheus와 함께 상세 계획 작성 |
| `/review` | 계획 검토 | Momus가 계획을 비판적으로 검토 |
| `/sisyphus` | 멀티에이전트 실행 | 복잡한 작업을 에이전트들이 협업 |
| `/deepsearch` | 심층 검색 | 코드베이스 전체 검색 |
| `/analyze` | 분석 | 깊은 분석 및 조사 |
| `/ultrawork` | 최대 성능 모드 | 병렬 에이전트로 빠른 처리 |

### 4.3 실사용 예시

#### /plan - 계획 수립

```
사용자: /plan 사용자 알림 시스템 구현

Prometheus: "알림 시스템에 대해 몇 가지 질문이 있습니다:

1. 알림 채널은 어떤 것들이 필요한가요?
   - 이메일? 푸시? 인앱?
   
2. 실시간 알림이 필요한가요?
   - WebSocket? SSE? Polling?

3. 알림 우선순위 구분이 필요한가요?

..."

사용자: "이메일이랑 인앱만. 실시간은 SSE로."

Prometheus: "이해했습니다. 계획을 작성합니다..."

[상세 계획 문서 생성]
```

#### /review - 계획 검토

```
사용자: /review

Momus: "계획을 검토했습니다. 다음 문제점들이 있습니다:

❌ 치명적 문제:
1. 알림 실패 시 재시도 로직 누락
2. 대량 알림 시 rate limiting 미고려

⚠️ 개선 권장:
1. 알림 템플릿 관리 방안 불명확
2. 사용자별 알림 설정 기능 누락

✅ 잘된 점:
1. SSE 선택 적절
2. 데이터 모델 설계 양호

수정 후 다시 검토 요청해주세요."
```

#### /sisyphus - 멀티에이전트 실행

```
사용자: /sisyphus 알림 시스템 구현해줘

Sisyphus: "작업을 분할하고 에이전트에게 할당합니다:

📋 작업 목록:
├─ [1] 데이터 모델 정의 (진행 중)
├─ [2] API 엔드포인트 구현 (대기)
├─ [3] SSE 연결 구현 (대기)
├─ [4] 프론트엔드 컴포넌트 (대기)
└─ [5] 테스트 작성 (대기)

[Frontend-Engineer에게 위임: 알림 UI 컴포넌트]
[직접 처리: 백엔드 API]

..."
```

#### /deepsearch - 심층 검색

```
사용자: /deepsearch 메모리 누수 원인

Deepsearch: "코드베이스 전체를 검색합니다...

🔍 검색 범위:
- 이벤트 리스너 등록/해제
- setInterval/setTimeout
- 캐시 관련 코드
- 구독 패턴

📍 발견된 잠재적 문제:
1. src/hooks/useWebSocket.ts:45
   - cleanup에서 listener 해제 누락
   
2. src/services/cache.ts:120
   - TTL 없는 캐시 항목
   
3. src/components/Chart.tsx:89
   - 컴포넌트 언마운트 시 interval 미정리

..."
```

### 4.4 커스텀 Command 만들기

**Command 파일 위치**: `~/.claude/commands/deploy-check.md`

```markdown
# /deploy-check

배포 전 체크리스트를 자동으로 실행합니다.

## Steps

1. 린트 검사 실행
   ```bash
   npm run lint
   ```

2. 타입 체크
   ```bash
   npm run typecheck
   ```

3. 테스트 실행
   ```bash
   npm run test
   ```

4. 빌드 확인
   ```bash
   npm run build
   ```

5. 환경 변수 확인
   - 필수 환경 변수가 설정되어 있는지 확인
   - .env.example과 비교

6. 결과 리포트
   - 각 단계 성공/실패 요약
   - 실패 시 원인 분석
```

---

## 5. Hooks

### 5.1 Hooks란?

Hooks는 **특정 이벤트가 발생할 때 자동 실행되는 트리거**입니다.

```
이벤트 발생 → Hook 실행 → 자동화된 작업
```

### 5.2 지원되는 Hook 타입

| Hook | 트리거 시점 | 용도 |
|------|------------|------|
| `PreToolUse` | 도구 실행 전 | 위험한 명령 차단 |
| `PostToolUse` | 도구 실행 후 | 결과 로깅, 후처리 |
| `Notification` | 작업 완료 시 | 알림 전송 |
| `Stop` | 세션 종료 시 | 정리 작업 |

### 5.3 실사용 예시

#### 위험한 명령 차단

```javascript
// ~/.claude/hooks/pre-tool-use.js
export default {
  name: "safety-check",
  event: "PreToolUse",
  
  async handler({ tool, params }) {
    // rm -rf 차단
    if (tool === "bash" && params.command.includes("rm -rf /")) {
      return {
        decision: "block",
        message: "위험한 명령이 감지되었습니다."
      };
    }
    
    // 프로덕션 DB 접근 경고
    if (params.command?.includes("prod") && params.command?.includes("db")) {
      return {
        decision: "ask",
        message: "프로덕션 DB에 접근하려고 합니다. 계속할까요?"
      };
    }
    
    return { decision: "allow" };
  }
};
```

#### 작업 완료 알림

```javascript
// ~/.claude/hooks/notification.js
export default {
  name: "slack-notify",
  event: "Stop",
  
  async handler({ summary, duration }) {
    // 10분 이상 걸린 작업만 알림
    if (duration > 600) {
      await fetch(process.env.SLACK_WEBHOOK, {
        method: "POST",
        body: JSON.stringify({
          text: `Claude 작업 완료: ${summary} (${duration}초)`
        })
      });
    }
  }
};
```

#### TODO 자동 추적

```javascript
// ~/.claude/hooks/todo-tracker.js
export default {
  name: "todo-tracker",
  event: "PostToolUse",
  
  async handler({ tool, result }) {
    if (tool === "todowrite") {
      // TODO 변경 시 자동 로깅
      console.log("[TODO 변경]", result.todos);
      
      // 완료율 계산
      const completed = result.todos.filter(t => t.status === "completed").length;
      const total = result.todos.length;
      console.log(`진행률: ${completed}/${total}`);
    }
  }
};
```

### 5.4 Hook 설정

**설정 파일** (`~/.claude/settings.json`):

```json
{
  "hooks": {
    "preToolUse": [
      "~/.claude/hooks/safety-check.js"
    ],
    "postToolUse": [
      "~/.claude/hooks/todo-tracker.js"
    ],
    "notification": [
      "~/.claude/hooks/slack-notify.js"
    ]
  }
}
```

---

## 6. 실전 워크플로우 예시

### 6.1 새 기능 개발 (처음부터 끝까지)

```
1️⃣ 계획 수립
사용자: /plan 장바구니 기능 구현

   [Prometheus가 인터뷰 진행]
   [상세 계획 문서 생성]

2️⃣ 계획 검토  
사용자: /review

   [Momus가 비판적 검토]
   [문제점 수정]

3️⃣ 사전 조사
   [Explore] 기존 코드 패턴 검색
   [Librarian] 관련 라이브러리 문서 확인
   [병렬 실행]

4️⃣ 구현
사용자: /sisyphus 장바구니 구현 시작

   [TODO 자동 생성]
   [단계별 구현]
   [Frontend-Engineer에게 UI 위임]

5️⃣ 검증
   [테스트 자동 실행]
   [Playwright로 E2E 테스트]

6️⃣ 완료
   [Hook: Slack 알림 전송]
```

### 6.2 버그 긴급 수정

```
1️⃣ 문제 파악
사용자: "프로덕션에서 결제 에러 발생 중!"

   [자동으로 관련 로그 검색]
   [에러 패턴 분석]

2️⃣ 원인 분석
   [Oracle 상담]
   "근본 원인은 race condition으로 보입니다..."

3️⃣ 수정
   [Git-Master Skill 활용]
   [최소한의 변경으로 수정]
   [테스트 추가]

4️⃣ 배포
사용자: /deploy-check

   [자동 체크리스트 실행]
   [모든 검증 통과]

5️⃣ 모니터링
   [Hook: 배포 완료 알림]
```

### 6.3 코드 리뷰 자동화

```
1️⃣ PR 생성 시
   [Hook: PR 이벤트 감지]
   
2️⃣ 자동 리뷰
   [Explore: 변경된 파일 분석]
   [Oracle: 아키텍처 영향 검토]
   
3️⃣ 리뷰 코멘트 작성
   "다음 사항을 확인해주세요:
    - 라인 42: null 체크 추가 권장
    - 라인 89: 이 변경이 X에 영향..."

4️⃣ 테스트 실행
   [Playwright: 관련 E2E 테스트]
   [결과 리포트]
```

---

## 부록: 설정 파일 총정리

### 디렉토리 구조

```
~/.claude/
├── settings.json          # 전역 설정
├── mcp_servers.json       # MCP 서버 설정
├── CLAUDE.md              # 프로젝트 컨텍스트 (또는 프로젝트 루트에)
├── skills/
│   ├── payment.md         # 커스텀 스킬
│   └── deployment.md
├── commands/
│   ├── deploy-check.md    # 커스텀 커맨드
│   └── hotfix.md
└── hooks/
    ├── safety-check.js    # 안전 검사 훅
    ├── todo-tracker.js    # TODO 추적 훅
    └── slack-notify.js    # 알림 훅
```

### 빠른 시작 체크리스트

- [ ] `CLAUDE.md` 작성 (프로젝트 컨텍스트)
- [ ] 필요한 MCP 서버 연결
- [ ] 팀 공통 Skills 설정
- [ ] 안전 관련 Hooks 설정
- [ ] 자주 쓰는 Commands 정의

---

## 마무리

Claude Code의 고급 기능들을 활용하면:

1. **MCP**: 외부 도구와 원활한 연동
2. **Skills**: 도메인 전문가 수준의 작업
3. **Subagents**: 복잡한 작업의 효율적 분담
4. **Commands**: 반복 작업의 자동화
5. **Hooks**: 이벤트 기반 자동화

> **"도구를 아는 것과 도구를 마스터하는 것은 다르다"**
