# Hugo博客安装


<!--more-->

**部署hugo博客**

[TOC]

# 1.准备环境

安装依赖的工具：

- [Git](http://git-scm.com/)
- [Mercurial](http://mercurial.selenic.com/)
- [Go](http://golang.org/) 1.3+ (Go 1.4+ on Windows)

## 安装git

```bash
yum -y install git
git config --global user.name "xxx"
git config --global user.email "xxx"
ssh-keygen -t rsa -C "xxx"
cat /root/.ssh/id_rsa.pub
ssh git@github.com

#Hi xxx! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.      #测试ssh密钥链接
```

> 将id_rsa.pub 公钥内容复制到github上 ， **Settings > SSH and GPG keys > New SSH key**

![](http://cdn.linuxwf.com/img/20221103154959.png)

## 安装Go   

官网下载地址：https://go.dev/dl/

```bash
wget https://go.dev/dl/go1.19.3.linux-amd64.tar.gz
tar xf go1.19.3.linux-amd64.tar.gz -C /usr/local/
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile
source /etc/profile
go version
```

# 2.Hugo部署

## 1.下载Hugo

https://gohugo.io/getting-started/installing/

https://github.com/gohugoio/hugo/releases

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.105.0/hugo_0.105.0_Linux-64bit.tar.gz
mkdir /usr/local/hugo
tar xf hugo_0.105.0_Linux-64bit.tar.gz -C /usr/local/hugo/	

#解压后，文件夹中就会有个hugo绿色的文件，就是hugo的执行程序，不然hugo执行找不到路径
cp /usr/local/hugo/hugo /usr/local/bin/ 									
hugo version
```

## 2.生成站点

```bash
hugo new site path  				#path表示要安装的路径，如：/data/my_blog

#站点目录结构
├── archetypes					# markdown文章的模版,包括文章前缀注释写法
│   └── default.md	
├── config.toml					# 配置文件
├── content						# 网站内容，主要保存文章
├── data						# 生成网站可用的数据文件，可用在模版中
├── layouts						# 生成网站时可用的模版
├── public						# 通过hugo命令生成的静态文件，这是我们网站真正要发布的目录
├── static	# 静态文件，比如favicon等图标, 以及site.xml等, 将来其下的子目录和文件会在生成时候会自动复制到 public 目录中.
└── themes						# 保存可用的hugo主题
```

## 3.安装主题

- 安装完成后，hugo是没有默认的主题的，可以去主题官网上下载主题：https://gohugo.io/commands/hugo/，如：maupassant主题https://themes.gohugo.io/hyde-hyde/，每个主题进去之后都会有安装的方法和预览演示的效果。
- 直接clone主题到云服务器上的themes下

官方主题：https://themes.gohugo.io/   

**安装LoveIt主题**  https://github.com/dillonzq/LoveIt

```bash
cd /data/my_blog
git init
git clone https://github.com/dillonzq/LoveIt.git themes/LoveIt
cp -r themes/LoveIt/exampleSite/* ./
cp -r themes/LoveIt/i18n ./
```

首先进入`myblog\themes\LoveIt\exampleSite`目录下，复制`config.toml`文件，将其粘贴至`myblog`下以替换原有的`config.toml`文件，使用notepad++或者其他文本编辑器打开`myblog`目录下的`config.toml`文件，修改`baseURL`，`themeDir`以及`enableGitinfo`三行内容如下：

```bash
aseURL = "https://example.com"   ## 修改为你的github.io地址，格式为：https://yourusername.github.io
# [en, zh-cn, fr, pl, ...] determines default content language
# [en, zh-cn, fr, pl, ...] 设置默认的语言
defaultContentLanguage = "zh-cn"
# theme
# 主题
theme = "LoveIt"
# themes directory
# 主题目录
#themesDir = "../.." ## 注释掉该行

# website title
# 网站标题
title = "LoveIt"

# whether to use robots.txt
# 是否使用 robots.txt
enableRobotsTXT = true
# whether to use git commit log
# 是否使用 git 信息
enableGitInfo = false ## 由true改为false
# whether to use emoji code
# 是否使用 emoji 代码
enableEmoji = true
```

如果是 clone 了其他人的博客项目进行修改，则需要用以下命令进行初始化：

```bash
git submodule update --init --recursive
```

如果需要同步主题仓库的最新修改，需要运行以下命令：

```bash
git submodule update --remote
```

初始化主题基础配置后，我们可以在 `config.toml` 文件中进行站点细节配置，具体配置项参考各主题说明文档。

完成后，可以通过 `hugo new` 命令发布新文章。

```bash
hugo new posts/blog-test.md
```

## 4.查看效果

Hugo 会生成静态网页，我们在本地编辑调试时可以通过 `hugo server` 命令进行本地实时调试预览，无须每次都重新生成。

```bash
hugo server -D --disableFastRender
```

现在[`localhost:1313`](http://localhost:1313/)在浏览器的地址栏中输入

![](http://cdn.linuxwf.com/img/20221106005517.png)

win下可能会出现报错：

![](http://cdn.linuxwf.com/img/20221105163831.png)

运行以下命令：

```bash
git commit --allow-empty -m "first commit"
```

# 3.推送Github

生成静态页面

```bash
hugo -D
```

配置public文件夹的github

```bash
cd /data/my_blog/public
git init		#初始化
git remote add origin git@github.com:xinian888/xinian888.github.io.git 	#本地仓库与GitHub上的仓库进行关		
git add .		#把所以东西全部加进去
git commit -m "我的hugo博客第一次提交" 			#提交信息，把文件全部加到本地仓库里去
git push -u origin main   #把文件推上去

#同步本地仓库和远程仓库的代码
git pull --rebase origin main
#查看git全程仓库地址：		
git remote get-url origin
#更换远程仓库地址：
git remote set-url origin 地址
```

>报错：git push -u origin main 会报错
>
>error: src refspec main does not match any
>error: failed to push some refs to 'github.com:xinian888/xinian888.github.io.git'
>
>解决方法：git branch -M main   		#将本地master分支修改为main



# 4.配置GitHub Pages

![](http://cdn.linuxwf.com/img/20221106010144.png)

如果github pages 没有生效，可以查看 Actions

![](http://cdn.linuxwf.com/img/20221106015014.png)

