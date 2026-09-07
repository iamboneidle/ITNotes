___
___
Удобная штучка, чтобы хрнаить в быстром доступе в терминале всякие команды, которые не охото запоминать.

В файл `~/.zshrc` добавить:
```zsh
cheat() { source ~/Scripts/cheat.sh "$@" }
```
В файл`~/Scripts/cheat.sh` добавить:
```zsh
#!/bin/zsh

local green='\033[0;32m'
local reset='\033[0m'

local CHEAT_FILE=~/.cheatsheet
if [[ $# -eq 0 ]]; then
    echo -n "$green"
    cat "$CHEAT_FILE"
    echo -n "$reset"
    return
fi
echo -n "$green"
awk -v keyword="$1" '
    $0 ~ "^- " keyword " -$" { found=1; print; next }
    found && /^- [a-zA-Z0-9]+ -$/ { found=0 }
    found { print }
' "$CHEAT_FILE"
echo -n "$reset"
```
Также создать файл `~/.cheatsheet` следующего формата:
```
- linux -
ls -lah

- awk -
awk '{print $1}'

- git -
# Commit hard reset
g reset --hard HEAD~
# Commit safe reset
g reset HEAD~1

- code -
# Format JSON
Shift + Option + F
# Decode JWT token
Cmd + Shift + D

- k8s -
kubectl get po
```
