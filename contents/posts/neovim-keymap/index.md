---
title: "[Neovim] Custom Keymap 설정하기"
description: "Neovim configuration #6"
date: 2025-11-11
update: 2025-11-11
tags:
  - neovim
  - ide
series: "Neovim으로 개발환경 구축하기"
---

## 🚀 들어가며

Vim을 처음 사용해본 사람이라면 모두 한번쯤은, 습관적으로 `Ctrl+Z`로 취소를 하려다 프로세스가 일시정지되어 Vim이 꺼져버려 당황한 기억이 있을 것이다.🤦 나만 해도 교수님이 당부에 당부를 거듭하셨음에도 시험을 보다 습관적으로 `Ctrl+Z`를 누르고... 손을 달달 떨며 기억을 더듬어 `fg`로 복구한 경험이 있다 😵

이렇게 윈도우와 리눅스 및 Vim의 단축키가 다른 게 정말 불편한 요소 중 하나인데, Neovim에서 키 매핑을 통해 이 간극을 줄일 수 있다.

우리의 목표는 항상! VSCode와 가장 유사한 개발 환경을 만드는 것이므로 최대한 윈도우 및 VSCode와 이질감이 없게 커스텀을 해 보자.

## 🎯 파일 생성 및 적용

먼저 `lua/core/keymaps.lua` 파일을 생성한다. 이전 포스트처럼 파일 트리에서 `a`키로 생성하거나 터미널에서 `touch` 명령어를 사용하면 된다.

그 후 `init.lua` 파일에 생성한 `keymaps.lua` 파일을 import한다.

```lua
require 'core.keymaps'
```

## ⌨️ 키맵 옵션 설정

먼저 키맵 설정에 자주 사용할 옵션들을 변수로 정의해두면 코드가 훨씬 깔끔해진다.

```lua
local keyset = vim.keymap.set
local remap_opt = { remap = true }
local remap_silent_opt = { remap = true, silent = true }
local noremap_opt = { noremap = true }
local noremap_silent_opt = { noremap = true, silent = true }
```

- `keyset`: `vim.keymap.set`의 축약형으로 타이핑을 줄여준다.
- `noremap`: 재귀적 매핑을 방지한다. 키맵이 다른 키맵을 다시 호출하지 않도록 한다.
- `silent`: 명령어 실행 시 하단에 메시지를 표시하지 않는다.
- `remap`: 플러그인의 키맵을 재사용할 때 필요하다.

## 👔 Leader 키 설정

```lua
vim.g.mapleader = ' '
vim.g.maplocalleader = ' '

keyset({ 'n', 'v' }, '<Space>', '<Nop>', { silent = true })
```

**Leader 키**는 복잡한 명령어 조합의 시작점이 되는 키다. 윈도우가 보통 Ctrl키와 글자 키를 조합해 단축어를 지정하는 것처럼, Neovim에서는 리더 키와 다른 일반 키를 조합한다.

보통 `Space`를 많이 사용하는데, 손가락이 닿기 가장 편한 위치라 자주 쓰는 명령어들을 조합하기 좋다. `<Nop>`로 설정한 이유는 Space의 기본 동작(커서 오른쪽 이동)을 비활성화하여 Leader 키로만 사용하기 위함이다.

슬슬 지겨울 수도 있지만 😅 우리는 VSCode를 지향하니까.. 리더 키는 나중을 위해 일단 설정만 해 두었고 실제로 단축어를 설정해 두지는 않았다.

변수 순서는 '모드', '설정할 키맵', '실행할 명령', '옵션' 순이다.

## 🌐 전역 함수

### Neo-tree 토글 함수

```lua
function _G.toggle_neotree()
  local manager = require("neo-tree.sources.manager")
  local state = manager.get_state("filesystem")

  if state.winid and vim.api.nvim_win_is_valid(state.winid) then
    require('neo-tree.command').execute({ action = "close" })
  else
    require('neo-tree.command').execute({ action = "focus", source = "filesystem" })
  end
end
```

Neo-tree가 열려있으면 닫고, 닫혀있으면 열도록 하는 토글 함수다. `Ctrl+B`로 VSCode의 사이드바 토글처럼 사용할 수 있다.

## 🕹️ Normal 모드 - 편집 및 파일 관리

### 유용한 Normal 모드 키맵

```lua
keyset('n', 'dd',     '"_dd', noremap_opt)    -- 줄 삭제 (레지스터에 저장 안 함)
keyset('n', 'dt',     '<CMD>lua vim.diagnostic.config({virtual_text = not vim.diagnostic.config().virtual_text})<CR>', noremap_opt) -- LSP 진단 메시지 토글
```

`dd`를 재정의해서 삭제한 내용이 클립보드를 오염시키지 않도록 했다. Vim에서는 삭제도 복사처럼 레지스터에 저장되는데, 이게 은근 불편할 때가 많다.

또한 diagnostic이 너무 많이 표시되어 있으면 코드와 구별되지 않을 때가 있어, 이를 토글할 수 있는 `dt` 단축키를 추가했다.

### VSCode 스타일 단축키

```lua
keyset('n', '<C-Q>',  '<CMD>qa<CR>', noremap_opt)        -- 모든 창 닫기
keyset('n', '<C-W>',  '<CMD>q<CR>', noremap_opt)         -- 현재 창 닫기
keyset('n', '<C-A>',  'gg<S-V>G', noremap_opt)           -- 전체 선택
keyset('n', '<C-S>',  '<CMD>noautocmd w<CR>', noremap_opt) -- 저장 (포맷팅 없이)
keyset('n', '<C-Z>',  'u', noremap_opt)                  -- Undo
keyset('n', '<C-S-Z>', '<C-R>', noremap_opt)             -- Redo
keyset('n', '<C-X>',  'dd', noremap_opt)                 -- 현재 줄 잘라내기
keyset('n', '<C-C>',  'yy', noremap_opt)                 -- 현재 줄 복사
keyset('n', '<C-V>',  'p', noremap_opt)                  -- 붙여넣기
keyset('n', '<C-B>',  '<CMD>lua toggle_neotree()<CR>', noremap_opt) -- 파일 트리 토글
keyset('n', '<C-_>',  'gcc', remap_opt)                  -- 주석 토글 (Ctrl+/)
```

VSCode에서 넘어온 사람들이 가장 헷갈려하는 부분을 해결했다. 특히 `Ctrl+S` 저장이 먹히지 않으면 엄청 답답한데, 이제 익숙한 단축키로 편하게 작업할 수 있다.

`<C-_>`가 `Ctrl+/`인 이유는 터미널에서 `Ctrl+/`가 `Ctrl+_`로 인식되기 때문이다. 주석 토글 기능은 Comment 플러그인을 설치하면 활용할 수 있다.

### Leader 키 조합 - Telescope & Git (TBD)

```lua
keyset('n', 'ff',     '<CMD>Telescope find_files<CR>', noremap_opt)  -- 파일 검색
keyset('n', 'fg',     '<CMD>Telescope live_grep<CR>', noremap_opt)   -- 텍스트 검색
keyset('n', 'fh',     '<CMD>Telescope help_tags<CR>', noremap_opt)   -- 도움말 검색
keyset('n', 'fb',     '<CMD>Telescope buffers<CR>', noremap_opt)     -- 버퍼 목록

keyset('n', 'fs',     '<CMD>Gitsigns diffthis<CR>', noremap_opt)     -- Git diff 보기
keyset('n', 'fd',     '<CMD>Gitsigns preview_hunk<CR>', noremap_opt) -- Git hunk 미리보기
```

Telescope는 파일 찾기, 텍스트 검색 등을 할 수 있는 강력한 퍼지 파인더다. `Space + ff`만 누르면 프로젝트 전체에서 파일명으로 검색이 가능하고, `Space + fg`로 내용까지 검색할 수 있다.

Git 관련 기능도 `Space + f` 조합으로 통일해서 외우기 쉽게 만들었다.

(TBD)

## ✍️ Insert 모드 - 편집 중 단축키

```lua
keyset('i', '<C-A>',    '<ESC><ESC>gg<S-V>G', noremap_opt)     -- 전체 선택
keyset('i', '<C-S>',    '<ESC><ESC>:w<CR>', noremap_opt)       -- 저장
keyset('i', '<C-Z>',    '<ESC><ESC>ua', noremap_opt)           -- Undo
keyset('i', '<C-S-Z>',  '<ESC><ESC><C-R>a', noremap_opt)       -- Redo
keyset('i', '<C-X>',    '<ESC><ESC>dda', noremap_opt)          -- 현재 줄 잘라내기
keyset('i', '<C-C>',    '<ESC><ESC>yya', noremap_opt)          -- 현재 줄 복사
keyset('i', '<C-V>',    '<ESC><ESC>pi', noremap_opt)           -- 붙여넣기
keyset('i', '<C-B>',    '<CMD>lua toggle_neotree()<CR>', noremap_opt) -- 파일 트리
keyset('i', '<C-_>',    '<ESC><ESC>gcca', remap_opt)           -- 주석 토글
keyset('i', '<S-TAB>',  '<C-V><TAB>', noremap_opt)             -- 실제 탭 문자 입력
```

Insert 모드에서도 VSCode처럼 저장하고 복사/붙여넣기를 할 수 있다. `<ESC><ESC>`를 두 번 넣은 이유는 가끔 한 번으로는 모드 전환이 제대로 안 될 때가 있어서다.

명령 실행 후 다시 Insert 모드로 돌아가도록 끝에 `a` 또는 `i`를 붙였다.

`Shift+Tab`은 `expandtab` 옵션 때문에 스페이스로 변환되는 것을 방지하고 실제 탭 문자를 입력할 때 사용한다.

## 👁️ Visual 모드 - 블록 선택 및 편집

### 기본 단축키 수정

```lua
keyset('v', 'd',      '"_d', noremap_opt)   -- 삭제 (레지스터 저장 안 함)
keyset('v', 'c',      '"_c', noremap_opt)   -- 변경 (레지스터 저장 안 함)
```

Visual 모드에서 `d`와 `c`를 재정의해서 불필요한 레지스터 오염을 방지했다. 실제로 잘라내기를 하고 싶을 때는 `Ctrl+X`를 사용하면 된다.

```lua
keyset('v', '<BS>',     'd<ESC><ESC>i', noremap_opt)  -- 백스페이스로 삭제 후 Insert
keyset('v', '<C-A>',    '<ESC><ESC>gg<S-V>G', noremap_opt)
keyset('v', '<C-S>',    '<ESC><ESC>:w<CR>', noremap_opt)
keyset('v', '<C-Z>',    '<ESC><ESC>u', noremap_opt)
keyset('v', '<C-S-Z>',  '<ESC><ESC><C-R>', noremap_opt)
keyset('v', '<C-X>',    'd', noremap_opt)             -- 잘라내기
keyset('v', '<C-C>',    'y', noremap_opt)             -- 복사
keyset('v', '<C-V>',    'p', noremap_opt)             -- 붙여넣기
keyset('v', '<C-B>',    '<CMD>lua toggle_neotree()<CR>', noremap_opt)
keyset('v', '<C-_>',    'gc', remap_opt)              -- 주석 토글
```

`Ctrl` 키 조합은 이전과 유사하다.

## 📑 버퍼 관리 - Function 키 활용

```lua
keyset('n', '<F1>',   '<CMD>BufferGoto1<CR>', noremap_silent_opt)
keyset('n', '<F2>',   '<CMD>BufferGoto2<CR>', noremap_silent_opt)
keyset('n', '<F3>',   '<CMD>BufferGoto3<CR>', noremap_silent_opt)
keyset('n', '<F4>',   '<CMD>BufferGoto4<CR>', noremap_silent_opt)
keyset('n', '<F5>',   '<CMD>BufferGoto5<CR>', noremap_silent_opt)
keyset('n', '<F6>',   '<CMD>BufferGoto6<CR>', noremap_silent_opt)
keyset('n', '<F7>',   '<CMD>BufferGoto7<CR>', noremap_silent_opt)
keyset('n', '<F8>',   '<CMD>BufferGoto8<CR>', noremap_silent_opt)
keyset('n', '<F9>',   '<CMD>BufferGoto9<CR>', noremap_silent_opt)
keyset('n', '<F10>',  '<CMD>BufferLast<CR>', noremap_silent_opt)
```

Barbar 플러그인을 사용하면 상단에 VSCode처럼 탭이 표시된다. Function 키로 버퍼를 번호로 바로 이동할 수 있어서 여러 파일을 오갈 때 엄청 편하다.

`F10`은 마지막 버퍼로 이동하도록 설정했다.



## 🎓 기타 고급 기능 (TBD)

### Vim 명령어 축약 (Command Abbreviation)

```lua
vim.cmd([[
  ca ff Telescope find_files
  ca fg Telescope live_grep
  ca fh Telescope help_tags
  ca fb Telescope buffers
  ca fk Telescope keymaps
  ca fc Telescope commands
  ca fch Telescope command_history
  ca fsh Telescope search_history
  ca fhl Telescope highlights
  ca ww SudaWrite
]])
```

Command 모드(`:` 입력)에서 `ff`만 쳐도 `Telescope find_files`가 자동 완성된다. 긴 명령어를 짧게 줄여서 사용할 수 있어 편리하다.

`ww`는 Suda 플러그인으로 관리자 권한이 필요한 파일을 저장할 때 사용한다.

### 텍스트 검색 및 치환

```lua
vim.cmd([[
  vnoremap <F2>  y/<C-R>=escape(@",'/\')<CR><CR>N:%s/<C-R>=escape(@",'/\')<CR>/<C-R>=escape(@",'/\')<CR>/g<Left><Left>
  vnoremap <F3>  y/<C-R>=escape(@",'/\')<CR><CR>N
]])
```

Visual 모드에서:
- `F2`: 선택한 텍스트를 전체 파일에서 일괄 치환할 수 있는 명령어를 자동으로 생성
- `F3`: 선택한 텍스트를 검색

VSCode의 `Ctrl+H` (찾기 및 바꾸기) 기능과 비슷하게 만든 키맵이다.

### Telescope 커서 모드 검색

```lua
vim.cmd([[
  nnoremap <F12> :lua require("telescope.builtin").grep_string({layout_strategy='cursor',layout_config={width=0.5, height=0.45}})<CR>
  inoremap <F12> <ESC><ESC>:lua require('telescope.builtin').grep_string({layout_strategy='cursor',layout_config={width=0.5, height=0.45}})<CR>
]])
```

`F12`를 누르면 커서 아래 단어를 프로젝트 전체에서 검색한다. 함수 정의를 찾거나 변수 사용처를 추적할 때 유용하다.

## ✨ 마치며

지금까지 업로드했던 포스트 중에서 가장 다듬어지지 않았다는 느낌이 다소 든다. 아직 Neovim의 모든 기능들을 익히지 못하기도 했고, 플러그인을 정리해가면서 키맵을 계속 업데이트해야하기 때문인 것 같다.

특히 아직 포스팅하지 않은 LSP와 Telescope 설정이 완료되지 않아서 완성된 키맵 파일을 내놓기는 조금 무리가 있다. 포스팅을 이어가면서 나만의 설정을 계속 정리하고, 키맵 포스팅도 수정해 나가야겠다.