---
title: "使用github action和Firebase搭配来托管博客"
tags: ["折腾"]
date: 2020-02-22T02:16:00+08:00
draft: false
categories: "萨达姆做好了战斗准备"
---

## 絮絮叨叨
考虑到个人博客的流量不会很大(应该是除了自己基本没有( >﹏<。)),使用vps的话就太麻烦了.自己首先弄好linux环境,然后要配置nginx或者Apache之类的,之后是申请SSL证书,解析DNS...虽然步骤也不是很多,但是万一VPS就被DDOS了呢,就被hack了呢...其实不用vps最主要原因是...穷´_>`
正经说来,搭建个人博客这种静态网站的主要工作有:
- 选择~~白嫖的~~服务器,国外的服务器可以用Github Pages, Google的Firebase,Netfly etc..如果域名有备案的话,可以使用七牛的对象存储,每月10G的免费流量. 腾讯云cos新用户只免费6个月了.其实买的话也挺便宜,流量不多每个月也就几块钱.但我懒得备案了就用了Firebase,国内访问似乎还行.
- 然后就是把域名解析到服务器上.现在用的是Cloudflare管理dns解析.现在Github Pages支持HTTPS了,Firebase也行.直接根据提示解析过去就行. 假如托管在VPS上,可以自己申请Let'sEncrypt的证书,然后要在Nginx或Apache上设置好https的路由.
- 选择一个静态博客生成器,Jekyll/Hexo/Hugo/etc...其实用哪个都没所谓,保存好自己的markdown文件,然后哪个有心仪的主题就用哪个.甚至都可以自己写一个,解析markdown到HTML,再折腾出一套css就ok了.
- 将hugo或其他生成器生成的静态网站内容推送到托管的服务器上面,一般是`public`文件夹里面的内容.
- 最后建议设置一个闹钟每天提醒写博客,不然只有域名或vps续费的时候才会想起来自己居然还有个博客`¯\_(ツ)_/¯`

## 使用Google Firebase托管博客
使用[firebase](https://firebase.google.com/)的原因很简单:
- 免费
- 国内暂时还能访问(Firebase官网还是要梯子`╮(￣▽￣)╭`)
- 能自定义域名
申请Firebase帐号时,记得选择Spark类型账户.申请完之后创建一个类型为Hosting的新项目.可以使用firebase-cli将本地文件夹推送到服务器上.
```
curl -sL firebase.tools | bash  #安装firebase-cli
firebase login #完成本地登录
cd $BLOG     #切换到自己的博客文件夹
firebase init   
```
执行`firebase init`时会询问托管类型,因为是博客,选择Hosting就行啦.然后会有这么一个问题
```
? What do you want to use as your public directory? (public)
```
`Hugo`的生成的静态网站就在`public`文件夹下,因此默认回车就行.有的则是在`dist`等其他文件夹下,别填错了.错了也没关系,在`firebase.json`里改就行.
然后接下来执行
```
firebase login:ci
```
这会生成一个TOKEN,在github action自动推送到firebase里会用到,记得保存在`Github Repo`里的`Settings ->Secrets', 假如保存的名字为`FIREBASE_TOKEN`,那么`github action`可以直接读取.
```
${{ secrets.FIREBASE_TOKEN }}
```

至此和`Firebase`相关的工作就都完成了,`deploy`的步骤会交给`github action`来完成.


## 使用Github Action来构建博客
因为有了Github Action这种工具,什么安装Hugo, Hexo,Nodejs,yarn环境什么的,本地电脑可以完全不考虑,新建好文件夹保存markdown即可.
当然考虑到折腾博客主题,还是可以安装相应的博客程序.我这里使用的是`Hugo`, 主题使用的是[echo](https://github.com/forecho/hugo-theme-echo). 此时在我的Hugo博客文件夹下最重要的文件夹和文件夹只有两个:`content/`和`config.toml`,前者是我存放着我的markdown文件,后者是博客的设置. 
还有一个文件也挺重要:`firebase.json`,这是对应的`Firebase`设置.
然后要新建一个名为`.github/workflows`的文件夹
```
mkdir -p .github/workflows
```
这个文件夹存放着`github action`的各种`workflow`.
所以要编辑`.gitignore`文件,将除开`.github`, `content/`,  `config.toml`,  `firebase.json`的文件和文件夹通通写进去.
这里要注意的是图片存放的位置,别把这个文件夹也写进去了.我把`hugo`的图片存放在`/static/img`下,所以没有把`static/`加进`gitignore`.

然后要编辑`github action`的`workflow`,位置在刚刚建立的`workflows`文件夹下, 文件格式为`yml`,名字随意.主要有以下几个步骤:
1. 从`repo`拉取代码,通过`checkout`完成
2. 安装`Hugo`.由于我修改了主题,需要重新`build`主题,所以还要安装`Node`和`yarn`,如果没有这个要求则可以不用.
3. 接下来是使用`Hugo`生成博客代码
4. 然后是将拉取下来的`repo`里的代码复制到博客目录下
5. 安装主题到博客目录,这里我需要用`yarn`重新`build`一次.记得切换目录
6. 接下来就可以用`hugo`生成静态文件了
7. 最后是安装`firebase`和推送到`firebase`服务器啦. 这里没有用`github`上那个开源的`firebase action`,似乎有坑,目录总是不对.

很简单对不对ლ(╹◡╹ლ)


```
name: HugoToFirebase

on: [push]  # 选择在push的时候开始构建

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest

    steps:  
      - name: Checkout
        uses:  actions/checkout@v1

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "latest"
          extended: true
      - name: setup Node
        uses: actions/setup-node@v1
        with:
          node-version: '10.x'
      - name: setup Yarn
        uses: borales/actions-yarn@v2.0.0

      - name: new site
        run: |
          hugo new site $HOME/blog


      - name: copy file to blog dir
        run: |
          rm -rf $HOME/blog/content/
          /bin/cp -rvf config.toml $HOME/blog/config.toml
          /bin/cp -rvf content/ $HOME/blog/content/
          /bin/cp -rvf firebase.json $HOME/blog/
        
      - name: install theme
        run: |
          git clone https://github.com/Sisylocke/hugo-theme-even.git $HOME/blog/themes/even
          cd $HOME/blog/themes/even
          yarn install
          yarn build
                    
      - name: build
        run: |
          cd $HOME/blog
          HUGO_ENV=production hugo --gc --minify
      
      - name: push to firebase
        run: |
          sudo curl -o "/usr/local/bin/firebase" -L https://firebase.tools/bin/linux/latest
          sudo chmod +rx "/usr/local/bin/firebase"
          cd $HOME/blog
          firebase use khaki-161613 --token ${{ secrets.FIREBASE_TOKEN }}
          firebase deploy --token ${{ secrets.FIREBASE_TOKEN }}
```

## 总结一下
重点就是有了`github action`,本地的`markdown`可以更灵活的保存啦,而且不用其他考虑啥啥啥博客生成器了,只要能得到一堆静态文件,哪家都一样.现在你看我`github repo`是多么简洁
```
.
├── config.toml
├── content
│   └── post/
├── firebase.json
├── .github
│   └── workflows/
└── .gitignore

```
其中`content`文件夹是使用`hugo`时不小心的残留,以后会只留下里面的`post`文件夹.
这么一来, 要是以后不想用`Hugo`了,直接修改`workflow`代码即可,`repo`里的目录结构都不用动一下的,想想就觉得省心.
另外因为保存到了`gihub`上面, 也不用担心文章丢失的问题啦. 像以前用vps搭博客, 一不小心就`rm -rf`了(虽然拢共也就就两篇╮(￣▽￣)╭).

之前也考虑过图床的问题, 本来打算自建的,但是在思考为什么要自建图床这个问题后,我陷入了迷茫, `firebase`, `github`它不香吗?嗯, 等哪天`Firebase`凉了或者有更好的去处后再来折腾, 先暂时就这样了吧.

ლ(╹◡╹ლ)
