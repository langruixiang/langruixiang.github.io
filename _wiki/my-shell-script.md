---
layout: wiki
title: 脚本提升效率
categories: [效率]
description: 一些shell脚本，iTerm2+zsh体验，提高工作效率
keywords: shell
---

## iterm2 tab自动变色

每个程序员日常都会登陆很多环境，联调环境，测试环境，线上环境等等。这些环境很相似，容易搞不清楚当前处在本地还是远端，线下环境还是生产环境。特别是你把线上环境当做线下环境进行了误操作，离卷铺盖卷儿回家不远了。

于是我想要的一种效果是，当我在本地时，iterm2的tab是黑色，登陆线下环境的时候，iterm2的tab自动变成绿色，登陆线上环境的时候，iterm2的tab自动变成红色，这样我就能很清楚的知道我是在本地还是线下还是线上。

![](/images/wiki/iterm_tab_color.png)

参考以下脚本：

```
#!/bin/bash
# iTerm2 tab color commands
# 修改自https://iterm2.com/documentation-escape-codes.html

if [[ -n "$ITERM_SESSION_ID" ]]; then
    tab-color() {
        echo -ne "\033]6;1;bg;red;brightness;$1\a"
        echo -ne "\033]6;1;bg;green;brightness;$2\a"
        echo -ne "\033]6;1;bg;blue;brightness;$3\a"
    }
    tab-red() { tab-color 255 0 0 }
    tab-green() { tab-color 0 255 0 }
    tab-blue() { tab-color 0 0 255 }
    tab-reset() { echo -ne "\033]6;1;bg;*;default\a" }

    function iterm2_tab_precmd() {
        #tab-reset
        tab-color 0 0 0
    }

    function iterm2_tab_preexec() {
        if [[ "$1" =~ ".*_online.*" ]]; then
            tab-color 255 0 0
        elif [[ "$1" =~ ".*_offline.*" ]]; then
            tab-color 0 255 0
        fi
    }

    autoload -U add-zsh-hook
    add-zsh-hook precmd  iterm2_tab_precmd
    add-zsh-hook preexec iterm2_tab_preexec
fi
```


原理其实就是给每个命令加了钩子函数：

* precmd在敲命令之前运行，把iterm2 tab变成黑色
* preexec在命令回车后，执行前执行，我登陆机器的命令全部写在不同的脚本中，线上的脚本名称均包含\_online字样，线下的脚本名称均包含\_offline字样，根据正则表达式将tab颜色改变，此处可以按需定制
* 最后在~/.zshrc里```source /path/to/iterm2_color.zsh```即可

## 自定义命令的参数补全

mac中下载的文件默认下载到~/Download目录下，对于一些看完就删的文件，我就留在Download目录下，定期删除。对于需要保存的文件，就需要频繁执行如下命令：

```
mv $DOWNLOAD_HOME/xxx.file .../.../.../xxx.file
```

我的使用习惯是，在目标目录下拷贝Download目录先的文件，于是我写了一个mvd脚本放在/usr/local/bin目录下，用于快速移动下载的文件，比如

```
mvd book.pdf
```

就是把Download目录下的book.pdf移动到当前目录，瞬间省去一大坨输入内容，脚本如下。脚本内容很简单，就是把我DOWNLOAD_HOME环境变量下的对应文件名移动到输入命令时的目录

```
#!/bin/bash
mv $DOWNLOAD_HOME/$1 `pwd`
```

当我屁颠屁颠的使用这个命令时，shit，不能通过tab补全，我得输入文件的全名... 那还不如我输入mv那个原始命令呢，至少能tab补全，不用输入一大坨文件名，尤其当文件名很长的时候。

于是查了一下bash下命令补全，有了以下脚本：

```
#!/bin/bash

function _mvd() {
    local cur prev opts

    COMPREPLY=()

    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    opts=`ls $DOWNLOAD_HOME`

    COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
    return 0
}

complete -F _mvd mvd
```

bash下可以使用compgen进行补全，比如(因为我默认使用zsh所以需要这样执行)

```
echo "compgen -W 'aa ab bc bd' a" | /bin/bash
```
就会输出

```
aa ab
```

Bash还有几个内置的变量用来辅助补全功能

* ```COMP_WORDS```: 类型为数组，存放当前命令行中输入的所有单词；
* ```COMP_CWORD```: 类型为整数，当前光标下输入的单词位于COMP_WORDS数组中的索引；
* ```COMPREPLY```: 类型为数组，候选的补全结果；
* ```COMP_WORDBREAKS```: 类型为字符串，表示单词之间的分隔符；
* ```COMP_LINE```: 类型为字符串，表示当前的命令行输入；

所以cur表示当前光标下的单词，而prev则对应上一个单词

opts则是以空格分割的侯选项，这里我要匹配DOWNLOAD_HOME下的文件，所以以```ls $DOWNLOAD_HOME```结果作为候选项.```complete -F _mvd mvd```即为把\_mvd函数绑定给mvd命令

默认情况下，zsh不识别complete目录需要执行

```
autoload bashcompinit
bashcompinit
```
然后在~/.zshrc里即可```source /path/to/mycomplete.sh``` 这个脚本可以完成多个自定义命令的补全脚本的绑定

## 本地git仓库跳转到远程仓库主页

有时候我们在本地仓库push一个提交之后，需要跳转到远程仓库的主页，查看一下预览效果和文档格式，尤其对于托管文档的git仓库更是如此。

如果每次push后，再打开浏览器，输入仓库地址，貌似很麻烦，能不能输入一个命令，直接打开仓库对应的主页呢，看如下脚本：

```
#!/bin/bash

cd `pwd`

url=`git remote -v | awk '{print $2}' | uniq | head -n 1`

is_github=`echo $url | grep github`
if [ -n "$is_github" ]; then
    host=`echo $url | awk -F'@' '{print $2}' | awk -F':' '{print $1}'`
    path=`echo $url | awk -F':' '{print "/"$2}'`
    open "https://$host$path"
    exit 0
fi

is_netease_gitlab=`echo $url | grep xxx.xxx.com`
if [ -n "$is_netease_gitlab" ]; then
    host=`echo $url | awk -F'@' '{print $2}' | awk -F':' '{print $1}'`
    path=`echo $url | awk -F'[0-9]*' '{print $2}'`
    open "https://$host$path"
    exit 0
fi
```

其实就是利用git remote -v提取到对应仓库的地址和路径，拼凑成仓库主页的url，再打开url。我平时主要用github和我们公司内部gitlab，这两者对应的地址构成不太一样，所以进行了一下判断，分别处理，其他站点可以类似处理。
