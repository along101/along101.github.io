---
title: Hexo+Github搭建个人博客
date: 2018-07-15 11:29:57
tags: [Hexo,Github,博客]
categories: [工具]
author: 李志明
---

本篇作者`李志明`，转至 [http://techblog.ppdai.com/2018/07/06/20180706/](http://techblog.ppdai.com/2018/07/06/20180706/)

## 前言

作为程序猿，相信大家都有过这样一个想法，搭建属于自己的博客网站，在上面写写技术文章，记录生活点滴，坚持下去会发现这是一件很有成就感的事情。最近刚好在学习这部分内容，深入进去后发现里面坑很多，为了节省大家的时间，少走一点弯路，我整理出了这篇文章供大家参考。

<!--more-->

## 为什么选择Hexo

之前在网上搜了一下目前比较流行的静态博客框架，最后目标锁定在Jekyll和Hexo上，两者都支持Markdown语法，这点我很喜欢，Jekyll基于Ruby实现，安装Jekyll需要搭建Ruby环境，而Hexo基于NodeJs实现，在Windows上安装NodeJs开发环境比Ruby简单，另外Hexo的主题相对来说更符合我的审美，所以最终选择了Hexo。

什么是Hexo？官网对它的介绍是：

> Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

## 准备工作

### 安装Node.js

下载地址：https://nodejs.org/en/download/

推荐下载LTS版本的msi文件，默认64-bit，也可根据自己的Windows版本选择32-bit。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/nodejs-install/nodejs-download.png)

保持默认设置即可，一路Next。安装完成后打开命令行窗口，输入命令：

```bash
$ node -v
$ npm -v
```

结果如下图所示，则说明安装正确，可以进行下一步，如果不正确，请回头检查自己的安装过程。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/nodejs-install/nodejs6.png)

### 安装git

下载地址：https://git-scm.com/downloads

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-install/git-download.png)

保持默认设置即可，一路Next。安装完成后打开命令行窗口，输入：

```bash
$ git --version
```

结果如下图所示，则说明安装正确，可以进行下一步，如果不正确，请回头检查自己的安装过程。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-install/git11.png)

### github配置

第一步，注册一个github账号，记得点击邮箱中的验证链接，注册地址：https://github.com

第二步，生成SSH keys

什么是ssh：ssh是Secure Shell（安全外壳协议）的缩写，建立在应用层和传输层基础上的安全协议。为了便于访问github，要生成ssh公钥，这样就不用每一次访问github都要输入用户名和密码。

1.本地成功安装git后，单击鼠标右键，选择Git Bash Here，打开git bash。

2.输入命令：

```bash
$ ssh-keygen -t rsa -C "xxx@xxx.com"
```

引号中的内容是你在github上的注册邮箱，之后一路回车，结果如图所示：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-sshkey1.png)

3.上一步已经成功的生成了ssh key，接下来输入：

```bash
$ eval "ssh-agent -s"
```

然后输入：

```bash
$ ssh-add ~/.ssh/id_rsa
```

这一步可能会报错：`Could not open a connection to your authentication agent` ，这时直接输入：

```bash
$ ssh-agent bash
```

再次输入：

```bash
$ ssh-add ~/.ssh/id_rsa
```

结果如图所示：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-sshkey2.png)

4.用cat命令查看key的内容：

```bash
$ cat ~/.ssh/id_rsa.pub
```

选中内容，右键复制备用，如图：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-sshkey3.png)

第三步，配置SSH keys

打开github页面，找到setting中的ssh keys，点击新增按钮，输入任意的title，将刚才复制的key粘贴进去保存即可。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config1.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config2.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config3.png)

第四步，测试

输入命令：

```bash
$ ssh -T git@github.com
```

结果如图：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-sshkey4.png)

到这里为止，准备工作就全部完成了。

## Hexo的安装与配置

第一步，安装Hexo

打开命令行窗口，输入命令：

```bash
$ npm install -g hexo-cli
```

安装完成后输入：

```bash
$ hexo version
```

结果如下图所示，则说明安装正确。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo2.png)

如果报错：`'hexo'不是内部或外部命令，也不是可运行的程序或批处理文件` ，则需要检查环境变量配置是否正确，如下图所示，编辑Path变量值，在结尾处加上：`C:\Program Files\nodejs\node_global;`（文件hexo.cmd所在目录）

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/nodejs-install/nodejs7.png)

第二步，初始化Hexo

进入任意目录，比如F盘，然后指定一个文件夹名，这里以blog为例，命令如下：

```bash
$ hexo init blog
```

结果如下图所示，F盘下会多出一个blog文件夹：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo3.png)

接下来进入blog目录：

```bash
$ cd blog
```

第三步，安装必要的依赖

```bash
$ npm install
```

第四步，生成静态文件

```bash
$ hexo generate
```

该命令的简写形式为：

```bash
$ hexo g
```

执行完毕后会在blog目录下生成一个public文件夹，里面包含了博客网站的所有静态资源。

第五步，启动服务器

```bash
$ hexo server
```

该命令的简写形式为：

```bash
$ hexo s
```

默认情况下，访问地址为：http://localhost:4000/

另外也可以指定端口（比如8000）：

```bash
$ hexo s -p 8000
```

第六步，验证

在浏览器中打开上面的地址，你将会看到：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo1.png)

到这里为止，Hexo的安装与相关配置就全部完成了。

## 将Hexo与Github Pages联系起来

第一步，创建代码库

1.登录github，点击页面右上角的加号，选择New repository

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config4.png)

2.在Repository name中填写 `yourname.github.io` ，注意这里的yourname指的是你的github用户名，如果你的名字是kirito，那就填 `kirito.github.io` ，Description中可以填一些简单的描述，不写也没关系，然后点击Create repository

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config5.png)

3.正确创建之后，你会看到如下界面：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/git-config/git-config6.png)

第二步，编辑站点配置文件

打开blog目录下的_config.yml文件，编辑deploy模块，内容如下：

```
deploy:
  type: git
  repo: git@github.com:lizhiming1995/yourname.github.io.git
  branch: master
```

注意这里的repo地址应该换成你第一步创建的代码库的地址。

第三步，安装一个扩展

进入blog目录，打开命令行窗口，输入命令：

```bash
$ npm install hexo-deployer-git --save
```

安装完成后，就可以一键部署到github上了。

第四步，部署

```bash
$ hexo deploy
```

该命令的简写形式为：

```bash
$ hexo d
```

执行结果如图所示：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo4.png)

这样你public目录下的静态文件就上传到你的代码库中了。

第五步，激活GitHub Pages

打开代码库的Settings页面，找到GitHub Pages，选择master branch，然后点击Save按钮，如图：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo5.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo6.png)

最后会提示你：`Your site is ready to be published at http://yourname.github.io/`

这就是你的博客地址了，任何人都可以访问哦。

## 绑定自己的域名

第一步，在万网、腾讯云、阿里云等提供域名注册的域名服务商处购买一个域名，这里以阿里云为例，购买地址：https://wanwang.aliyun.com/

第二步，打开控制台，给域名添加DNS解析

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/dns1.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/dns2.png)

添加两条解析记录，记录类型为CNAME，主机记录分别填@和www，记录值填之前GitHub Pages提供的域名，注意没有http的前缀，如下图：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/dns3.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/dns4.png)

添加完后别人用www和不用www都能访问你的网站。

第三步，在blog目录的source文件夹下创建一个CNAME文件，记住不要有文件后缀名，编辑CNAME文件，里面写你购买的域名，例如 `yourname.com` ，记住不要有www，创建完成后每一次执行 `hexo g` 都会在public文件夹下生成CNAME文件，方便后面的部署

第四步，在blog目录下打开命令行窗口，运行 `hexo g` ，再运行 `hexo d`

第五步，在浏览器输入你购买的域名，你会发现该域名已经指向了你在github上的博客地址

注意：设置域名解析需要几分钟的时间，若完成以上步骤依然无法访问，请过几分钟再尝试

## Hexo入门

我们先来看一下blog的目录结构：

```
+ blog
  + public        //静态资源文件夹，内容会推送到代码库
  + scaffolds     //模板文件夹，新建文章时，Hexo会根据模板来建立文件
  + source        //资源文件夹，Markdown和HTML文件会被解析并放到public文件夹，而其他文件会被拷贝过去
  + themes        //主题文件夹，Hexo会根据主题来生成静态页面
  - _config.yml   //网站的配置信息，可以在此配置大部分的参数
  - package.json  //应用程序的信息和依赖关系
```

方便起见，我们把网站的语言设置为中文，编辑blog目录下的_config.yml文件，将language这一项设置为 `language: zh-CN`（参考blog/themes/landscape/languages目录），将url这一项设置为 `url: http://yourname.com`（你购买的域名，若未购买可以用 `http://yourname.github.io` 代替），其他配置项请根据自己的需要进行设置。

接下来新建一篇文章：

```bash
$ hexo new [layout] <title>

```

layout可选值有：draft（草稿）、page（页面）、post（文章），对应模板文件夹中的3个文件，如果没有设置layout的话，默认使用_config.yml中的default_layout参数（默认值post）代替。若标题包含空格，请使用引号括起来。

现在，我们来新建一篇名为test的文章，输入以下命令：

```bash
$ hexo new test
```

结果在source/_posts目录下生成了test.md文件，内容如下：

```
---
title: test
date: 2018-06-23 19:14:56
tags:
---
```

这里给出Front-matter的概念，Front-matter是文件最上方以 `---` 分隔的区域，用于指定文件的变量。

常见参数：title（标题）、date（创建日期）、tags（标签）、categories（分类），只有文章（post）支持标签和分类参数，建议文章分类只写一个，标签可以有多个，写法为 `tags: [tag1,tag2,tag3]` ，注意每个参数的冒号后面都应该有一个空格，这一点同样体现在_config.yml文件中

编辑test.md文件，内容如下：

```
---
title: test
date: 2018-06-23 19:14:56
tags: [tag1,tag2,tag3]
categories: java
---
文章正文
```

保存后刷新页面，通常情况下页面会自动更新，若修改没有生效，则需要重新执行以下命令：

```bash
$ hexo g
$ hexo s
```

这里再介绍一个命令：

```bash
$ hexo clean
```

它的作用是清除缓存文件 (db.json) 和已生成的静态文件 (public)。在某些情况下（尤其是更换主题后），如果你发现对站点的更改无论如何也不生效，你可能需要运行该命令。

打开网站首页，你会看到刚才设置的标签和分类生效了：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo7.png)

发现Hello World这篇文章的内容被折叠起来了吗，很简单，只需要在文章正文合适的地方加上 `<!--more-->` 这一行代码就搞定了。

菜单栏只有Home和Archives？没关系，我们可以加个页面（page），这里以about为例，在blog目录下打开命令行窗口，输入命令：

```bash
$ hexo new page about
```

结果会在source目录下生成about文件夹，里面包含一个index.md文件，文件内容就是about页面的内容。

还没有结束，编辑blog/themes/landscape目录下的_config.yml文件，修改menu的配置为：

```
menu:
  首页: /
  归档: /archives
  关于: /about
```

保存刷新页面，你会看到导航栏里多了“关于”，点进去就是about页面啦，目前只有一个标题，内容待编辑，注意页面是不支持设置标签和分类的，只有文章才支持。

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo8.png)

最后，我们来总结一下发布文章的流程：

第一步，`hexo new <title>` 生成一篇文章，这里的title指文件名，不建议使用中文

第二步，编辑文章，修改title、tags等参数，这里的title指文章标题，可以使用中文

第三步，`hexo s` 本地预览效果，不满意继续修改

第四步，`hexo g` 生成静态文件

第五步，`hexo d` 将静态文件推送至代码库

第四步和第五步可以合并成一条命令，`hexo d -g` ，表示部署之前预先生成静态文件。修改配置与发布文章的流程相似，最后都需要执行第三四五步。

## Hexo进阶

### 添加RSS订阅功能

RSS是在线共享内容的一种简易方式，也叫简易信息聚合，全称Really Simple Syndication。当网站内容更新时，可以通过订阅RSS源在RSS阅读器上获取更新的信息，大多数的内容提供网站都会提供RSS订阅功能，方便用户去获取最新的内容。

1.安装feed插件

Hexo有一个专门生成RSS文件的插件 `hexo-generator-feed` ，进入blog目录，打开命令行窗口，输入命令：

``` bash
$ npm install hexo-generator-feed --save
```

2.启用插件

编辑blog目录下的_config.yml文件，添加如下内容：

```
# Extensions
Plugins:
- hexo-generator-feed
# Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20
```

3.生成RSS

``` bash
$ hexo g
```

如果生成了atom.xml就表示成功了：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo9.png)

在浏览器中打开 http://localhost:4000/atom.xml ，你会看到订阅功能已开启：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo10.png)

4.部署

``` bash
$ hexo d
```

5.使用RSS订阅功能

这里以Office的Outlook邮箱为例，订阅地址假设为 `http://spring2go.com/atom.xml`，如图：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo11.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo12.png)

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo13.png)

### 添加站点地图

站点地图是一种文件，你可以通过该文件列出你网站上的网页，从而将你网站内容的组织架构告知Google和其他搜索引擎。Googlebot等搜索引擎网页抓取工具会读取此文件，以便更加智能地抓取你的网站。

1.安装sitemap插件

``` bash
$ npm install hexo-generator-sitemap --save
$ npm install hexo-generator-baidu-sitemap --save
```

2.编辑blog目录下的_config.yml文件，添加如下内容：

```
Plugins:
- hexo-generator-sitemap
- hexo-generator-baidu-sitemap

sitemap:
  path: sitemap.xml

baidusitemap:
  path: baidusitemap.xml
```

3.生成站点地图文件

``` bash
$ hexo g
```

如果生成了sitemap.xml和baidusitemap.xml就表示成功了：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo15.png)

4.部署

``` bash
$ hexo d
```

5.确认博客是否被收录

在百度或者谷歌输入下面格式的内容，如果能搜索到就说明被收录，否则未收录，可能需要等上一段时间。

```
site:xxx.com
```

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo14.png)

### 使用模板功能

现在我们生成的每一篇新文章都只有title、date、tags三个参数，通常情况下我们还会有categories参数和一些自定义的参数（如何使用自定义参数我们后面讲），每次都要手动加上这些参数会浪费很多时间，这时候模板的作用就出来了。

打开scaffolds文件夹，你会看到里面有draft、page、post三个模板，对应草稿、页面、文章，我们日常使用最多的就是文章，所以这里以文章为例，其他两个模板请根据自己的需要进行修改。

模板的参数是可以设置默认值的，我们假设categories参数的默认值为 `随笔` ，然后自定义一个参数 `author` ，默认值为 `kirito` ，因为每篇文章的标签是不确定的，所以这里不进行设置，编辑post.md文件，内容如下：

```
---
title: {{ title }}
date: {{ date }}
tags: 
categories: 随笔
author: kirito
---
```

让我们用模板生成一篇新文章：

```bash
$ hexo new test2
```

打开blog/source/_posts目录下的test2.md文件，可以看到以下内容：

```
---
title: test2
categories: 随笔
author: kirito
date: 2018-06-27 22:02:33
tags:
---
```

接下来我们只需要写好文章，设置一下tags就可以发布了。

### 使用自定义参数

文章参数里的title、date、categories和tags都在页面上有所展示，那我们自定义的参数该如何使用和展示呢？

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo16.png)

通过控制台我们可以看到，每篇文章都是一个 `class="article article-type-post"` 的 `article` 标签，结构如下：

```html
<article id="post-test" class="article article-type-post" itemscope="" itemprop="blogPost">
    <div class="article-meta">
        <a href="/2018/06/23/test/" class="article-date">...</a>
        <div class="article-category">...</div>
    </div>
    <div class="article-inner">
        <header class="article-header">...</header>
        <div class="article-entry" itemprop="articleBody">...</div>
        <footer class="article-footer">...</footer>
    </div>
</article>
```

title、date、categories和tags的值分别显示于article-header、article-date、article-category和article-footer，要使用自定义参数，我们需要修改主题的源文件，打开 `blog/themes/landscape/layout/_partial` 目录下的article.ejs文件，可以看到代码中的标签与class名都与上面一致，接下来我们让作者的名字显示在分类右边，编辑article-meta模块的代码：

```html
<div class="article-meta">
    <%- partial('post/date', {class_name: 'article-date', date_format: null}) %>
    <%- partial('post/category') %>
    <div style="letter-spacing:2px;color:#999;line-height:1em;">
        <%- post.author %>
    </div>
</div>
```

这里为了方便直接将样式写在了div标签里，更好的做法是为div添加一个class，将样式写进 `landscape/source/css/_partial`
 目录下的article.styl文件。当然，自定义标签可以用在其他地方，样式也可以根据你的喜好来定制。
 
让我们打开浏览器来看下效果，你会发现文章的自定义标签生效了：

![](https://images-1256966106.cos.ap-shanghai.myqcloud.com/hexo/hexo17.png)

值得注意的是，我们刚才修改的文件是article.ejs，这是跟主题有关的，换一个主题，也许文件的路径和名字都变了，甚至格式也不再是ejs而是swig，不过修改文件的思路都是一样的，明确自己要修改哪一个模块，然后到主题的相关目录下，模仿源代码的语法进行修改，最后记住，源文件里使用Tab键会导致修改的代码无效或者报错，请使用空格。

那么教程到这里就结束了，快来试试搭建自己的博客吧，有什么问题可以在评论区留言~

ps：后面我会单独整理一篇关于Hexo主题的文章，不用期待...