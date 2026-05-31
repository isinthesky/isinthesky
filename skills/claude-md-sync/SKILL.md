---
name: claude-md-sync
description: 저장소의 주요 디렉토리 레이어를 식별해 계층형 CLAUDE.md 문서를 현재 코드 기준으로 생성·갱신하고, 루트 문서 맵 인덱스를 동기화하는 워크플로. 문서 작성(author)은 codex(codex:codex-rescue write 모드)가 수행하고, Claude는 발견·선정·검증·인덱스 동기화를 오케스트레이션한다. 스택 무관(Electron·FastAPI·Rust·일반 client/server). 기존 하우스 스타일이 있으면 그대로 따르고 없으면 권장 템플릿을 쓴다. "CLAUDE.md 정리/업데이트", "레이어 문서 만들어줘" 류에 사용.
---

# claude-md-sync

계층형 CLAUDE.md 문서를 **현재 코드 기준**으로 발견 → 생성/갱신 → 인덱스 동기화 → 검증하는 범용 워크플로. 특정 프레임워크에 종속되지 않는다.

## 작성 주체 — 문서는 codex가 쓴다

| 단계 | 주체 | 비고 |
| --- | --- | --- |
| 발견·레이어 선정·브리프 구성 | **Claude** | §1~3 |
| **CLAUDE.md 생성/갱신(author)** | **codex** (`codex:codex-rescue`, **write 모드**) | §5 |
| 루트 문서 맵 갱신 | **codex** | §6 (CLAUDE.md이므로 codex가 씀) |
| 검증(DoD)·재위임 판단 | **Claude** | §7 |

codex 호출은 검증된 Agent 경로를 그대로 쓴다(슬래시 명령 아님):

```
Agent(subagent_type="codex:codex-rescue",
      prompt="--fresh <지시문 + 템플릿 + 제약>",   # 라우팅 플래그는 프롬프트 맨 앞
      run_in_background=true)                      # 레이어 多=background, 소수=false
```

- **write 모드 유지**: 문서를 *써야* 하므로 "read-only/수정 금지" 문구를 넣지 않는다(그러면 codex가 `--write`로 동작).
- foreground/background는 `run_in_background`로, `--fresh`/`--resume`는 프롬프트로 (review-to-zero 스킬과 동일 규약).

## 원칙

- **현재 코드 기준**: 작성 주체(codex)가 해당 디렉토리의 실제 파일/진입점/공개 API(export·pub·router)를 직접 읽고 반영한다. 추측으로 파일·규칙을 적지 않는다 — 지시문에 "실존 파일만"을 명시.
- **기존 스타일 우선**: 저장소에 이미 CLAUDE.md 하우스 스타일이 있으면 **그 섹션 어휘·포맷을 그대로 따른다**(최소 변경). 없을 때만 아래 권장 템플릿을 쓴다.
- **간결**: 레이어당 ~20–40줄. 코드 나열이 아니라 "역할 / 규칙 / 수정·이해 도움" 중심.
- **스프롤 방지**: 모든 디렉토리가 아니라 **주요 레이어만** 문서화(선정 기준 §2). 애매하면 상위 문서에 한 줄로 흡수.

## 1) 발견 (discover) — 스택 무관 (그대로 실행 가능)

```bash
# 기존 CLAUDE.md (빌드·벤더·의존 제외)
find . -name CLAUDE.md \
  -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/target/*' \
  -not -path '*/dist/*' -not -path '*/build/*' -not -path '*/.venv/*' \
  -not -path '*/vendor/*' -not -path '*/__pycache__/*' 2>/dev/null | sort

# 소스 루트 자동 감지 → 1~2 depth 하위 중 CLAUDE.md 없는 레이어 (bash·zsh 공통)
# ⚠️ zsh는 unquoted 변수를 단어분할하지 않으므로 `find $ROOTS`(X) 대신 루트별 루프를 쓴다(실측: zsh에서 0건 버그).
for r in src app lib crates packages services cmd internal pkg; do
  [ -d "$r" ] || continue
  find "$r" -maxdepth 2 -type d \
    -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/__pycache__/*' \
    -print0 2>/dev/null
done | sort -z | while IFS= read -r -d '' d; do
  [ -f "$d/CLAUDE.md" ] || echo "(없음) $d"
done
```

## 2) 레이어 선정 기준 (주요 레이어만)

**문서화 대상** — 다음을 모두 만족:
- 독립된 책임/경계가 있다(아키텍처 레이어, trust boundary, 또는 도메인 모듈).
- 비자명한 코드가 있고 자주 수정된다.

**제외** — 상위 문서에 한 줄로 흡수하거나 문서 없음:
- 빌드 산출물·벤더: `dist/ build/ target/ node_modules/ .venv/ vendor/ __pycache__/`
- 정적/자원: `assets/ fonts/ public/ static/`
- 테스트 전용: `__tests__/ tests/ spec/`(테스트 규칙은 상위 문서 한 줄로)
- 단순 배럴·상수만 있는 디렉토리.

## 3) 스택별 적용 매핑 (레이어 = 무엇을 문서화하나)

| Stack | 대표 레이어(문서화 대상) | 그 레이어에 명시할 경계/신뢰 규칙 예 |
| --- | --- | --- |
| **Electron** | `main` / `renderer` / `shared`(preload·IPC 계약) | renderer는 `fs`/`electron` 직접 접근 금지·비밀키는 main 전용·IPC 채널 단일 소스 |
| **FastAPI / 서버(Python)** | `routers`(api) / `services` / `schemas`(pydantic) / `models`·`db` / `core`(config·deps) | 입력검증=schema, 비밀/토큰=서버 전용, DB 세션·트랜잭션 경계, 의존성 주입 규칙 |
| **Rust** | workspace `crates` / 도메인 `modules` / `bin`·`lib` / `ffi`·`unsafe` | `unsafe` 격리, 공개 API(`pub`) 표면 최소화, 에러 타입 경계, feature flag |
| **Client side(SPA)** | `routes`/`pages` / `components` / `store`(state) / `lib`·`api` / `styles` | 비밀값 클라 노출 금지, API 래퍼 단일화, 표현/상태 분리 |
| **Server side(일반)** | `handlers`/`controllers` / `services` / `repositories`(data) / `domain` / `middleware` | 인증·인가 경계, 트랜잭션 경계, 외부 I/O 격리, 에러 응답 계약 |

> 핵심: 스택이 달라도 문서화 단위는 **"경계가 뚜렷한 레이어/모듈"** 로 같다. 위 매핑은 그 레이어를 찾는 힌트일 뿐.

## 4) 권장 템플릿 (기존 스타일 없을 때만)

```markdown
# <Layer Name> Guide
> Path: `<상대경로>/CLAUDE.md` | Layer: <한 줄 디스크립터>

<이 레이어가 담당하는 것 1~2문장>

## <인벤토리>            # 레이어 역할 / 핵심 진입점 / 파일 구성 / 등록되는 라우트·핸들러 중 택
- `파일|모듈`: 역할 한 줄

## 규칙                  # 경계 / 작성 / 보안 규칙 중 해당되는 것
- ...

## 수정 체크리스트        # 또는 수정 흐름 템플릿(순서가 중요한 경우)
1. ...

## 확인 포인트 / 자주 발생하는 실수   # (선택) gotcha·탐색 힌트
- ...
```

- 이 `> Path:` 헤더는 **이 권장 템플릿 한정** 요소다. 기존 저장소가 다른 스타일(예: `# X 레이어 가이드`, Path 헤더 없음)을 쓰면 **그 스타일을 따르고 Path 헤더를 강요하지 않는다**(실측: codex가 제공된 비-Path 스타일을 정확히 재현함).
- 권장 템플릿을 쓸 때 `> Path:` 헤더는 **실제 경로와 일치**해야 한다.
- 인벤토리엔 **실제 존재하는 파일/모듈만**. 사라진 항목·없는 핸들러를 적지 않는다.

## 5) 생성/갱신 절차 — Claude가 브리프, codex가 작성

레이어마다(또는 선정 레이어 묶음으로):

1. **[Claude] 브리프 구성**: codex에 넘길 지시문을 만든다 — 대상 경로, 레이어 역할·경계(§3 매핑), 따라야 할 **템플릿/기존 스타일 샘플**(저장소에 기존 문서 있으면 그 한 개를 본보기로 첨부), 제약(실존 파일만·간결 ~20–40줄·기존 문서는 드리프트 수정).
2. **[codex] 위임(write 모드)**: codex가 디렉토리의 실제 파일·공개 API(`index.*` export, `mod.rs`/`pub`, `__init__.py`, router 등록)·핵심 진입점을 **직접 읽고** CLAUDE.md를 생성/갱신한다.
   ```
   Agent(subagent_type="codex:codex-rescue",
         prompt="--fresh 다음 디렉토리들의 CLAUDE.md만 생성/갱신하라: <경로 목록>.
                 ★범위 고정: CLAUDE.md 외 어떤 파일도 생성·수정·삭제하지 말 것(소스코드 불가침).
                 각 문서는 이 템플릿/스타일을 따른다: <템플릿 또는 기존 샘플>.
                 역할·규칙·수정 체크리스트·경계(trust boundary)를 포함.
                 실존 파일만 나열하고, 관찰 가능한 코드만 기술(REST/CRUD 등 추측성 수식 금지).
                 기존 문서는 정확한 서술을 보존하고 stale한 부분만 고친다(최소 diff).
                 ★스타일 일치: 저장소의 기존 CLAUDE.md 스타일(헤더 형식·섹션 어휘)을 그대로 따른다.
                 기존 스타일이 없을 때만 권장 템플릿(§4)을 쓰고, 그 경우에만 `> Path:` 헤더를 실제 경로와 일치시킨다.",
         run_in_background=true)
   ```
   - 레이어가 많으면 한 번에(background), 소수면 `run_in_background=false`.
   - 이어서 추가 위임은 프롬프트에 `--resume`.
   - **실측 메모**: 소형 레이어 1개 작성 ≈ 16k 토큰·~2.5분. 레이어 수에 비례하므로 多수는 background+묶음 위임으로.
3. **[codex] 경계·보안 규칙 명시**: 각 레이어의 trust boundary(§3 예시)를 반드시 한 항목으로 — 지시문에 명시해 누락 방지.
4. **[Claude] 수령·점검**: codex 산출물을 §7로 검증, 미흡하면 `--resume`로 재위임.

## 6) 문서 맵(인덱스) 동기화 — codex가 갱신

- 문서 루트의 `CLAUDE.md`(보통 저장소 루트, **monorepo면 각 패키지/크레이트 루트**)도 CLAUDE.md이므로 **codex 위임으로** 문서 맵을 갱신한다(§5 지시문에 포함).
- 형식 예: `- \`<경로>/CLAUDE.md\`: <한 줄 설명>`
- 문서 맵이 없으면 만든다(전체 레이어 문서로 가는 단일 진입점). 문서 루트가 여러 개면 각각 동기화.
- Claude는 작성하지 않고, 링크 유효성만 §7에서 검증한다.

## 7) 검증 (DoD) — Claude가 실행

> background로 위임했으면 **codex 완료 통지를 기다린 뒤** 검증한다.

```bash
# (a) 문서 맵 링크가 모두 실재하는지 (문서 루트의 CLAUDE.md 기준)
grep -oE '[A-Za-z0-9_./-]+/CLAUDE\.md' CLAUDE.md | sort -u | while read -r p; do
  [ -f "$p" ] || echo "끊긴 링크: $p"
done
# (b) [Path 헤더 스타일에만 적용] Path 헤더가 있는 문서만 경로 일치 검사
#     ※ `> Path:` 미사용 스타일(예: `# X 레이어 가이드`)은 건너뜀 — 실측: 정상 문서를 거짓 플래그하던 버그 수정.
find . -name CLAUDE.md -not -path '*/node_modules/*' -not -path '*/.git/*' 2>/dev/null | while read -r f; do
  grep -q '^> *Path:' "$f" || continue
  rel=${f#./}; grep -qF "Path: \`$rel\`" "$f" || echo "Path 불일치: $f"
done
# (c) [advisory·휴리스틱, 오탐 많음] 백틱으로 언급한 소스 파일의 실재 여부 — 수동 확인용
#     오탐 원인: 패키지명(`sql.js`)·하위디렉토리 파일·타 레이어 cross-ref·네이밍 컨벤션 언급.
#     ※ 실질 드리프트 방지의 1차 보증은 codex가 author 시 실제 코드를 읽는 것(§5). 아래는 보조일 뿐.
find . -name CLAUDE.md -not -path '*/node_modules/*' -not -path '*/.git/*' 2>/dev/null | while read -r f; do
  d=$(dirname "$f")
  grep -oE '`[A-Za-z0-9_./-]+\.(ts|tsx|js|jsx|py|rs|go)`' "$f" | tr -d '`' | sort -u | while read -r ref; do
    base=$(basename "$ref")
    [ -e "$d/$ref" ] || [ -e "./$ref" ] || [ -n "$(find "$d" -name "$base" -print -quit 2>/dev/null)" ] \
      || echo "확인 필요(오탐 가능): $f → $ref"
  done
done
```

- [ ] 주요 레이어마다 문서 존재(또는 상위에 한 줄 흡수)
- [ ] (Path 헤더 쓰는 스타일이면) 헤더 = 실제 위치 (§7-b); 미사용 스타일엔 비적용
- [ ] 드리프트/환각 0 — **1차 보증은 codex의 실제-코드 기반 author(§5)**, §7-c는 advisory 보조(오탐 많음)
- [ ] 루트 문서 맵 링크 전부 유효 (§7-a)
- [ ] 레이어당 간결(요약 + 규칙 + 수정법)
- [ ] codex가 CLAUDE.md 외 파일을 건드리지 않음 — git이면 `git status`, 비-git이면 사전 파일목록 스냅샷 비교(실측: polygom-auth는 비-git)

## 가드레일

- **세션 = 대상 저장소(필수 전제)**: codex의 쓰기 가능 샌드박스는 **Claude Code 세션이 시작된 저장소**로 한정된다. 타 경로는 read/write 모두 거부(실측: polygom-auth 대상 시 `outside the writable roots`, bwrap 차단). → 이 스킬은 **문서화할 저장소 안에서 직접 실행**해야 하며, 다른 저장소를 원격 문서화할 수 없다.
- **작성은 codex, 검증은 Claude**: Claude가 CLAUDE.md를 직접 쓰지 않는다(브리프·검증·인덱스 점검만). codex 호출은 `codex:codex-rescue` **write 모드**(읽기전용 문구 금지).
- **범위 고정(필수)**: write 모드는 저장소 전체 쓰기 권한이므로, 위임 프롬프트에 "CLAUDE.md 외 파일 불가침"을 반드시 넣는다. (실측: 이 문구가 있으면 codex가 다른 파일을 건드리지 않음을 확인 — 없으면 위험)
- **추측성 수식 금지**: 코드에서 관찰되지 않는 라벨(REST/CRUD/프로토콜 등)을 붙이지 않도록 지시(실측: codex가 과수식하는 경향).
- codex 미설치/미인증이면 중단하고 `/codex:setup` 안내(작성 위임 불가).
- 추측으로 파일·규칙 기재 금지 — 지시문에 "실존 파일만·드리프트 수정"을 명시해 codex가 실제 코드 기준으로 쓰게 한다.
- 모든 디렉토리에 문서를 흩뿌리지 말 것(§2 선정 기준) — 선정은 Claude가 통제(codex에 "이 경로들만" 위임).
- 기존 하우스 스타일·섹션 어휘를 바꾸지 말 것 — 기존 문서 1개를 본보기로 지시문에 첨부.
- 대량이면 레이어 단위로 나눠 위임(긴 작업=background)하고, 단락마다 사용자 확인.
- 빌드/벤더/자원/테스트 디렉토리는 문서화 대상에서 제외.
