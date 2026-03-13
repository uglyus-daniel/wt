# VSCode Titlebar Color — Design Spec

## Goal

`wt new` 로 워크트리를 생성할 때 해당 워크트리의 `.vscode/settings.json`에 랜덤 상단바 색상을 자동으로 적용한다. 워크트리 간 시각적 구분을 목적으로 한다.

## Decisions

| 항목 | 결정 |
|------|------|
| 색상 선택 방식 | 큐레이션 팔레트 (16색) 중 `$RANDOM` 랜덤 선택 |
| 전경색 | 배경색 luminance 기준으로 흰색/검정 자동 결정 |
| 기존 settings.json 처리 | 머지 — `titleBar.*` 키만 추가/업데이트, 나머지 보존 |
| 구현 방식 | Bash 내 `python3` 한 줄 JSON 처리 (외부 의존성 없음) |

## Implementation

### Call Site

`cmd_new`의 `install_dependencies_in_worktree "$wt_path"` 호출 직후, `success "Worktree ready: $wt_path"` 직전에 다음을 삽입한다:

```bash
apply_vscode_titlebar_color "$wt_path"
```

### New Function: `apply_vscode_titlebar_color()`

**stdout 제약:** 이 함수의 모든 출력은 반드시 `info`/`warn`/`success` (stderr)를 사용해야 한다. stdout 에 절대 쓰면 안 된다. `cmd_new`의 stdout은 shell wrapper가 `cd` 경로로 캡처하기 때문에 오염 시 auto-cd가 깨진다.

**함수 동작 순서:**

1. Tailwind 기반 16색 팔레트 배열 정의
2. `$RANDOM % 16` 으로 색상 인덱스 선택
3. `python3`에 선택된 HEX를 넘겨서 luminance 계산, fg 색상 결정, inactive 색상 계산, JSON 머지까지 한 번에 처리
4. `python3`가 없으면 `warn` 후 조용히 건너뜀 (워크트리 생성은 계속)

### Luminance Calculation

Bash는 부동소수점을 지원하지 않으므로 luminance 계산은 `python3`에 위임한다. WCAG relative luminance:

```python
def linearize(c):
    c = c / 255.0
    return c / 12.92 if c <= 0.04045 else ((c + 0.055) / 1.055) ** 2.4

L = 0.2126 * linearize(R) + 0.7152 * linearize(G) + 0.0722 * linearize(B)
fg = "#000000" if L > 0.179 else "#ffffff"
```

**참고:** 현재 16색 팔레트는 모두 luminance > 0.179 이므로 항상 검정 전경이 선택된다. 이는 팔레트가 중간 밝기로 선정됐기 때문이며, luminance 분기는 향후 어두운 색상 추가 시를 위한 future-proofing이다.

### Inactive Background Calculation

`inactiveBackground` = active 색상의 R/G/B 각 채널을 10% 어둡게:

```python
def darken(hex_color, factor=0.9):
    r = int(hex_color[1:3], 16)
    g = int(hex_color[3:5], 16)
    b = int(hex_color[5:7], 16)
    return "#{:02x}{:02x}{:02x}".format(int(r*factor), int(g*factor), int(b*factor))
```

### VSCode Settings Keys

```json
"workbench.colorCustomizations": {
  "titleBar.activeBackground": "<selected-color>",
  "titleBar.activeForeground": "<auto-fg>",
  "titleBar.inactiveBackground": "<darkened-10pct>",
  "titleBar.inactiveForeground": "<auto-fg>aa"
}
```

### JSON Merge Logic (python3)

```python
import json, sys, os

settings_path, active_bg, active_fg, inactive_bg, inactive_fg = sys.argv[1:]

# 파일이 있으면 읽기, 없으면 빈 객체
if os.path.exists(settings_path):
    try:
        with open(settings_path) as f:
            settings = json.load(f)
    except (json.JSONDecodeError, ValueError):
        # JSONC (주석 포함) 또는 손상된 파일 — 건드리지 않음
        print("JSONC_OR_INVALID", end="")
        sys.exit(0)
else:
    settings = {}

if "workbench.colorCustomizations" not in settings:
    settings["workbench.colorCustomizations"] = {}

settings["workbench.colorCustomizations"].update({
    "titleBar.activeBackground": active_bg,
    "titleBar.activeForeground": active_fg,
    "titleBar.inactiveBackground": inactive_bg,
    "titleBar.inactiveForeground": inactive_fg,
})

# 디렉터리 생성 후 쓰기
os.makedirs(os.path.dirname(settings_path), exist_ok=True)
with open(settings_path, "w") as f:
    json.dump(settings, f, indent=2)
    f.write("\n")
```

Bash 쪽에서 `python3`의 stdout을 캡처해 `"JSONC_OR_INVALID"` 가 반환되면 `warn "Could not parse .vscode/settings.json (may contain comments or be malformed). Skipping titlebar color."` 를 출력하고 계속 진행한다.

### File Handling

- `.vscode/` 디렉터리가 없으면 python3 스크립트 내 `os.makedirs`로 자동 생성
- `settings.json`이 없으면 빈 객체 `{}`에서 시작
- **JSONC / 손상 파일**: `json.JSONDecodeError` 발생 시 파일을 수정하지 않고 경고만 출력
- **비원자적 쓰기**: `open(p, 'w')` 는 원자적이지 않다. 쓰기 중 프로세스 중단 시 파일이 비워질 수 있다. 이는 개발용 settings 파일이므로 허용 가능한 리스크로 문서화하고 수용한다.
- `python3`가 없는 환경: 경고만 출력하고 계속 진행

### Color Palette (16색)

Tailwind CSS v3 500단계 기반 (중간 밝기, 채도 높음):

```
#3b82f6  blue-500
#10b981  emerald-500
#f59e0b  amber-500
#ef4444  red-500
#8b5cf6  violet-500
#ec4899  pink-500
#06b6d4  cyan-500
#84cc16  lime-500
#f97316  orange-500
#14b8a6  teal-500
#6366f1  indigo-500
#a855f7  purple-500
#22c55e  green-500
#eab308  yellow-500
#0ea5e9  sky-500
#f43f5e  rose-500
```

## Out of Scope

- 기존 워크트리에 소급 적용하는 별도 명령 (`wt colorize`)
- `.wtconfig`를 통한 팔레트 커스터마이징
- 색상 중복 방지 (다른 워크트리와 같은 색상이 될 수 있음)
