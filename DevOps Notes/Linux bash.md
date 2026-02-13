colour root
Add this at bottom of `/root/.bashrc`:
```
force_color_prompt=yes  
if [ -n "$force_color_prompt" ]; then     
PS1='\[\e]0;\u@\h: \w\a\]\ \[\033[01;31m\]\u@\h\[\033[00m\]:\ \[\033[01;34m\]\w\[\033[00m\]\$ ' 
fi
```
```
source /root/.bashrc
```