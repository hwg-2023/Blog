---
title: "Hexo搭建全流程"
date: 2024-06-25 01:10:00
categories:
  -Hexo
cover: http://zan-cloud.tech:9001/tuchuang/math1.png
---

##  必要安装内容

1. git 、node.js 

详细安装可以直接去百度搜索，这里已经装好了，使用的是最新的node -21.7.3



##  本地如何将Hexo项目跑起来

1. 使用 npm 命令

   ```shell
   npm install -g hexo
   # 为了全局部署hexo
   ```

2. 直接 init 整个hexo

   ```shell
   hexo init
   #会将hexo整个项目初始化
   ```

3. 直接运行访问 http://localhost:4000/

   ```shell
   hexo s
   #应该就是 start 的含义
   ```



* 启动访问效果：

  ![image-20240602002059558](http://zan-cloud.tech:9001/tuchuang/image-20240602002059558.png)





云服务器安装 git 的命令；

sudo yum install git -y

不知道为什么会有这么多不可达的情况出现，极有可能就是因为那啥了，云服务器上外网被墙了。

看来可能要给云服务器搞个梯子了。



接下来是装 node.js ，可以先安装 nvm ，然后通过nvm 安装node.js ，不然 hexo 无法运行，毕竟hexo就是用node.js 写的

因为CentOS版本过低，所以导致了我这边下载的较新版的node.js 无法使用，所以这边可能要降低 node.js的版本

先用 nvm uninstall <current_version> 来降低一波

事实证明，CentOS7 最高的node支持版本是 16.x



接下来是关键，就是如何配置 本地 blog 通过 git 推送文章到云服务器上并完成自动部署的操作。



云服务器上的hexo部署在 /home/www/hexo 上面



接下来为 git 用户添加 SSH 密钥：

目前为止，对git仓库进行的所有操作都需要输入密码，我们可以通过为git用户添加SSH密钥的方式来实现免密登录。原理：在本地生成一对密钥文件（分别是公钥和私钥），将公钥文件上传到服务器上；服务器收到连接请求时，会将本地的公钥与服务器的公钥进行比对，用私钥解密服务器发来的一段信息并将其发回，验证通过后准许连接。



使用 root 账户 登录，为公钥文件和文件夹设置权限。

```shell
chmod 600 /home/git/.ssh/authorized_keys
chmod 700 /home/git/.ssh
```



将.ssh文件夹及其文件的所有权移交给git用户。

chown -R git:git /home/git/.ssh

成功！做到了不需要密码就可以登录的操作了



接下来创建 Git 仓库并配置自动部署

其中 /home/repo 是Git仓库（自定义的）

mkdir -p /home/repo     #该目录为git仓库所在的位置，可根据喜好自行设置
cd /home/repo     #进入该目录
git init --bare blog.git     #创建一个名为blog的仓库，--bare参数为创建裸库



进入 `/home/repo/blog.git/hooks` 目录，找到 `post-receive` 钩子文件（若无则新建）。

右键编辑，输入 `git --work-tree=/home/www/hexo --git-dir=/home/repo/blog.git checkout -f` 并保存。（work-tree填写hexo的部署目录，git-dir填写Git仓库的目录）。

我这边确实要自己部署一下hexo了。



云服务器在载入 hexo 文件的时候出现了一个很莫名其妙的报错：

![image-20240615181204150](http://zan-cloud.tech:9001/tuchuang/image-20240615181204150.png)



还发现了一个bug，就是每次重新开始会话的时候都会出现 node -v 和 npm -v 不可用的情况，这种情况出现是因为可能每次启动新的 shell 会话的时候系统默认选择的 版本和你下载的版本出现了不一，所以每次都导致你必须使用 nvm use xxx 命令来进行重新选择。



nvm alias default 16 

默认选定 16 版本的 node



md ，竟然卡在了安装 hexo 上面，气死我了。。。。



从理论上来讲，我用本地的机器试过了，绝对不是node.js版本的问题，因为16.20.1是可以运行最新的Hexo版本的

感觉问题出在了一开始安装了node.js 用宝塔安装了，所以这边出现了路径问题

莫名其妙就成功拉？？？？？？？

离了个大谱。

```shell
sudo chown -R $(whoami) ~/.npm
sudo chown -R $(whoami) /root/.nvm/versions/node/v16.20.2/lib/node_modules
```

拿下！！！



![image-20240615231514685](http://zan-cloud.tech:9001/tuchuang/image-20240615231514685.png)



git --work-tree=/home/www/hexo --git-dir=/home/repo/blog.git checkout -f        work-tree填写hexo的部署目录，git-dir填写Git仓库的目录



修改权限：

```shell
chmod +x /home/repo/blog.git/hooks/post-receive     #为钩子文件授予可执行权限（+x）
chown -R git:git /home/repo     #将仓库目录的所有权移交给git用户
chown -R git:git /home/www/hexo     #将hexo部署目录的所有权移交给git用户
```



本地hexo修改：





git clone https://github.com/amehime/hexo-theme-shoka.git ./themes/shoka





为网站配置SSL证书以及采用https协议访问

主要是下载证书后这个 Nginx 的配置应该怎么写的问题。



```
server 
  {
    listen        443 ssl;
    server_name   zan-cloud.tech;                     #自己的域名
    
    ssl_certificate zan-cloud.tech_server.crt;        #.crt证书路径
    ssl_certificate_key zan-cloud.tech_server.key;    #.key证书路径
    
    ssl_session_cache shared:SSL:1m; 
    ssl_session_timeout 5m;
    
    ssl_ciphers HIGH:!aNULL:!MD5;                     #禁用某些算法协议
    ssl_prefer_server_ciphers on;
    
    location / {
      root /home/www/hexo;                            #部署时候的文件夹（服务器上的）
      index index.html index.htm;
    }
  }  
```



基本已经拿下这个 Hexo 了总的来说



目前最重要的是更换样式和熟悉Hexo相关配置。





```yaml
markdown:
  render: # 渲染器设置
    html: false # 过滤 HTML 标签
    xhtmlOut: true # 使用 '/' 来闭合单标签 （比如 <br />）。
    breaks: true # 转换段落里的 '\n' 到 <br>。
    linkify: true # 将类似 URL 的文本自动转换为链接。
    typographer: 
    quotes: '“”‘’'
  plugins: # markdown-it 插件设置
    - plugin:
        name: markdown-it-toc-and-anchor
        enable: true
        options: # 文章目录以及锚点应用的 class 名称，shoka 主题必须设置成这样
          tocClassName: 'toc'
          anchorClassName: 'anchor'
    - plugin:
        name: markdown-it-multimd-table
        enable: true
        options:
          multiline: true
          rowspan: true
          headerless: true
    - plugin:
        name: ./markdown-it-furigana
        enable: true
        options:
          fallbackParens: "()"
    - plugin:
        name: ./markdown-it-spoiler
        enable: true
        options:
          title: "你知道得太多了"
          
minify:
  html:
    enable: true
    exclude: # 排除 hexo-feed 用到的模板文件
      - '**/json.ejs'
      - '**/atom.ejs'
      - '**/rss.ejs'
  css:
    enable: true
    exclude:
      - '**/*.min.css'
  js:
    enable: true
    mangle:
      toplevel: true
    output:
    compress:
    exclude:
      - '**/*.min.js'
	
highlight:
  enable: false
prismjs:
  enable: false
  
  
autoprefixer:
  exclude:
    - '*.min.css'
    

algolia:
  appId: #Your appId
  apiKey: #Your apiKey
  adminApiKey: #Your adminApiKey
  chunkSize: 5000
  indexName: #"shoka"
  fields:
    - title #必须配置
    - path #必须配置
    - categories #推荐配置
    - content:strip:truncate,0,2000
    - gallery
    - photos
    - tags


keywords: "408单知识点归纳","数学知识点归纳","论文书写进度","日常"#站点关键词，用 “,” 分隔

feed:
    limit: 20
    order_by: "-date"
    tag_dir: false
    category_dir: false
    rss:
        enable: true
        template: "themes/shoka/layout/_alternate/rss.ejs"
        output: "rss.xml"
    atom:
        enable: true
        template: "themes/shoka/layout/_alternate/atom.ejs"
        output: "atom.xml"
    jsonFeed:
        enable: true
        template: "themes/shoka/layout/_alternate/json.ejs"
        output: "feed.json"
```



一般用来部署的操作都是连用的：

```shell
hexo clean && hexo g && hexo s
hexo clean && hexo generate && hexo deploy

```





又又又遇到问题了，在云服务器上装那几个玩意的时候，翻车了。。。

就是用来渲染 主题 shoka 那几个依赖，云服务器上一直装不了。
