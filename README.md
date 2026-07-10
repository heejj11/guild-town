# guild-town

Claude/Codex CLI 세션을 RPG 길드원으로 보여주는 픽셀아트 뷰 (cc-dashboard 데이터 사용).

## 실행
```bash
python3 -m http.server 8080
# 브라우저: http://localhost:8080/guild.html
```

라이브 데이터는 `cc-dashboard` 서버(`127.0.0.1:7317`)가 떠 있어야 보입니다. 서버가 없으면 데모 모드로 표시됩니다.

## 다시 내려받아서 쓰기
```bash
git clone <repo-url>
cd guild-town
python3 -m http.server 8080
```

브라우저에서 `http://localhost:8080/guild.html`을 열면 됩니다.

## 구성
- `guild.html`: 단독 실행 가능한 길드 타운 웹 화면
- `SPEC.md`: 구현 스펙과 디자인 방향

외부 빌드 과정 없이 정적 HTML 파일만 서빙하면 됩니다.
