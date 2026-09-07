___
___
# Tags
#linux 
___
# Содержание
- [[#1. MacOS]]
	- [[#1.1. Необходимый софт]]
		- [[#1.1.1. iTerm]]
		- [[#1.1.2. Доп. софт]]
	- [[#1.2. .zshrc]]
	- [[#1.3. Cheat sheet]]
___
# 1. MacOS
## 1.1. Необходимый софт
### 1.1.1. iTerm
Донастройка hotkeys:
1. `Send Escape Sequence` => `Option` + `<-` = `b`
2. `Send Escape Sequence` => `Option` + `->` = `f`
3. `Split Vertically with Profile` => `Command` + `=`
### 1.1.2. Доп. софт
```shell
brew install kubectx
brew install zsh-syntax-highlighting
```
## 1.2. .zshrc
``` fold
alias ctx=kubectx
alias ns=kubens
alias tg=terragrunt
alias tf=terraform
alias k=kubectl

ll() {
    ls -lah "$@"
}

kld() {
    kubectl logs deployment/$@
}

kgp() {
    kubectl get po
}

ktp() {
    kubectl top pod
}

cheat() {
    local CHEAT_FILE=~/.cheatsheet

    if [[ $# -eq 0 ]]; then
        cat "$CHEAT_FILE"
        return
    fi

    awk -v keyword="$1" '
        $0 ~ "^- " keyword " -$" { found=1; print; next }
        found && /^- [a-zA-Z0-9]+ -$/ { found=0 }
        found { print }
    ' "$CHEAT_FILE"
}
```
## 1.3. Cheat sheet
``` fold
- linux -
# сортировка cpu/mem
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -20
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%mem | head -20
ps -o rss,args | awk '{print $1/1024 " MB", $2}' | sort -k1 -nr | head -50

- awk -
# commands

- k8s -
# commands
```

___
