---
title: 初识hexo及搭建Yilia主题博客
date: 2016-12-31 11:51:18
tags: [hexo, yilia]
toc: true
---

## hexo基本命令

|简称|全名|含义|  
|:---|:---|:---|    
|hexo n "blog"|hexo new "blog"|新建文章|  
|hexo p|hexo public|~~|  
|hexo g|hexo generate|生成|  
|hexo s|hexo server|启动服务预览|  
|hexo d|hexo deploy|部署|
|hexo s -g|hexo server --generate|发布|  
|hexo d -g|hexo deploy --generate|部署|

<!-- more -->

## Yilia主题搭建博客

博客目录：  
<!-- ![目录](img/20170101105937.png) -->
{% asset_img 20170101105937.png 这是一个新的博客的图片的说明 %}




### 获取Yilia主题

```bash
git clone https://github.com/litten/hexo-theme-yilia.git yilia
```


### 根目录下_config.yml配置
进入目录`blog\`下中的_config.yml文件配置如下：  

    # Hexo Configuration
    ## Docs: https://hexo.io/docs/configuration.html
    ## Source: https://github.com/hexojs/hexo/

    # Site
    title: mraye # 博客标题
    subtitle: 面朝大海 春暖花开 #博客副标题
    description: 面朝大海 春暖花开 #博客副标题
    author: mraye
    language: zh-CN #语言
    timezone: Asia/Shanghai #时区

    # URL
    ## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
    url: 'https://mraye.github.io/'  #用于绑定域名, 其他的不需要配置
    root: /
    permalink: :year/:month/:day/:title/
    permalink_defaults:

    # Directory
    source_dir: source #源文件
    public_dir: public #生成的网页文件
    tag_dir: tags #标签
    archive_dir: archives #归档
    category_dir: categories #分类
    code_dir: downloads/code
    i18n_dir: :lang #国际化
    skip_render:

    # Writing
    new_post_name: :title.md # File name of new posts
    default_layout: post
    titlecase: false # Transform title into titlecase
    external_link: true # Open external links in new tab
    filename_case: 0
    render_drafts: false
    post_asset_folder: true
    relative_link: false
    future: true
    highlight:
      enable: true
      line_number: true
      auto_detect: false
      tab_replace:

    # Category & Tag
    default_category: uncategorized
    category_map:
    tag_map:

    # Archives
    #Archives 默认值为2,修改为1,Archives页面就只会列出标题,而非全文
    ## 2: Enable pagination
    ## 1: Disable pagination
    ## 0: Fully Disable
    ## Archives
    # 2: 开启分页
    # 1: 禁用分页
    # 0: 全部禁用
    archive: 1
    category: 1
    tag: 1

    ## server_ip: 0.0.0.0
    logger: false
    logger_format:

    jsonContent:
        meta: false
        pages: false
        posts:
          title: true
          date: true
          path: true
          text: true
          raw: false
          content: false
          slug: false
          updated: false
          comments: false
          link: false
          permalink: false
          excerpt: false
          categories: false
          tags: true

    # Date / Time format
    ## Hexo uses Moment.js to parse and display date
    ## You can customize the date format as defined in
    ## http://momentjs.com/docs/#/displaying/format/
    date_format: YYYY-MM-DD
    time_format: HH:mm:ss

    # Pagination
    ## Set per_page to 0 to disable pagination
    per_page: 5 #每页文章数, 设置成 0 禁用分页
    pagination_dir: page

    # Extensions
    ## Plugins: https://hexo.io/plugins/
    ## Themes: https://hexo.io/themes/
    theme: yilia

    # Deployment
    ## Docs: https://hexo.io/docs/deployment.html
    deploy:
      type: git  #部署类型, 本文使用Github
      repository: git@github.com:mraye/mraye.github.io.git #部署的仓库的SSH
      branch: master

    # plugins:
    # - hexo-generator-feed   #此插件用于RSS订阅


### 主题Yilia中_config.yml配置
进入目录`blog\themes\yilia\`下中的_config.yml文件  
yilia主题中的_config.yml配置:   

    # Header
    menu:
      主页: /
      归档: /archives
    #  关于我: /about
    #  标签: /categories
      # 归档: /categories/hexo
      # 所有文章: /archives
    #  随笔: /tags/随笔/
    # SubNav
    subnav:
      github: "https://github.com/mraye"
      # weibo: "#"
      # rss: "#"
      #douban: "#"
      #mail: "mailto:litten225@qq.com"
      #facebook: "#"
    rss: /atom.xml
    # 是否需要修改 root 路径
    # 如果您的网站存放在子目录中，例如 http://yoursite.com/blog，
    # 请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/。
    root:

    # Content
    # 文章太长，截断按钮文字
    excerpt_link: more
    # 文章卡片右下角常驻链接，不需要请设置为false
    show_all_link: '阅读全文'
    fancybox: true
    # 数学公式
    mathjax: false
    # 是否在新窗口打开链接
    open_in_new: false

    # 打赏
    # 请在需要打赏的文章的md文件头部，设置属性reward: true

    # 打赏基础设定：0-关闭打赏； 1-文章对应的md文件里有reward:true属性，才有打赏； 2-所有文章均有打赏
    reward_type: 0
    # 打赏wording
    reward_wording: '谢谢你请我吃糖果'
    # 支付宝二维码图片地址，跟你设置头像的方式一样。比如：/assets/img/alipay.jpg
    alipay:
    # 微信二维码图片地址
    weixin:

    # Miscellaneous
    google_analytics: ''
    favicon: /favicon.png

    #你的头像url
    avatar: /img/avatar.jpg # 自定义图片放在 /themes/yilia/source/img/下即可

    #是否开启分享
    share_jia: false

    #是否开启多说评论，填写你在多说申请的项目名称 duoshuo: duoshuo-key
    #若使用disqus，请在博客config文件中填写disqus_shortname，并关闭多说评论
    duoshuo: mraye #就是在设置多说域名.duo.com之前，例如域名为xxx.duoshuo.com,那么这里就写xxx

    #是否开启云标签
    tagcloud: true

    # 按年份归档,被坑了好久
    nav:
        name: By Year
        url: /archives

    # 智能菜单
    # 如不需要，将该对应项置为false
    # 比如
    #smart_menu:
    #  friends: false
    smart_menu:
      innerArchive: '所有文章'
      aboutme: '关于我'

    #开启——
    aboutme: 面朝大海，春暖花开  



### 主题完善

#### 修改头像  

将头像图片放在`/themes/yilia/source/img/avatar.jpg`中，然后在yilia的配置文件`_confing.yml`中配置:    

    # Miscellaneous
    # google_analytics: ''
    favicon: /favicon.png




#### 修改背景颜色

个人喜欢暗黑色，所以就将背景颜色稍稍修改了一下,找到`/themes/yilia/source/main.css`下的`#container.show .anm-canvas`属性，修改为如下就可以:      

```css
#container.show .anm-canvas {
    display: block;
    position: fixed;
    background-color: #272822  # <<<
}
```

#### 自动生成目录
在`/themes/yilia/layout/_partial/article.ejs`中找到`<%- post.content %>`位置,在`<%- post.content %>`的前面加入一下代码:  
```bash
else { %>
       <% if(!index && post.toc && post.toc !== false){ %>
          <div id="toc" class="toc-article">
            <strong class="toc-title">文章目录</strong>
            <%- toc(post.content) %>
          </div>
        <%}%>
        <%- post.content %>
<% } %>
```
在每次写文章的时候只需要将`toc`设置为`true`即可,如下:   
```bash
title: 初识hexo及搭建Yilia主题博客
date: 2016-12-31 11:51:18
tags: [hexo, yilia]
toc: true
```




### hexo写作技巧
+ 若希望页面显示时内容在一定程度就不显示,使用`more`标签:  
```bash
<!-- more -->
```








---
>**文章历史：**    
`2016-12-31` 创建
