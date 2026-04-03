---
layout: post
title: "CMUX로 Claude Code Agent Team 사용하기"
subtitle: "2026-04-01-claude-code-team-agent-cmux.md"
date: 2026-04-01 14:40:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [ai, claude, claude code, cmux, tmux]
---

# CMUX Claude Code Agent Team 사용하기

## Claude Code Agent Team이란?

Claude Code Agent Team은 **여러 개의 Claude Code 인스턴스가 하나의 팀으로 협업**하는 멀티 에이전트 기능이다. 2026년 2월 Opus 4.6 모델과 함께 출시된 실험적 기능으로, Claude Code v2.1.32 이상에서 사용할 수 있다.

### 핵심 개념

| 구성요소 | 역할 |
|----------|------|
| **Team Lead** | 팀을 생성하고, 작업을 분배하며, 결과를 종합하는 메인 세션 |
| **Teammate** | 각자 독립된 컨텍스트 윈도우에서 할당된 작업을 수행하는 에이전트 |
| **Task List** | 팀원들이 공유하며, 작업을 claim하고 완료 상태를 추적하는 공유 태스크 목록 |
| **Mailbox** | 에이전트 간 직접 메시지를 주고받는 통신 시스템 |

### Subagent와의 차이

| | Subagent | Agent Team |
|--|----------|------------|
| **통신** | 메인 에이전트에게만 결과 보고 | 팀원끼리 직접 메시지 교환 |
| **조율** | 메인 에이전트가 모든 작업 관리 | 공유 태스크 리스트로 자율 조율 |
| **컨텍스트** | 결과가 메인 컨텍스트로 요약 반환 | 각 팀원이 완전히 독립된 컨텍스트 보유 |
| **적합한 경우** | 결과만 필요한 단순 위임 | 토론과 협업이 필요한 복잡한 작업 |

> Subagent가 "인턴에게 조사를 시키고 결과만 받는 것"이라면, Agent Team은 "프로젝트 팀을 구성해서 각자 맡은 부분을 하되 서로 소통하는 것"이다.

### 적합한 사용 사례

- **리서치 & 리뷰**: 여러 팀원이 서로 다른 측면을 동시 조사
- **새 모듈/기능 개발**: 팀원별로 독립된 모듈을 담당
- **경쟁 가설 디버깅**: 팀원마다 다른 원인을 병렬 추적
- **크로스 레이어 작업**: 프론트엔드, 백엔드, 테스트를 각각 담당

### 활성화 방법

Agent Team은 기본적으로 비활성화되어 있다. `settings.json`에 다음을 추가하여 활성화한다:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 디스플레이 모드

| 모드 | 설명 | 조건 |
|------|------|------|
| **in-process** | 모든 팀원이 하나의 터미널에서 실행. `Shift+Down`으로 순회 | 모든 터미널에서 동작 |
| **split panes** | 각 팀원이 별도 pane에 표시. 동시에 모든 출력 확인 가능 | tmux, iTerm2, 또는 **cmux** 필요 |

---

## cmux란?

[cmux](https://github.com/manaflow-ai/cmux)는 AI 코딩 에이전트를 위한 **Ghostty 기반 macOS 네이티브 터미널**이다.
수직 탭, 에이전트 알림, 내장 브라우저, GPU 가속 렌더링을 제공하며, **tmux 없이** Claude Code Agent Team의 split pane 모드를 네이티브로 실행할 수 있다.

---

## cmux가 에이전트 협업에 강한 이유

Agent Team의 핵심 가치는 **에이전트 간 소통과 공유**다. 그러나 이를 제대로 활용하려면 여러 에이전트의 상태를 동시에 관찰하고, 적시에 개입할 수 있어야 한다. cmux는 이 부분에서 기존 터미널 대비 명확한 이점을 제공한다.

### 1) 에이전트 간 메시지 교환의 가시성

Agent Team에서 팀원들은 Mailbox를 통해 직접 메시지를 주고받는다. 예를 들어 Agent 1이 "UserService 구현 완료, API 스펙은 이렇다"라는 메시지를 Agent 2에게 보내면, Agent 2는 이를 받아 통합 테스트를 시작한다.

| 환경 | 메시지 교환 관찰 방식 |
|------|----------------------|
| **in-process** (일반 터미널) | `Shift+Down`으로 한 번에 하나의 에이전트만 확인. 메시지가 오갔는지 확인하려면 에이전트를 일일이 순회해야 함 |
| **tmux split pane** | 여러 pane에서 동시 확인 가능하나, 어떤 에이전트가 메시지를 기다리는지 알 수 없음 |
| **cmux split pane** | 모든 에이전트를 동시에 보면서, **입력 대기 중인 에이전트에 파란색 알림**이 표시됨. `Cmd+Shift+U`로 즉시 이동 가능 |

### 2) 공유 태스크 리스트 모니터링

Agent Team은 공유 Task List를 통해 작업을 조율한다. 팀원이 태스크를 claim하고, 완료하면 의존성이 있는 다음 태스크가 자동으로 unblock된다.

- **일반 터미널**: Lead 세션에서만 전체 태스크 상태를 확인할 수 있다
- **cmux**: 사이드바에서 각 에이전트의 현재 작업 상태를 한눈에 확인하고, 하단 상태바에서 `Tasks: 3/5 complete` 같은 전체 진행률을 실시간 추적할 수 있다

### 3) 적시 개입이 가능한 알림 시스템

에이전트 협업에서 가장 큰 낭비는 **팀원이 입력을 기다리는데 아무도 모르는 상황**이다.

- Agent 2가 "Agent 1의 API 스펙이 필요합니다"라고 메시지를 보내고 대기 중일 때
- Agent 1이 permission 승인을 기다리며 멈춰 있을 때
- Lead가 Plan 승인 요청을 보냈을 때

cmux는 이런 대기 상태를 **파란색 링 알림**으로 즉시 알려준다. `Cmd+I`로 알림 패널을 열면 모든 대기 중인 에이전트를 한곳에서 확인하고 순서대로 처리할 수 있다.

### 4) 내장 브라우저로 문서 공유 맥락 확보

에이전트가 참조하는 문서(API 문서, PR 페이지, 이슈 등)를 터미널 옆에 분할하여 열 수 있다. 에이전트가 "PR #42의 변경사항을 확인해주세요"라고 할 때, `Cmd+Shift+L`로 바로 해당 페이지를 열어 맥락을 공유하며 작업할 수 있다.

---

## 1. 설치

### Homebrew (권장)

```bash
brew tap manaflow-ai/cmux
brew install --cask cmux
```

### DMG 직접 다운로드

[cmux-macos.dmg](https://github.com/manaflow-ai/cmux/releases/latest/download/cmux-macos.dmg) 다운로드 후 Applications 폴더로 드래그. Sparkle을 통해 자동 업데이트된다.

---

## 2. Claude Code Teams 실행

### 기본 실행

```bash
cmux claude-teams
```

이 명령만으로 teammate 모드가 실행되며, 각 팀원이 **네이티브 분할 pane**으로 표시된다.

### 프로젝트 디렉토리 지정

```bash
cmux claude-teams /path/to/project
```

### 프롬프트와 함께 실행

```bash
cmux claude-teams -p "UserService와 TodoService를 구현하고 테스트를 작성해줘" --add-dir /path/to/project
```

### 모델 지정 + worktree 격리

```bash
cmux claude-teams --model sonnet --worktree /path/to/project
```

> `cmux claude-teams` 뒤에 오는 인자는 모두 `claude` CLI에 그대로 전달된다.

---

## 3. cmux에서의 팀 작업 화면

![cmux Claude Teams 워크스페이스](/img/posts/ai/claude/cmux-claude-team-1.png)              

- **왼쪽 사이드바**: 워크스페이스 목록 (Lead, Agent 1, Agent 2), Git 브랜치 및 PR 상태 표시
- **메인 영역**: 각 에이전트가 네이티브 분할 pane으로 동시에 표시
- **알림**: 에이전트가 입력을 기다리면 **파란색 링 + Awaiting Input 배지**가 표시된다
- `Cmd+Shift+U`로 가장 최근 미읽 알림 에이전트로 즉시 이동할 수 있다
- 하단 상태바에서 활성 에이전트 수, 태스크 진행률, 토큰 사용량을 확인할 수 있다

---

## 4. 주요 단축키

| 동작 | 단축키 |
|------|--------|
| 새 워크스페이스 | `Cmd+N` |
| 우측 분할 | `Cmd+D` |
| 아래 분할 | `Cmd+Shift+D` |
| 알림 패널 열기 | `Cmd+I` |
| 최신 미읽 알림으로 이동 | `Cmd+Shift+U` |
| 브라우저 분할 | `Cmd+Shift+L` |
| 워크스페이스 전환 | `Cmd+1` ~ `Cmd+8` |
| 다음 탭 | `Ctrl+Tab` |
| 브라우저 주소창 | `Cmd+L` |

---

## 5. cmux vs tmux 비교

| 기능 | tmux | cmux |
|------|------|------|
| 에이전트 알림 | 없음 | 파란색 링 + 알림 패널 |
| 미읽 알림 이동 | 없음 | `Cmd+Shift+U` |
| 내장 브라우저 | 없음 | 터미널 옆 브라우저 분할 |
| Git/PR 상태 | 없음 | 사이드바에 브랜치, PR 상태 표시 |
| GPU 가속 | 없음 | Ghostty 기반 렌더링 |
| 설정 | `tmux.conf` | Ghostty config 호환 + `cmux.json` |
| SSH 원격 접속 | `ssh` + tmux attach | `cmux ssh user@remote` (브라우저 포함) |
| Claude Teams | `/omc-teams` 스킬 필요 | `cmux claude-teams` 한 줄 |

---

## 6. 프로젝트별 커스텀 명령어

프로젝트 루트에 `cmux.json`을 만들어 프로젝트별 작업을 정의할 수 있다.

```json
{
  "commands": {
    "dev": "npm run dev",
    "test": "npm test",
    "team": "cmux claude-teams --model sonnet"
  }
}
```

---

## 7. 추가 기능

- **내장 브라우저**: Chrome, Firefox, Arc 등에서 쿠키와 세션 자동 가져오기
- **SSH 지원**: `cmux ssh user@remote`로 원격 워크스페이스 생성, 브라우저는 원격 네트워크를 통해 라우팅
- **스크립트 가능**: CLI와 소켓 API로 자동화 가능

---

## 8. 실전 예제

### 예제 1: 병렬 코드 리뷰

```text
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

**cmux에서의 협업 흐름:**

1. `cmux claude-teams`로 실행하면 3개의 리뷰어 pane이 자동 분할된다
2. 각 리뷰어가 독립적으로 PR을 분석하는 과정이 **모든 pane에서 동시에 보인다**
3. Security 리뷰어가 "SQL Injection 가능성 발견"을 Mailbox로 공유하면, Performance 리뷰어 pane에서 이 메시지를 수신하고 관련 쿼리의 성능 영향도 추가 분석하는 것을 실시간으로 확인할 수 있다
4. `Cmd+Shift+L`로 PR 페이지를 내장 브라우저에 열어, 리뷰어의 지적 사항을 코드와 대조하며 확인한다
5. 3명 모두 완료되면 Lead가 결과를 종합 — 하단 상태바에서 `Tasks: 3/3 complete` 확인

### 예제 2: 경쟁 가설 디버깅

```text
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific debate.
```

**cmux에서의 협업 흐름:**

1. 5개 에이전트가 각각 다른 가설을 추적 (연결 타임아웃, 메모리 누수, 이벤트 루프 블로킹 등)
2. Agent 3이 "이벤트 루프 블로킹이 원인"이라는 증거를 발견하고 Mailbox로 브로드캐스트하면, 다른 에이전트 pane에서 이 메시지를 수신하고 반박 근거를 찾는 모습이 **동시에 보인다**
3. Agent 1이 "타임아웃 설정 문제는 아님"이라고 자기 가설을 기각하고 idle 상태에 들어가면 — cmux 사이드바에 해당 에이전트가 **비활성 표시**로 바뀐다
4. 아직 작업 중인 에이전트와 대기 중인 에이전트를 사이드바에서 한눈에 구분할 수 있어, 토론이 수렴되는 시점을 쉽게 판단할 수 있다
5. 최종적으로 살아남은 가설이 실제 원인 — Lead가 종합 보고서를 작성

### 예제 3: 기능 병렬 구현

```text
Create a team with 3 teammates to implement in parallel:
- Teammate 1: UserService (src/userService.js)
- Teammate 2: TodoService (src/todoService.js)
- Teammate 3: Integration tests (test/)
Use Sonnet for each teammate.
```

**cmux에서의 협업 흐름:**

1. 3개 pane에서 각 팀원이 독립된 파일을 동시에 구현하는 과정이 보인다
2. Teammate 3 (테스트 담당)은 Teammate 1, 2의 구현 완료를 기다려야 한다 — Task List에 의존성이 설정되어 **자동으로 blocked 상태**
3. Teammate 1이 UserService 구현을 완료하면 → Mailbox로 "API 스펙: addUser(name, email) → {id, name, email}" 메시지를 Teammate 3에게 전달
4. cmux에서 Teammate 3의 pane에 **파란색 알림 링**이 뜨고, `Cmd+Shift+U`로 이동하면 Teammate 3이 메시지를 받아 테스트 코드를 작성하기 시작한 것을 확인할 수 있다
5. Teammate 2도 완료되면 의존성이 모두 해소 → Teammate 3이 통합 테스트까지 자동 진행
6. 전체 진행 상황은 하단 상태바 `Tasks: 5/5 complete`로 확인

---

## 9. 팀 운영 팁

- **팀 규모**: 3~5명으로 시작. 팀원당 5~6개 태스크가 적당하다.
- **파일 충돌 방지**: 각 팀원이 서로 다른 파일을 담당하도록 작업을 분배한다.
- **Lead 대기 유도**: Lead가 직접 구현하려 하면 `Wait for your teammates to complete their tasks before proceeding`라고 지시한다.
- **Plan 승인 모드**: 위험한 작업은 `Require plan approval before they make any changes`로 팀원의 계획을 먼저 검토할 수 있다.
- **팀 종료**: 작업 완료 후 반드시 Lead에게 `Clean up the team`을 지시한다. 팀원을 먼저 종료한 뒤 cleanup한다.

---

## 참고 링크

- Claude Code Agent Teams 공식 문서: <https://code.claude.com/docs/en/agent-teams>
- cmux GitHub: <https://github.com/manaflow-ai/cmux>
- cmux Discord: <https://discord.gg/xsgFEVrWCZ>
- cmux X: [@manaflowai](https://x.com/manaflowai)