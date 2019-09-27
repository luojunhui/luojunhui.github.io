---
title: 以豆瓣电影为例,带你入门scrapy爬虫开发  
date: 2019-07-09 16:52:10  
tags:
- scrapy
- python 
categories:
- Python
---

> 最近一时冲动开通了一年某视频网站VIP会员,我却不知道有什么值得去看的,故看看一些经典电影吧,所以去豆瓣电影搜一波评分高的来作参考.  
> 在娱乐的同时也不能放松学习,本文将讲解python知名爬虫框架scrapy开发[豆瓣电影](https://movie.douban.com/explore "豆瓣选电影")信息爬虫.  
> 开发环境: ubuntu + python3

## 磨刀不误砍柴工,先搞定开发环境:  
安装必要依赖:   
```bash
sudo apt-get install python-dev python-pip libxml2-dev libxslt1-dev zlib1g-dev libffi-dev libssl-dev
```

创建python3虚拟环境:  
``` bash 
pip install virtualenv virtualenv -p
python3 .venv source .venv/bin/activate
```

安装scrapy: 
```bash 
    pip install scrapy 
```

安装完后,来验证一下是否安装成功(有版本信息即成功):  
```bash 
    scrapy version
```

一般来说执行以上命令就完成了,但每个人的环境都不一样,如果有报错,请参考官方文档:[https://docs.scrapy.org/en/latest/intro/install.html](https://docs.scrapy.org/en/latest/intro/install.html)
或者使用[anaconda](https://docs.anaconda.com/anaconda/ "anaconda"),如果pip安装速度太慢的话,请考虑使用pip国内源,这个网上很多教程.

## 善用工具之创建项目:
   使用scrapy命令生成一个标准项目,并进入项目根目录:
   ```bash
       scrapy startproject doubanmovie
    cd doubanmovie
    ```  
生成第一个spider文件:
```bash 
scrapy genspider douban movie.douban.com
```
   使用`tree`查看整个项目结构,请战略性忽略文件夹`__pycache__`下的内容:
 ```text
 .
├── doubanmovie
│   ├── __init__.py                         # 包定义
│   ├── items.py                            # 模型
│   ├── middlewares.py                      # 中间件
│   ├── pipelines.py                        # 管道
│   ├── __pycache__                         # 战略性忽略
│   │   ├── __init__.cpython-36.pyc
│   │   └── settings.cpython-36.pyc
│   ├── settings.py                         # 配置文件
│   └── spiders                             # spider(蜘蛛)文件夹,可以定义多个spider
│       ├── douban.py                       # 生成的spider(蜘蛛)文件
│       ├── __init__.py                     # 包定义
│       └── __pycache__                     # 战略性忽略
│           ├── douban.cpython-36.pyc
│           └── __init__.cpython-36.pyc
└── scrapy.cfg                              # scrapy运行的配置文件
 ```
## 分析网站,然后编码:
用浏览器(推荐使用chrome/firefox)打开[豆瓣电影](https://movie.douban.com "豆瓣电影"),
点击网页上的`选电影`,然后就能看到我们要即将要爬的数据(见下图):`电影类型`,`电影名称`,`电影评分`.

[![doubanmovie.png](https://i.loli.net/2019/07/09/5d2433718bdf787219.png)](https://i.loli.net/2019/07/09/5d2433718bdf787219.png)

***
首先得分析网页源码,右键查看源代码,搜索第一部电影的名字`恶人传`,发现搜不到内容,
那么就判断该网页的电影数据是动态加载的,也就是常见的`AJAX`,
那么我们打开浏览器开发者工具(F12), 切换到`netWork/网络`后刷新下当前网页,
分析到两个请求地址:
* 电影类型: 
``` url
  https://movie.douban.com/j/search_tags?type=movie&source=
 ```
  
* 电影列表: 
 ```url
  https://movie.douban.com/j/search_subjects?type=movie&tag=%E7%83%AD%E9%97%A8&sort=recommend&page_limit=20&page_start=0
```

电影列表请求地址参数表:

| 参数 | 值 | 
|---| --- | 
| type | movie |
| tag | 热门(通过URL解码得出) |
| sort | recommend |
| page_limit | 展示电影的数量 |
| page_start | 从第几条记录开始 |

很明显,先获取到了所有的电影类型,然后每个类型作为参数拼凑到电影列表中,
就能请求到该电影类型下的数据,接下来就开始写代码.   
编辑`items.py`,虽然可以直接用`dict`接收数据,但规范一点还是写个item吧.
```python
# -*- coding: utf-8 -*-
import scrapy


class DoubanmovieItem(scrapy.Item):
    name = scrapy.Field() # 电影名字
    rate = scrapy.Field() # 电影评分
    tag = scrapy.Field() # 电影类型
```
编辑`douban.py`: 
```python
# -*- coding: utf-8 -*-
# 引用python自带的json
import json

import scrapy
# 引用我们定义好的item模型
from ..items import DoubanmovieItem


class DoubanSpider(scrapy.Spider):
    # 这个spider(蜘蛛的名字)
    name = 'douban'
    # 不在此允许范围内的域名就会被过滤,不会进行爬取,其实这个可要可不要
    allowed_domains = ['movie.douban.com']
    # 这是入口
    start_urls = ['https://movie.douban.com/j/search_tags?type=movie&source=']
    # 定义电影详情url
    movie_info_url = 'https://movie.douban.com/j/search_subjects?' \
                     'type=movie&tag={tag}&sort=recommend&page_limit={page_limit}&page_start={page_start}'

    # 从start_urls获取第一个url访问后会进入这个方法
    def parse(self, response):
        # body_as_unicode能尽量返回文本是unicode的,比较好处理
        body = response.body_as_unicode()
        # 字符串转json
        obj = json.loads(body)
        tags = obj.get('tags', [])
        for tag in tags:
            cur_page = 1
            # 定义meta,可以传递参数到下一个方法中
            meta = dict(tag=tag,  # 电影类型
                        page_limit=20,  # 展示电影的数量
                        page_start=(cur_page - 1) * 20,  # 从第几条记录开始
                        cur_page=cur_page
                        )
            # 生成每个类型下的第一页的电影列表url
            first_page_url = self.movie_info_url.format(**meta)
            yield scrapy.Request(url=first_page_url, callback=self.parse_movie_list_page, meta=meta)

    # 解析电影列表
    def parse_movie_list_page(self, response):
        # parse里定义的meta
        meta = response.meta

        body = response.body_as_unicode()
        obj = json.loads(body)
        subjects = obj.get('subjects', [])
        # 将列表里的电影信息提取出来
        for movie in subjects:
            item = DoubanmovieItem()
            item['name'] = movie['title']
            item['rate'] = movie['rate']
            item['tag'] = meta['tag']
            yield item

        # 进行翻页请求,本文只作参考,所以就爬前三页内容后停止
        cur_page = meta['cur_page']
        if cur_page < 3:
            meta['cur_page'] = cur_page + 1
            meta['page_start'] = (meta['cur_page'] - 1) * 20
            next_page_url = self.movie_info_url.format(**meta)
            # 这里的回调还是本方法,因为格式都没有变动,只是内容变了
            yield scrapy.Request(url=next_page_url, callback=self.parse_movie_list_page, meta=meta)

```

修改`settings.py`: 
```python
# USER_AGENT,模拟浏览器UA,可以自行修改成其他的
USER_AGENT = 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36'
# 是否遵守ROBOTSTXT
ROBOTSTXT_OBEY = False
```
## 牛刀小试之运行项目:
在项目根目录下,控制台输入命令(得先激活虚拟环境):`scrapy crawl douban -o
movie.json`,如无意外便能看到一堆信息输出,在底部看到`'item_scraped_count':
1020`,说明爬到了1020条数据,保存在`movie.json`里,大家可以打开这个文件看看是否真的有1020条电影数据信息.  

到这里就说明这个爬虫开发完成了,查看完整代码请查看github:[https ://github.com/luojunhui/doubanmovie](`https://github.com/luojunhui/doubanmovie`)

## 写在最后:
也许过一段时间,网页发生改动,到时候就需要自行分析网页了,本文仅仅是抛砖引玉,scrapy更多操作等着你挖掘!