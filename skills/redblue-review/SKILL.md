---
name: redblue-review
description: codex(RED)로 공격하고 Claude(BLUE)로 검증·최소수정 방어하는 N사이클 적대적 코드 리뷰 워크플로. 사이클마다 문서화·테스트·커밋하고 마지막에 회귀 재공격+종합 보고서를 만든다. "red/blue 리뷰", "적대적 코드 리뷰", "공격/방어 사이클", "codex로 공격하고 방어", "N 사이클 취약점 점검·하드닝" 요청 시 사용. 비용·시간이 크므로 사용자가 명시적으로 요청했을 때만 실행한다.
---

# RED Team vs BLUE Team 적대적 코드 리뷰 사이클

codex plugin을 **RED Team**(취약점 공격/탐지)으로, Claude를 **BLUE Team**(검증·방어·하드닝)으로 두고, 한 코드베이스를 **N 사이클**(기본 6)에 걸쳐 공격→방어→문서화→테스트→커밋한다. 각 사이클은 서로 다른 초점(구조/엔드포인트/도메인 서브시스템/보안/동시성/회귀)을 공격한다.

> 핵심 철학: RED는 **공격만**(코드 수정 금지), BLUE는 **검증 후 최소 변경**으로만 방어. 사이클마다 **커밋+테스트 체크포인트**를 두어 회귀를 조기 차단하고, 큰 리팩토링/계약 변경/인프라 변경은 **근거와 함께 이월**한다.

## 전제 조건
- git 저장소일 것 (사이클별 커밋).
- **codex CLI** 사용 가능 (`which codex`).
- **codex 플러그인** 설치됨 — RED는 CLI 단독이 아니라 codex 플러그인(`codex-companion.mjs` / `codex:codex-rescue` 서브에이전트)에 의존한다. CLI나 플러그인 중 하나라도 없으면 중단하고 `/codex:setup`을 안내한다.
- 프로젝트의 **테스트 실행 방법**을 알 것(또는 탐색). 없으면 사용자에게 확인.

## 0단계: 시작 전 확인 (AskUserQuestion)

> ⚠️ **외부 전송 고지(필수)**: RED(codex)는 코드베이스를 외부 LLM 제공자(OpenAI)로 전송해 분석한다. 비공개/사내 코드라면 **반드시 사용자 동의를 먼저 받고**, 미동의 시 실행하지 않는다.

비용/리스크가 크므로 시작 전 다음을 확정한다(`AskUserQuestion`은 **호출당 최대 4문항**이므로 2회로 나눠 묻거나 관련 항목을 multiSelect로 묶는다):
1. **외부 전송 동의**: codex가 코드를 외부(OpenAI)로 전송함에 동의하는가(비공개 코드면 특히 확인). 미동의 시 중단.
2. **BLUE 실행자**: Claude가 방어 구현(권장) vs codex가 방어도 수행.
3. **작업 브랜치**: 전용 작업 브랜치(권장, 예 `blue`/`review`)에 사이클별 커밋 vs 현재 브랜치.
4. **검증 정책**: 사이클별 커밋+테스트(권장) vs 커밋만 vs 검증 없음.
5. **공격 범위**: 코드 품질+보안 통합(권장) vs 코드 품질 위주.
6. (선택) **사이클 수**(기본 6)와 각 사이클 **초점**.

블라인드 스폿 지적: 중간 검토 없는 다중 사이클 자율 수정은 회귀·스코프 크리프·상충 수정 위험이 있으므로 사이클별 체크포인트를 강하게 권장한다.

## 1단계: 셋업
- `TaskCreate`로 `Setup + Cycle 1..N + 종합보고서` 태스크를 만들어 진행 추적.
- 작업 브랜치 생성/전환.
- 문서 디렉토리 생성: `docs/redteam/` (또는 프로젝트 관례에 맞는 위치). `README.md`에 진행 방식·사이클 표·문서 포맷을 기록.
- **코드베이스 정찰**: `Explore` 서브에이전트로 아키텍처/엔드포인트/도메인 서브시스템/교차관심사 지도를 작성(BLUE의 검증·수정 컨텍스트). 결론만 받고 파일 덤프는 받지 않는다.
- **테스트 베이스라인**: 변경 전 테스트를 1회 돌려 **기존(pre-existing) 실패 목록**을 확정. 이후 "신규 회귀 0"의 기준이 된다.

## 2단계: 사이클 루프 (각 Cycle k)

기본 초점 템플릿(프로젝트에 맞게 조정):
| Cycle | 초점 |
|-------|------|
| 1 | 코드 구조 일원화 / 견고성(아키텍처·계층·에러처리·중복) |
| 2 | 엔드포인트/인터페이스 효율·정확성(blocking I/O·검증·상태코드·N+1) |
| 3 | 프로젝트 핵심 도메인 서브시스템(연결관리·외부연동·오케스트레이션) |
| 4 | 보안(path traversal·injection·SSRF·secret·DoS) |
| 5 | 동시성/리소스/신뢰성(race·태스크 수명·DB세션·락·shutdown) |
| 6 | 회귀 재공격(이전 수정 재검증) + 교차 하드닝 |

### (a) RED 공격 — codex (읽기 전용 강제)

> ⚠️ **읽기 전용 불변식**: `codex:codex-rescue`는 기본적으로 쓰기 가능(`task --write`, workspace-write 샌드박스)으로 동작한다. 프롬프트에 "수정 금지"만 적는 것으로는 보장되지 않으므로 **read-only 샌드박스로 실행되도록 강제**해야 한다.

**RED 호출 방식(둘 중 택1):**

1. **코드 전반/파일 초점 공격** — `Agent`로 `subagent_type: "codex:codex-rescue"` 호출. 프롬프트 **첫 줄에 read-only를 명시**해 codex-rescue가 `--write`를 붙이지 않도록 한다(그래야 read-only 샌드박스로 실행됨):
   - "READ-ONLY. 리뷰/진단만 한다. **파일 수정·패치 적용·git 작업 금지.** 저장소는 이미 작업 브랜치에 체크아웃됨 — `git checkout` 금지, 파일은 직접 읽기만."
2. **변경분(diff) 적대 리뷰 (read-only 하드 보증 — diff가 있으면 우선)** — 직전 BLUE 커밋이나 브랜치 전체를 정조준할 때는, 리뷰어가 구조적으로 read-only를 강제하는 codex 플러그인 리뷰를 쓴다(옵션 1의 프롬프트 유도보다 안전). 용도로 구분:
   - `adversarial-review` — **설계·접근 도전**("BLUE 방어가 근본 차단인가, 우회 가능한가"). 마지막 사이클 **회귀 재공격**에 특히 적합.
   - `review`(내장 defect 리뷰) — diff에 새로 생긴 **구체 결함**을 더 캐고 싶을 때.
   ```bash
   SCRIPT="$(find ~/.claude/plugins/marketplaces -path '*plugins/codex/scripts/codex-companion.mjs' | head -1)"
   [ -f "$SCRIPT" ] || { echo "codex companion 미발견 — 플러그인 확인"; exit 1; }
   node "$SCRIPT" adversarial-review --wait --scope branch --base <직전-사이클-기준-ref> "이번 사이클 초점 …"
   ```
   `--base`로는 직전 사이클의 `cycle-k` 태그(아래 (e))를 쓴다.

**프롬프트 공통 포함 사항:**
- **이번 사이클 초점**과 **공격할 핵심 파일 목록**(정찰 지도 기반).
- 이전 사이클이 다룬 범위를 알려 **중복 회피**, 마지막 사이클은 **이전 BLUE 수정 재공격(REGRESSION) + 미수정 잔여(REMAINING)**.
- 출력 포맷 강제: 각 발견을 `ID(Ck-NN) / Title / Severity(CRITICAL|HIGH|MEDIUM|LOW + 근거) / Location(file:line) / Problem(공격 시나리오) / Impact / Suggested fix` 마크다운으로. **건수는 강제하지 말고 실제 있는 만큼**(없으면 "없음", 상한 ~15건 권장 — 억지로 채우면 오탐만 늘어난다).
- 인증 부재 등 **의도된 설계는 제외**하라고 명시(해당 시).

**RED 직후 안전 점검:** `git status --short`로 **작업트리 변경 0**을 확인한다. RED가 파일을 건드렸다면(샌드박스 미적용 등) 해당 변경을 즉시 폐기(`git checkout -- .` / `git clean -fd`)하고 read-only로 재호출한다.

**샌드박스 실패 시:** macOS는 codex가 Seatbelt(`sandbox-exec`), Linux는 bubblewrap(`bwrap`)을 사용한다. 샌드박스 초기화 오류(macOS의 seatbelt 거부, Linux `bwrap: loopback` 등)가 나면 **재시도**한다(브랜치 전환 불필요·로컬 git만 사용 명시).

### (b) BLUE 방어 — Claude
1. **검증**: 각 발견을 실제 코드와 대조. 오탐/이미-안전(예: 이미 락으로 보호됨, 별도 디렉토리)을 가려낸다. RED 오탐을 잡는 것도 성과다.
2. **선별**: 명백·저위험·고가치 수정만 이번 사이클에 적용(**최소 변경 원칙**). 다음은 **이월**하고 문서에 근거 기록:
   - 대규모 아키텍처 리팩토링, transport/응답 **계약 변경**(상태코드·필드), **인프라 변경**(인증 도입·키 기반 SSH), 한도값/정책 **결정 필요** 항목.
   - ⚠️ **에스컬레이션**: 이월 대상이 **CRITICAL/HIGH**(예: 인증 없는 RCE 경로, 데이터 유실)면 사이클 끝까지 묻어두지 말고 **즉시 사용자에게 통지**하고 우선 처리 여부를 묻는다. "이월=점검 완료"로 오인되지 않게 summary 최상단에도 노출한다.
3. **구현**: 기존 코드의 패턴/유틸을 먼저 찾아 재사용(예: 같은 파일의 기존 검증 정규식, 기존 `to_thread` 오프로드 패턴, sink-level 방어). 같은 부류 버그는 일관된 방식으로 수정.
4. 같은 파일에 여러 수정이 필요하면 `old_string`이 고유하도록 충분한 컨텍스트로 나눠 Edit.

흔한 방어 레시피(참고 — **Python/asyncio·FastAPI 비동기 서비스에 튜닝됨**; JS/Go/Rust 등 타 스택은 개념(오프로드·경계검증·sink 방어·태스크 수명)만 차용하고 구체 API는 해당 스택에 맞게 치환):
- 이벤트 루프 blocking → `await asyncio.to_thread(...)` 오프로드 / 동시화는 `asyncio.gather`(+`Semaphore`로 상한).
- 무음 실패(`except: pass`, 오류→0/empty) → 최소 로깅 추가(동작 유지).
- path traversal → 입력 정규식 검증 + 경로 구분자/`..` 거부, sink에서 `resolve().relative_to(root)` 컨테인먼트.
- command injection → argv(`create_subprocess_exec`) 또는 인자 화이트리스트 검증(`^[A-Za-z0-9._-]+$`).
- 입력 경계 → 리스트/숫자에 `min/max_length`·`ge/le`.
- race → 락 범위 확대/원자화, fail-open vs fail-closed는 **데이터 무결성 관점**으로 판단(과소추정이 더 위험하면 last-known 보존).
- 백그라운드 루프/태스크 → 예외 격리(`try/except` in loop), `create_task`는 레지스트리+`add_done_callback`으로 참조·예외 관측.

### (c) 문서화 — `docs/redteam/cycle-k.md`
구조: ① 공격(RED 발견 표) ② 방어(FIX: 검증결과+수정+파일 / 오탐 / 이월+사유) ③ 검증(테스트 결과·변경 파일) ④ 잔여 리스크·이월.

### (d) 테스트
프로젝트 테스트를 실행하고 **베이스라인 대비 신규 실패 0**을 확인.
- **신규 실패가 생기면 커밋하지 않는다.** 원인을 수정해 재검증으로 0으로 되돌리거나, 되돌릴 수 없으면 이번 사이클의 해당 수정을 `git checkout`/`git revert`로 **롤백**한 뒤 문서에 사유를 남긴다. 회귀를 안고 다음 사이클로 넘어가지 않는다.
- 테스트가 불가하면 **구문만** 빠르게 확인(import·실행·`.pyc` 생성 없음). 전체 파일을 검사하고 실패만 모아 보고(성공 시 무출력, 실패 시 `file:line` + exit 1):
  ```bash
  python3 -c '
  import ast, sys
  errs = []
  for p in sys.argv[1:]:
      try: ast.parse(open(p, encoding="utf-8").read(), p)
      except SyntaxError as e: errs.append(f"{p}:{e.lineno}: {e.msg}")
  if errs: print("\n".join(errs))
  sys.exit(1 if errs else 0)
  ' <파일…>
  ```

### (e) 커밋
**(d)에서 신규 실패 0을 확인한 경우에만** 커밋한다. 의도한 파일만 명시적으로 `git add`(런타임 산출물·무관 변경 제외) 후 사이클별 커밋. 커밋 메시지에 적용 ID·이월 ID·테스트 결과 요약. 사이클 종료마다 가벼운 태그를 남겨두면(`git tag cycle-k`) 다음 RED의 회귀 재공격에서 `--base cycle-k`로 바로 쓸 수 있다.

## 3단계: 종합 보고서 — `docs/redteam/summary.md`
- 통계(총 발견/심각도 분포/적용·이월·오탐/회귀 VERIFIED), 사이클별 커밋 표, 적용 방어 분류, **우선순위별 미해결 후속**, 설계상 수용(ACCEPTED) 항목, 비고(테스트 실행법·pre-existing 실패).
- 전 사이클을 통틀어 반복 지적된 최우선 항목을 명확히 표시.

## 운영 메모
- RED 호출(서브에이전트든 companion 스크립트든)은 백그라운드가 아닌 **순차** 실행(각 사이클은 직전 방어 위에서 재공격). 테스트는 `run_in_background`로 돌리고 완료 통지 후 이어가면 효율적.
- 푸시는 outward-facing이므로 사용자가 명시 요청하기 전엔 **로컬 커밋만** 한다.
- 프로젝트별 테스트 환경 차이(컨테이너/가상환경/누락 의존성)는 셋업 단계에서 한 번 해결하고 메모리/문서에 기록.
- **중단·재개**: 각 사이클은 커밋(+`cycle-k` 태그)과 `docs/redteam/cycle-k.md`로 자기완결적 체크포인트가 된다. 중단 후 재개 시 `TaskGet`/마지막 커밋·문서를 읽어 **다음 미완료 사이클부터** 이어가고, 같은 사이클을 중복 실행하지 않는다.
