[1]: https://github.com/mawenbao/niu-x2-sidebar
[2]: https://github.com/mawenbao/extract_headings
[3]: https://github.com/mawenbao/niux2_lazyload_helper
[4]: https://github.com/mawenbao/pelican-update-date
[5]: https://github.com/mawenbao/pelican-blog-content/tree/master/plugins/summary
[6]: https://github.com/mawenbao/pelican-blog-content/tree/master/plugins/sitemap
[7]: https://github.com/mawenbao/pelican-blog-content
[8]: http://www.dongxf.com/3_Build_Personal_Blog_With_Pelican_And_GitHub_Pages.html

# Dreams8记录
本仓库用于存放Pelican博客的源文件和配置等。

## 依赖
### 初始化插件
主题插件以子项目方式添加

    git submodule add https://github.com/dreams8/niu-x2-sidebar.git themes/niu-x2-sidebar
    git submodule add https://github.com/mawenbao/extract_headings.git plugins/extract_headings
    git submodule add https://github.com/mawenbao/niux2_lazyload_helper.git plugins/niux2_lazyload_helper
    git submodule add https://github.com/mawenbao/pelican-update-date.git plugins/pelican-update-date

更新

    git submodule update

### Pelican依赖
* [niu-x2-sidebar][1]主题
* [extract_headings][2]插件: 从html文件里提取h1~h6标题并生成目录列表
* [niux2_lazyload_helper][3]插件: 延迟加载图片
* [pelican-update-date][4]插件: 提取文章内的修改时间
* [sitemap][5]插件: 生成sitemap
* [summary][6]插件: 提取第一句话作为摘要

### python依赖
主题或插件的额外依赖

* pelican-minify: 压缩html文件
* beautifulsoup4: 解析html文件
* Pillow: PIL

使用pip安装

    pip install pelican-minify beautifulsoup4
    pip install PIL --allow-external PIL --allow-unverified PIL

### 更新
参考：

[小贴士](http://pelican-docs-zh-cn.readthedocs.org/en/latest/tips.html)

    pip install ghp-import
    
    git clone https://github.com/dreams8/dreams8.github.io.git && cd dreams8.github.io
    git branch gh-pages

    git checkout gh-pages
    ghp-import ../output/
    git checkout master
    git merge gh-pages
    git push --all

## 感谢
目录结构参考[MWB日常笔记][7]。
Pelican搭建教程参考[dongxf][8]
