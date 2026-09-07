___
___
Данная функция нужна для того, чтобы удобно на время исключать файлы из видимости `git` без необходимости редактирования `.gitignore`.

В файл `~/.zshrc` добавить:
```zsh
alias gr=gitrm
gitrm() { source ~/Scripts/gitrm.sh "$@" }
```

В файл `~/Scripts/gitrm.sh` добавить:
```zsh
#!/bin/zsh

local red='\033[0;31m'
local green='\033[0;32m'
local yellow='\033[0;33m'
local reset='\033[0m'

if [[ "$1" == "-h" || "$1" == "-help" || "$1" == "--help" ]]; then
cat <<EOF
usage: gitrm

Working with files
    gitrm <path_to_file>         exclude file
    gitrm -d <path_to_file>      include file back
Other
    gitrm -h | -help | --help    show help window
    gitrm -s                     show excluded files

! Prefer using this command from root directory of your git repo.
EOF
    return 0
fi

local git_dir
git_dir=$(git rev-parse --git-dir 2>/dev/null)
if [[ -z "$git_dir" ]]; then
    echo "${red}gitrm: not a git repository${reset}"
    return 1
fi

local exclude_file="$git_dir/info/exclude"

local repo_root
repo_root=$(git rev-parse --show-toplevel 2>/dev/null)
repo_root=${repo_root:A}

if [[ "$1" == "-s" ]]; then
    if [[ ! -s "$exclude_file" ]]; then
        echo "${yellow}gitrm: .git/info/exclude is empty${reset}"
    else
        echo "${green}gitrm rules in $exclude_file:${reset}"
        awk 'NF && $0 !~ /^[[:space:]]*#/ {printf "%6d\t%s\n", NR, $0}' "$exclude_file"
    fi
    return 0
fi

if [[ "$1" == "-d" ]]; then
    local target_path="$2"
    if [[ -z "$target_path" ]]; then
        echo "${red}gitrm: usage: gitrm -d <path_to_file>${reset}"
        return 1
    fi

    if [[ ! -f "$exclude_file" ]]; then
        echo "${yellow}$target_path is not ignored by gitrm now${reset}"
        return 0
    fi

    local line_num
    line_num=$(grep -nxF "$target_path" "$exclude_file" | head -n1 | cut -d: -f1)

    if [[ -n "$line_num" ]]; then
        sed -i.bak "${line_num}d" "$exclude_file" && rm -f "${exclude_file}.bak"
        echo "${green}removed $target_path from gitrm${reset}"
    else
        echo "${yellow}$target_path is not ignored by gitrm now${reset}"
    fi
    return 0
fi

local target_path="$1"
if [[ -z "$target_path" ]]; then
    echo "${red}gitrm: usage: gitrm <path_to_file> — exclude file${reset}"
    echo "${red}gitrm: usage: gitrm -d <path_to_file> — include file back${reset}"
    return 1
fi

local absolute_target="${target_path:A}"
if [[ "$absolute_target" == "$repo_root"/* ]]; then
    target_path="${absolute_target#$repo_root/}"
elif [[ "$absolute_target" == "$repo_root" ]]; then
    target_path="."
else
    echo "${red}gitrm: path is outside the repository${reset}"
    return 1
fi

mkdir -p "$(dirname "$exclude_file")"
[[ -f "$exclude_file" ]] || touch "$exclude_file"

local line_num
line_num=$(grep -nxF "$target_path" "$exclude_file" | head -n1 | cut -d: -f1)

if [[ -n "$line_num" ]]; then
    echo "${yellow}already ignored by gitrm, see .git/info/exclude: L$line_num${reset}"
else
    echo "$target_path" >> "$exclude_file"
    local new_line_num
    new_line_num=$(wc -l < "$exclude_file" | tr -d ' ')
    echo "${green}gitrm will ignore $target_path, see .git/info/exclude: L$new_line_num${reset}"
fi
```
