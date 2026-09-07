___
___
Милый баннер с котом, напоминающий, что макбук надо ребутать хотя бы раз в неделю.

В файл `~/.zshrc` добавить:
```zsh
banner() { source ~/Scripts/banner.sh "$@" }
```
В файл`~/Scripts/banner.sh` добавить:
```zsh
#!/bin/zsh

local R='\033[0;31m'
local P='\033[0;35m'
local C='\033[0;36m'
local Y='\033[0;33m'
local W='\033[1;37m'
local X='\033[0m'

local uptime_raw="$(uptime)"
local uptime_str="$(printf '%s\n' "$uptime_raw" | sed 's/.*up /up /' | sed 's/,.*//')"
local last_str="$(last $USER 2>/dev/null | grep -v 'still' | awk 'NR==1{print $4, $5, $6}' | sed 's/  */ /g')"
local reboot_needed=0
if [[ "$uptime_raw" =~ ([0-9]+)[[:space:]]+days? ]]; then
    local uptime_days=$match[1]
    if (( uptime_days > 7 )); then
        reboot_needed=1
    elif (( uptime_days == 7 )) && [[ "$uptime_raw" =~ days?,[[:space:]]+([0-9]+):([0-9]+) ]]; then
        local uptime_hours=$match[1]
        local uptime_minutes=$match[2]
        if (( uptime_hours > 0 || uptime_minutes > 0 )); then
            reboot_needed=1
        fi
    fi
fi

printf "${P}╔═══════════════════════════════════════╗\n"
printf "${P}║ ${W}. * . *  * .  /\___/\  * . . *  . * . ${P}║\n"
printf "${P}║ ${W}*  * . . *__ ( OwO ) )/. * . * * . .  ${P}║\n"
printf "${P}║ ${W}. *  *  * . \_|___|_ /*  . * .  *  *  ${P}║\n"
printf "${P}║ ${W}* . * .  .  [_______]  .  * . *  .  . ${P}║\n"
printf "${P}╠═══════════════════════════════════════╣\n"
printf "${P}║  ${C}  ${W}last login: %-20s${P}   ║\n" "$last_str"
printf "${P}║  ${Y}  ${W}%-32s${P}   ║\n" "$uptime_str"
if (( reboot_needed )); then
    printf "${P}║  ${R}  %-32s${P}   ║\n" "reboot needed"
fi
printf "${P}╚═══════════════════════════════════════╝\n"
printf "${X}\n"%
```