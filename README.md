# guild-town

Claude/Codex CLI 세션을 RPG 길드원으로 보여주는 픽셀아트 뷰 (cc-dashboard 데이터 사용).

## 실행
```bash
python3 -m http.server 8080 --bind 127.0.0.1
# 브라우저: http://localhost:8080/guild.html
```

라이브 데이터는 `cc-dashboard` 서버(`127.0.0.1:7317`)가 떠 있어야 보입니다. 서버가 없으면 데모 모드로 표시됩니다.

## 다시 내려받아서 쓰기
```bash
git clone <repo-url>
cd guild-town
python3 -m http.server 8080 --bind 127.0.0.1
```

브라우저에서 `http://localhost:8080/guild.html`을 열면 됩니다.

## 구성
- `guild.html`: 길드 타운 웹 화면
- `assets/guild-jobs.png`: 직업별 투명 픽셀 스프라이트 시트
- `assets/guild-jobs-action.png`: 직업별 공격·주문 시전 프레임
- `assets/tower-keeper.png`: 마법탑 상주 탑지기 NPC
- `assets/weapon-merchant.png`: 무기점 상주 무기상 NPC
- `assets/guild-archer.png`: 플레이어 직업용 궁수 스프라이트
- `assets/guild-rogue.png`: 플레이어 직업용 도적 스프라이트
- `SPEC.md`: 구현 스펙과 디자인 방향

외부 빌드 과정 없이 폴더 전체를 정적 파일로 서빙하면 됩니다.
