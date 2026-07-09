# 길드 타운 — 구현 스펙 (이 폴더의 guild.html 대상)

> 실행: 이 폴더에서 `python3 -m http.server 8080` 후 http://localhost:8080/guild.html
> 데이터: 대시보드 서버 http://127.0.0.1:7317 (cc-dashboard `pnpm dev` 떠 있어야 라이브)




# 길드 홀 픽셀아트 뷰 — 완성 스펙 (Codex 구현용)

대상 파일: **`guild.html` 단독** (자체완결 canvas 프로토타입).
서빙: vite `web/public` → `http://127.0.0.1:7173/guild.html`.
데이터: 대시보드 SSE `http://127.0.0.1:7317/api/agents/stream` (실시간 세션 목록).

> **사전 검토(Codex, 2026-07-09)**: 스펙 범위/실현성 확인 완료. Phase C의 `POST /api/open-terminal`은
> 이미 `{ cwd, command, prefer }`를 지원 → **서버 변경 불필요**. 아래 검증 스크립트는 스프라이트 맵만
> 걸러 검사하도록 수정됨(DEMO/CLAUDE_JOBS 등 오탐 제거). 파일 변경 없음.

---

## ★ 최우선 원칙: 디자인을 "예쁘게" 뽑을 것 ★

사용자의 1순위 관심사는 **비주얼 퀄리티**다. "못생기면 실패"다. 아래를 반드시 지킬 것:

- **픽셀아트 스타일**: 아기자기하고 선명한 미니 2D RPG 무드(스타듀밸리/옥토패스 미니 도트 느낌). 과하게 사실적 X, **귀엽고 또렷하게**.
- **아웃라인 필수**: 캐릭터/오브젝트마다 어두운 테두리로 배경에서 확 떠 보이게.
- **치비 비율**: 머리 크게, 몸 작게. 얼굴에 또렷한 두 눈 + 볼 홍조 등 표정.
- **셰이딩 3톤 이상**: base / shadow / highlight. 밋밋한 단색 금지.
- **통일된 팔레트**: 직업/오브젝트가 제각각 튀지 않게 색 조화. 채도·명도 톤 맞추기.
- **애니메이션으로 생동감**: idle bob(이미 있음), working 2프레임 모션(팔/도구 흔들기), 가능하면 걷기.
- **각 문자맵 행의 폭을 정확히 동일**하게(안 맞으면 픽셀 어긋남 — 아래 검증 스크립트로 매번 확인).
- **자체완결 유지**: 외부 이미지/폰트/CDN/스크립트 금지. 전부 canvas로 그림. `imageSmoothingEnabled=false` 유지.
- **후퇴 금지**: 현재 캐릭터(치비 16x18 + 아웃라인 + 음영)가 최소 기준선. 여기서 **더 예쁘게**, 절대 더 단순/못생기게 X.

---

## 지금까지 되어 있는 것 (현재 `guild.html` 기준 — 이걸 토대로 확장)

- 라이브 SSE로 세션 → **길드원 캐릭터** 렌더. 완료/실패는 무대에서 숨기고 "(+N 휴식)" 카운트.
- **스프라이트 = 문자맵 + 팔레트**: `drawSprite(g, sx, sy, px, map, pal)`.
  - `MAGE`, `WARRIOR` 두 베이스맵(현재 **16폭 x 18행**, 아웃라인/음영/눈/볼홍조 포함).
  - 팔레트 키: `O`아웃라인 `H`/`h`주색+그늘 `F`/`f`피부+그늘 `b`볼홍조 `E`눈 `e`투구눈광 `R`/`r`로브·갑옷+그늘 `W`하이라이트 `P`깃털 `V`바이저 `B`부츠.
  - `paletteFor(cls, dim)`가 클래스 색으로 팔레트 생성(dim=idle이면 desaturate).
- **직업 11종** `const CLASSES`: (Claude계열=mage) 견습마법사/마법사/대마법사/소환사/현자, (Codex계열=warrior) 견습병/전사/기사/대장장이/광전사, 공통 전령. 각 `{base, main, robe, weapon, aura?, line}`.
  - `autoClass(a)`가 데이터로 직업 자동 배정(서브에이전트→소환사/기사, 핸드오프중→전령, PR/브랜치→현자/대장장이, 토큰 많으면 상위직).
  - **전직**: 캐릭터 클릭 → `#jobmenu`에서 직업 선택 → `jobOverride[id]` 저장(localStorage) + "전직!" 이펙트. `hitTest(cx,cy)`로 클릭 판정.
- **레벨**: `levelOf(a)`=토큰+서브에이전트+PR. 머리 위 `Lv` 배지, 오르면 "LEVEL UP!" 파티클(`burst`, `fx[]`).
- `drawWeapon(g,sx,sy,PX,SPR_W,kind,hot,t)`: staff/orb/book/scroll/sword/hammer/axe.
- **폴더별 "방"**(`rooms`) + **퀘스트 보드**(왼쪽, working/needs_input 목록).
- 이름표/직업명/마나(토큰)바, 상태 말풍선(작업중 `···` / 입력필요 `!` / 대기 `z`).
- 상수: `PX=4`, `SPR_W=16*PX`, `SPR_H=18*PX`, `CELL_W=96`, `CELL_H=122`.
- 색 유틸: `darken`, `lighten`, `desat`, `hexToRgb`, `rgbToHex`.
- SSE 끊기면 `DEMO` 배열로 폴백(오프라인에서도 데모 보임).

### SSE agent 데이터 필드 (이걸로 연출)
`id, name, kind('claude'|'codex'|'background'|'subagent'|...), state('working'|'needs_input'|'idle'|'done'|'failed'|'unknown'), project(cwd), lastMessage, tokens{context,window}, subagents[], branch, pr, aiHandoff(문자열: 'IDLE'|'READY_FOR_CODEX'|'CODEX_RUNNING'|'CODEX_DONE'|'CODEX_FAILED'|'READY_FOR_CODEX_REVIEW' 등), aiWatched(bool)`

---

## Phase A — 캐릭터 아트 한 단계 더 (디자인 최우선)

1. `MAGE`/`WARRIOR` 베이스맵을 **더 예쁘게** 다듬기: 얼굴 표정(눈 하이라이트/미소), 모자 챙 그림자, 옷 주름 하이라이트, 손/소매 디테일. 필요하면 크기 살짝 키워도 됨(맵 폭 일관 + 상수 갱신 + 이름표/바 위치 재조정 `CELL_H`).
2. **직업별 실루엣 차별화 강화**(색만이 아니라 형태로): 소환사=후드+떠있는 오브, 현자=수염+책, 기사=풀플레이트+작은 방패, 대장장이=앞치마+망치, 광전사=뿔투구+도끼, 전령=어깨가방+두루마리. 베이스맵 2종 유지하되 **클래스별 accessory 픽셀을 덧그리는 함수**(예: `drawAccessory(cls,...)`) 도입 권장.
3. **working 2프레임 모션**: `t` 기반으로 팔/무기 1~2px 흔들기 또는 발 교차. 걷기(idle시 살짝 좌우 이동)도 여유되면.
4. 상위 직업은 **오라/파티클**로 격 표현(대마법사=반짝임, 광전사=불티 등).

## Phase B — 몬스터 전투 (working = 전투중)

데이터 매핑:
- `state==='working'` → 캐릭터 앞에 **몬스터** 1마리. 캐릭터 공격 모션 + 몬스터 피격 흔들림.
- `state==='needs_input'` → **위기**(몬스터에 밀림, "도와줘!" 말풍선, 발밑 링 주황).
- `state` 전환 done → **몬스터 격파** 연출(퍼프 파티클 + "+XP"/코인, 캐릭터 승리 포즈 잠깐).
- failed → **패배**(캐릭터 넉다운/누움).
구현:
- 몬스터 스프라이트 **2~3종**(슬라임/박쥐/벌레=bug) 문자맵으로. id 해시로 종류 고정.
- 몬스터 HP바: 실제 진행 못 재니 경과시간 기반 연출 or 데코.
- HUD/사이드에 **"오늘 처치한 몬스터 N"**(=완료 세션 수) 카운터.
- 슬라임 예시(12x8, 다듬어 사용): `S`몸 `s`그늘 `E`눈 `O`아웃라인
  ```
  ....OOOO....
  ..OOSSSSOO..
  .OSSSSSSSSO.
  OSSEESSEESSO
  OSSSSSSSSSSO
  OSSssssssSSO
  .OSSSSSSSSO.
  ..OOOOOOOO..
  ```

## Phase C — 마을 건물 + 클릭해서 Claude/Codex 새 세션

길드 홀을 **마을**로: 건물 배치 + 클릭 상호작용.
- 건물: **길드원 집**(프로젝트/폴더별), **대장간**(Codex 추천), **마법탑**(Claude 추천), **무기점**(선택), 기존 퀘스트 보드.
- 건물 픽셀아트(문자맵)로 그리기. 문 앞 간판/불빛 등 디테일.
- **건물 클릭 → 메뉴 "새 세션 시작: 🧙 Claude / ⚔ Codex"** → 실제 새 터미널 세션 시작:
  - 재사용 API(**서버 변경 불필요, Codex 확인됨**): `POST http://127.0.0.1:7317/api/open-terminal`, body `{ cwd, command: 'claude' | 'codex', prefer? }` (fetch, 로컬 CORS 허용).
  - 대장간=command 'codex', 마법탑=command 'claude', 집=그 폴더 cwd로 선택형.
  - 성공 시 잠시 후(폴링 2.5s) 새 길드원이 SSE로 등장. "새 길드원 합류!" 토스트.
- 주의: guild.html 자체완결 유지. open-terminal 실패해도 graceful.

## Phase D — Claude ↔ Codex "대화" 시각화 (핸드오프 연출) ★사용자 신규 요청

Claude와 Codex가 협업(핸드오프)할 때 **두 캐릭터가 대화하는 모습**을 보여준다.
- 트리거: 프로젝트의 `aiHandoff` 상태가 협업 단계일 때. 같은 project의 **Claude 마법사 + Codex 전사**를 서로 마주 보게 배치(둘 중 하나가 라이브 세션이 아니면 NPC 스탠드인으로 그림).
- **말풍선 대화**를 단계별로 순환 표시(2~3초마다 번갈아):
  - `READY_FOR_CODEX`: 🧙Claude "이거 맡아줘 📜" → ⚔Codex "접수! 갈게 ⚔" (Claude가 **두루마리를 건네는** 애니메이션)
  - `CODEX_RUNNING`: ⚔Codex "구현 중… 🔨" (Codex 망치질 모션, Claude "화이팅!" 지켜봄)
  - `CODEX_DONE`: ⚔Codex "완료! ✅" → 🧙Claude "확인할게 👀" (Codex가 결과물 **돌려주는** 모션)
  - `CODEX_FAILED`: ⚔Codex "막혔어 😵" → 🧙Claude "같이 보자 🤝"
  - `READY_FOR_CODEX_REVIEW`: 🧙Claude "리뷰 부탁 👀" → ⚔Codex "볼게"
- 연출: 두 캐릭터 사이에 **두루마리(scroll) 아이콘이 오가는** 파티클, 말풍선(둥근 사각형 + 꼬리), 하트/느낌표 이모트.
- (선택) 하단에 작은 **"길드 회의록"** 로그 패널: 최근 핸드오프 대화 3~4줄 텍스트로.
- 데이터: 각 agent의 `aiHandoff`로 단계 판정. 프로젝트별로 그룹핑해 한 쌍(또는 그 프로젝트의 대표 Claude/Codex)만 연출.

---

## Verification

```bash
# dev 서버 (없으면 띄우기)
lsof -nP -iTCP:7173 -sTCP:LISTEN | head -1 || (cd /Users/heejeongjeong/dev/cc-dashboard && nohup pnpm dev >/tmp/cc-dash-dev.log 2>&1 </dev/null & disown)

# 스프라이트 문자맵(투명칸 '.' 포함, 6행 이상)만 걸러서 각 줄 폭 일관성 검증
#  → DEMO/CLAUDE_JOBS/CODEX_JOBS 같은 비-스프라이트 배열은 자동 제외(오탐 방지)
node -e '
const fs=require("fs");const s=fs.readFileSync("guild.html","utf8");
for(const m of s.matchAll(/const ([A-Z_]+) = \[([\s\S]*?)\];/g)){
  const rows=[...m[2].matchAll(/[\x27"]([^\x27"]*)[\x27"]/g)].map(x=>x[1]);
  const isSprite = rows.length>=6 && rows.every(r=>/^[.\w]+$/.test(r) && r.includes("."));
  if(!isSprite) continue;
  const w=new Set(rows.map(r=>r.length));
  console.log(m[1],"rows",rows.length,"widths",[...w], w.size===1?"OK":"!! 폭 불일치");
}
'
```
그 후 브라우저에서 `http://127.0.0.1:7173/guild.html` 열어 **콘솔 에러 0**, 캐릭터/몬스터/건물이 **또렷하고 예쁜지**, 직업/상태/핸드오프 대화가 구분되는지 **육안 확인**. 데모 모드(SSE 끊김)에서도 예쁘게 보일 것.

## Risks / 주의

- **문자맵 각 행 폭 동일** (가장 흔한 버그). 검증 스크립트 필수.
- **자체완결**: 외부 리소스 금지, 전부 canvas.
- `imageSmoothingEnabled=false` 유지.
- **`guild.html` 이 파일만** 수정. `web/src/**`·서버 건드리지 말 것.
- open-terminal 등 백엔드 호출은 실패해도 graceful(에러로 화면 안 깨지게).
- **커밋/푸시 금지** — 변경만 남겨두면 Claude가 브라우저로 육안 리뷰 후 처리.
- 클릭 판정은 렌더 루프의 hover가 아니라 `hitTest(cx,cy)`로(타이밍 이슈 방지) — 건물 클릭도 동일 패턴.

## 완료 후

`Status: CODEX_DONE`로 바꾸고, 구현한 Phase(A~D)·추가한 스프라이트/함수·남은 리스크를 아래에 적을 것. 그다음 Claude가 육안 리뷰 → 사용자 보고 → 필요시 다음 라운드.

## Codex Watch Result

- Status: CODEX_DONE
- Finished: 2026-07-09 20:10:27

```text
tus || status === 'IDLE') return null;
  return HANDOFF_STATUS[status] ?? { label: status, cls: 'idle' };
}
// 자동화 적용 여부: .ai/HANDOFF.md 있으면(=aiHandoff 값 존재) 적용된 것
function isAutomationApplied(a: Agent): boolean {
  return a.aiHandoff != null;
}
const applying = ref<Set<string>>(new Set());
// 미적용 프로젝트에 .ai/ 스캐폴드 생성 → ai-watch 인식 대상이 됨
async function applyAutomation(a: Agent) {
  if (!a.project) return flash('폴더를 알 수 없어 적용할 수 없어요');
  if (applying.value.has(a.project)) return;
  applying.value = new Set(applying.value).add(a.project);
  flash('자동화 적용 중…');
  const r = await applyAiWatch(a.project);
  const next = new Set(applying.value);
  next.delete(a.project);
  applying.value = next;
  if (r.ok) {
    flash(r.created.length ? `자동화 적용됨 · ${r.created.length}개 파일 생성 🤖` : '이미 적용되어 있어요');
  } else {
    flash(r.message ?? '적용 실패');
  }
}
// 적용 취소: 폴더 자신의 .ai/HANDOFF.md 제거 → 감시 대상에서 제외
async function unapplyAutomation(a: Agent) {
  if (!a.project) return;
  if (
    !window.confirm(
      `'${shortPath(a.project)}' 의 ai-watch 적용을 취소할까요?\n.ai/HANDOFF.md 를 삭제해 감시 대상에서 뺍니다. (나머지 .ai 문서는 유지)`
    )
  )
    return;
  flash('적용 취소 중…');
  const r = await unapplyAiWatch(a.project);
  if (r.ok) flash('적용 취소됨 — 이제 감시 안 함');
  else if (r.inheritedFrom) flash(`상위 폴더(${shortPath(r.inheritedFrom)})에서 적용됨 — 거기서 취소하세요`);
  else flash(r.message ?? '취소 실패');
}
// 카드별 '더보기(⋯)' 메뉴 열림 상태 (한 번에 하나). 메뉴는 body로 Teleport해서
// 카드 리렌더(경과시간 1초 갱신)에 영향받지 않게 → 깜빡임 방지. 위치는 버튼 기준 고정.
const menuOpen = ref<string | null>(null);
const menuPos = ref<{ top: number; left: number }>({ top: 0, left: 0 });
const menuAgent = computed(() => agents.value.find((x) => x.id === menuOpen.value) ?? null);
function toggleMenu(a: Agent, e: MouseEvent) {
  if (menuOpen.value === a.id) {
    menuOpen.value = null;
    return;
  }
  const r = (e.currentTarget as HTMLElement).getBoundingClientRect();
  // 화면 오른쪽 끝 넘지 않게 살짝 보정 (메뉴 폭 ~200px)
  menuPos.value = { top: Math.round(r.bottom + 4), left: Math.round(Math.min(r.left, window.innerWidth - 210)) };
  menuOpen.value = a.id;
}
function runMenu(fn: () => void) {
  menuOpen.value = null;
  fn();
}
// 현재 세션들 중 자동화 미적용 프로젝트(중복 제거)
const unappliedProjects = computed(() => {
  const set = new Set<string>();
  for (const a of agents.value) {
    if (a.project && !isAutomationApplied(a)) set.add(a.project);
  }
  return [...set];
});
// 미적용 프로젝트 일괄 적용 (여러 폴더에 쓰므로 확인창)
async function applyAllAutomation() {
  const projects = unappliedProjects.value;
  if (!projects.length) return;
  const list = projects.map((p) => '· ' + shortPath(p)).join('\n');
  if (
    !window.confirm(`자동화 미적용 프로젝트 ${projects.length}개에 .ai/ 스캐폴드를 만듭니다:\n\n${list}\n\n계속할까요?`)
  )
    return;
  flash(`${projects.length}개 프로젝트 적용 중…`);
  let created = 0;
  let failed = 0;
  for (const p of projects) {
    const r = await applyAiWatch(p);
    if (r.ok) created += r.created.length;
    else failed++;
  }
  flash(
    failed ? `적용 완료 (실패 ${failed}) · 파일 ${created}개 생성` : `${projects.length}개 프로젝트 자동화 적용됨 🤖`
  );
}
// ⇄ Codex에 넘기기: 살아있는 Claude 세션이면 그 세션이 핸드오프를 쓰게 주입,
// 아니면 서버가 HANDOFF.md를 READY_FOR_CODEX 로 표시 → ai-watch가 Codex 실행.
async function handoffToCodex(a: Agent) {
  if (!a.project) return flash('폴더를 알 수 없어요');
  if (!isAutomationApplied(a)) return flash('먼저 자동화 적용이 필요해요 (🤖 자동화 적용)');
  if (
    !window.confirm(
      `'${a.name}' 작업을 Codex에 넘길까요?\nHANDOFF.md 를 READY_FOR_CODEX 로 표시하면 ai-watch가 Codex를 실행합니다.`
    )
  )
    return;
  const injectable = a.kind !== 'codex' && !!a.pid && a.state !== 'working';
  if (injectable) {
    const prompt =
      '지금까지 진행한 작업을 .ai/HANDOFF.md 에 정리해줘: 현재 목표, 완료한 것, 남은 일(Codex TODO), 변경/관련 파일, 검증 명령, 주의사항. 다 쓴 뒤 파일의 상태 줄을 정확히 "Status: READY_FOR_CODEX" 로 바꿔줘. 문서만 작성하고 구현은 하지 마.';
    const r = await sendInput(a.pid!, { text: prompt, enter: true });
    flash(r.ok ? 'Claude가 핸드오프 작성 중 → 완료되면 Codex 자동 실행 ⇄' : (r.message ?? '전송 실패'));
  } else {
    const r = await readyForCodex(a.project);
    flash(r.ok ? 'HANDOFF.md → READY_FOR_CODEX · Codex 실행 대기 ⇄' : (r.message ?? '실패'));
  }
}

// localStorage에 저장/복원되는 ref
function persisted<T>(key: string, def: T) {
  const raw = localStorage.getItem(key);
  let init = def;
  if (raw != null) {
    try {
      init = JSON.parse(raw) as T;
    } catch {
      /* 무시 */
    }
  }
  const r = ref<T>(init);
  watch(r, (v) => localStorage.setItem(key, JSON.stringify(v)));
  return r;
}

// 다크 모드
const theme = persisted<'light' | 'dark'>('cc.theme', 'light');
function applyTheme() {
  document.documentElement.classList.toggle('dark', theme.value === 'dark');
}
watch(theme, applyTheme);
applyTheme();
function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark';
}

// 핀(즐겨찾기) — 상단 고정
const pinned = ref<Set<string>>(new Set(JSON.parse(localStorage.getItem('cc.pinned') ?? '[]')));
function isPinned(a: Agent): boolean {
  return pinned.value.has(a.id);
}
function togglePin(a: Agent) {
  const s = new Set(pinned.value);
  if (s.has(a.id)) s.delete(a.id);
  else s.add(a.id);
  pinned.value = s;
  localStorage.setItem('cc.pinned', JSON.stringify([...s]));
}

// --- 키보드 단축키 ---
const searchEl = ref<HTMLInputElement | null>(null);
function onKey(e: KeyboardEvent) {
  if ((e.metaKey || e.ctrlKey) && e.key.toLowerCase() === 'k') {
    e.preventDefault();
    searchEl.value?.focus();
    return;
  }
  if (e.key === 'Escape') {
    if (menuOpen.value) return void (menuOpen.value = null);
    if (diffItem.value) return closeDiff();
    if (showTerm.value) return closeTerminal();
    if (selected.value) return close();
    return;

exec
/bin/zsh -lc 'git show --stat --oneline --decorate -5' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
9ac5421 (HEAD -> main, origin/main, origin/HEAD) 길드 스펙 핸드오프 정리: IDLE 대기 + Codex 사전검토 반영(검증 스크립트 오탐 수정)
 .ai/HANDOFF.md | 359 ++-------------------------------------------------------
 1 file changed, 10 insertions(+), 349 deletions(-)
a539256 길드 뷰 완성 스펙 핸드오프 작성 (Codex용, Status: IDLE 대기)
 .ai/HANDOFF.md | 508 +++++++++++++++++++++++++++++++++++++++++++++++++++++----
 1 file changed, 479 insertions(+), 29 deletions(-)
21d8611 길드 캐릭터 아트 리디자인: 치비+아웃라인+음영으로 예쁘게
 guild.html | 68 +++++++++++++++++++++++++++++++++++++++------------
 1 file changed, 52 insertions(+), 16 deletions(-)
7d5f5c4 길드 뷰: 직업 다양화 + 전직 + 레벨 시스템
 guild.html | 529 +++++++++++++++++++++++++++++++++++---------------
 1 file changed, 372 insertions(+), 157 deletions(-)
66ca560 길드 홀 픽셀아트 뷰 프로토타입 (guild.html)
 guild.html | 525 ++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 525 insertions(+)

codex
The recent commit history in the repo has advanced beyond the watcher snapshot: `HEAD` is now `9ac5421`, and `.ai/HANDOFF.md` remains the only dirty file. I’m running the handoff’s read-only sprite-width check and a final status check; build/test commands would write outputs and are not appropriate under this read-only profile.
exec
/bin/zsh -lc 'git status --short' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
exec
/bin/zsh -lc 'git rev-parse --short HEAD' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
9ac5421

exec
/bin/zsh -lc 'git diff --stat && git diff --name-only' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
exec
/bin/zsh -lc "node -e '
const fs=require(\"fs\");const s=fs.readFileSync(\"guild.html\",\"utf8\");
for(const m of s.matchAll(/const ([A-Z_]+) = \\[([\\s\\S]*?)\\];/g)){
  const rows=[...m[2].matchAll(/[\\x27\"](["'^'"\\x27\"]*)[\\x27\"]/g)].map(x=>x[1]);
  const isSprite = rows.length>=6 && rows.every(r=>/"'^[.'"\\w]+"'$/.test(r) && r.includes("."));
  if(!isSprite) continue;
  const w=new Set(rows.map(r=>r.length));
  console.log(m[1],"rows",rows.length,"widths",[...w], w.size===1?"OK":"!! 폭 불일치");
}
'"'" in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
MAGE rows 18 widths [ 16 ] OK
WARRIOR rows 18 widths [ 16 ] OK

codex
The final status check now shows a clean working tree, which differs from the watcher’s initial dirty-file snapshot. I’m checking the current handoff header so the final report reflects the repository state I actually observed.
exec
/bin/zsh -lc 'git log --oneline -3 --decorate' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
9ac5421 (HEAD -> main, origin/main, origin/HEAD) 길드 스펙 핸드오프 정리: IDLE 대기 + Codex 사전검토 반영(검증 스크립트 오탐 수정)
a539256 길드 뷰 완성 스펙 핸드오프 작성 (Codex용, Status: IDLE 대기)
21d8611 길드 캐릭터 아트 리디자인: 치비+아웃라인+음영으로 예쁘게

exec
/bin/zsh -lc "sed -n '1,40p' .ai/HANDOFF.md" in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
Status: CODEX_RUNNING

<!--
  이 문서는 "준비된 스펙"이다. 실제로 Codex에 넘기려면:
  - 이 첫 줄을 `Status: READY_FOR_CODEX` 로 바꾸면 ai-watch 데몬이 자동 실행하거나,
  - 이 파일 전체를 Codex 세션에 그대로 붙여넣어도 됨.
  아직은 IDLE = 대기(자동 실행 안 함).
-->

# 길드 홀 픽셀아트 뷰 — 완성 스펙 (Codex 구현용)

대상 파일: **`guild.html` 단독** (자체완결 canvas 프로토타입).
서빙: vite `web/public` → `http://127.0.0.1:7173/guild.html`.
데이터: 대시보드 SSE `http://127.0.0.1:7317/api/agents/stream` (실시간 세션 목록).

> **사전 검토(Codex, 2026-07-09)**: 스펙 범위/실현성 확인 완료. Phase C의 `POST /api/open-terminal`은
> 이미 `{ cwd, command, prefer }`를 지원 → **서버 변경 불필요**. 아래 검증 스크립트는 스프라이트 맵만
> 걸러 검사하도록 수정됨(DEMO/CLAUDE_JOBS 등 오탐 제거). 파일 변경 없음.

---

## ★ 최우선 원칙: 디자인을 "예쁘게" 뽑을 것 ★

사용자의 1순위 관심사는 **비주얼 퀄리티**다. "못생기면 실패"다. 아래를 반드시 지킬 것:

- **픽셀아트 스타일**: 아기자기하고 선명한 미니 2D RPG 무드(스타듀밸리/옥토패스 미니 도트 느낌). 과하게 사실적 X, **귀엽고 또렷하게**.
- **아웃라인 필수**: 캐릭터/오브젝트마다 어두운 테두리로 배경에서 확 떠 보이게.
- **치비 비율**: 머리 크게, 몸 작게. 얼굴에 또렷한 두 눈 + 볼 홍조 등 표정.
- **셰이딩 3톤 이상**: base / shadow / highlight. 밋밋한 단색 금지.
- **통일된 팔레트**: 직업/오브젝트가 제각각 튀지 않게 색 조화. 채도·명도 톤 맞추기.
- **애니메이션으로 생동감**: idle bob(이미 있음), working 2프레임 모션(팔/도구 흔들기), 가능하면 걷기.
- **각 문자맵 행의 폭을 정확히 동일**하게(안 맞으면 픽셀 어긋남 — 아래 검증 스크립트로 매번 확인).
- **자체완결 유지**: 외부 이미지/폰트/CDN/스크립트 금지. 전부 canvas로 그림. `imageSmoothingEnabled=false` 유지.
- **후퇴 금지**: 현재 캐릭터(치비 16x18 + 아웃라인 + 음영)가 최소 기준선. 여기서 **더 예쁘게**, 절대 더 단순/못생기게 X.

---

## 지금까지 되어 있는 것 (현재 `guild.html` 기준 — 이걸 토대로 확장)

- 라이브 SSE로 세션 → **길드원 캐릭터** 렌더. 완료/실패는 무대에서 숨기고 "(+N 휴식)" 카운트.

exec
/bin/zsh -lc 'git status --short --branch' in /Users/heejeongjeong/dev/cc-dashboard
 succeeded in 0ms:
## main...origin/main

codex
Blocked before implementation: this session is running with a read-only filesystem and approval disabled, so I cannot edit `guild.html`, `.ai/HANDOFF.md`, or `.ai/TASKS.md`. A minimal `apply_patch` write attempt was rejected by the sandbox.

Current repo state:
- Branch: `main`
- HEAD: `9ac5421`
- Working tree: clean
- `.ai/HANDOFF.md` still starts with `Status: CODEX_RUNNING`

Inspected:
- `AGENTS.md`
- `.ai/PROJECT.md`
- `.ai/RULES.md`
- `.ai/TASKS.md`
- `.ai/HANDOFF.md`
- `guild.html`
- Recent related files: `web/src/App.vue`, `web/src/utils/markdown.ts`, `web/src/components/TerminalView.vue`
- Recent commit history

Verification run:
```text
MAGE rows 18 widths [ 16 ] OK
WARRIOR rows 18 widths [ 16 ] OK
```

I did not run build/test because this read-only profile can’t safely create build artifacts or update task/handoff files. No files were changed.
hook: Stop
hook: Stop Completed
tokens used
59,505
Blocked before implementation: this session is running with a read-only filesystem and approval disabled, so I cannot edit `guild.html`, `.ai/HANDOFF.md`, or `.ai/TASKS.md`. A minimal `apply_patch` write attempt was rejected by the sandbox.

Current repo state:
- Branch: `main`
- HEAD: `9ac5421`
- Working tree: clean
- `.ai/HANDOFF.md` still starts with `Status: CODEX_RUNNING`

Inspected:
- `AGENTS.md`
- `.ai/PROJECT.md`
- `.ai/RULES.md`
- `.ai/TASKS.md`
- `.ai/HANDOFF.md`
- `guild.html`
- Recent related files: `web/src/App.vue`, `web/src/utils/markdown.ts`, `web/src/components/TerminalView.vue`
- Recent commit history

Verification run:
```text
MAGE rows 18 widths [ 16 ] OK
WARRIOR rows 18 widths [ 16 ] OK
```

I did not run build/test because this read-only profile can’t safely create build artifacts or update task/handoff files. No files were changed.
```
