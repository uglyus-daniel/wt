# wt

여러 레포지토리에서 공통으로 쓰는 범용 Git worktree 관리 스크립트입니다.

- 레포 루트 자동 감지
- `package.json` 위치 자동 감지 (`.wtconfig`로 오버라이드 가능)
- `.env` 자동 복사/동기화
- 패키지 매니저 자동 감지 (`bun` / `pnpm` / `yarn` / `npm`)

## 설치 (git clone 기준)

```bash
cd ~
git clone https://github.com/uglyus-daniel/wt.git
cd wt
chmod +x wt
./wt install
source ~/.zshrc
```

설치 후에는 어디서든 `wt` 명령을 사용할 수 있습니다.

```bash
wt help
```

## 기본 사용법

```bash
wt new <branch> [base-branch]
wt ls
wt cd <branch>
wt dev [branch]
wt sync [branch]
wt rm <branch>
```

## `.wtconfig` (옵션)

레포 루트에 `.wtconfig`를 두면 경로를 고정할 수 있습니다.

```bash
WT_WEB_DIR="uglyus-web"
WT_BASE_DIR="$HOME/worktrees/uglyus"
WT_ENV_FILE="uglyus-web/.env"
```

## 예시 (uglyus)

```bash
cd ~/uglyus
wt new feature/test-branch dev
wt ls
wt cd feature/test-branch
wt dev
```

## 참고

- `wt cd`는 쉘 함수가 필요해서 `./wt install`이 선행되어야 합니다.
- `wt ls`는 기본적으로 `fzf` 선택을 사용하며, 텍스트 목록은 `wt ls --plain`으로 볼 수 있습니다.
- `wt rm`은 브랜치를 생략하면 목록에서 고른 뒤 삭제 확인을 한 번 더 묻습니다.
- `wt rm`, `wt cd`는 브랜치를 생략하면 `fzf` 인터랙티브 선택을 사용합니다.
