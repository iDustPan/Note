
# 如何让Mac终端添加代码高亮

### 修改配置文件

`cd ~` 

`ls -lah` 确认是否有.bash_profile这个隐藏文件

` vi .bash_profile` 打开配置文件

光标定位到最后一行，输入`i`进入插入模式，把下面代码粘贴到最后一行：

```
# enables colorin the terminal bash shell export
export CLICOLOR=1

# sets up thecolor scheme for list export
export LSCOLORS=gxfxcxdxbxegedabagacad

# sets up theprompt color (currently a green similar to linux terminal)
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;36m\]\w\[\033[00m\]\$ '

# enables colorfor iTerm
export TERM=xterm-color
```

按`ESC` 输入`:wq` 保存退出;

`source .bash_profile` 使文件生效
