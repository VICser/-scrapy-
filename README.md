# -scrapy-
关于使用scrapy框架编写爬虫以及Ajax动态加载问题、反爬问题解决方案


总的来说，Python爬虫所做的事情分为两个部分，1：将网页的内容全部抓取下来，2：对抓取到的内容和进行解析，得到我们需要的信息。

 

目前公认比较好用的爬虫框架为Scrapy，而且直接使用框架比自己使用requests、 beautifulsoup、 re包编写爬虫更加方便简单。

 

1、关于Scrapy框架

 

简介： Scrapy是一个为了爬取网站数据，提取结构性数据而编写的应用框架。 其最初是为了 页面抓取 (更确切来说, 网络抓取 )所设计的， 也可以应用在获取API所返回的数据(例如 Amazon Associates Web Services ) 或者通用的网络爬虫。

 

官方文档地址 ： http://scrapy-chs.readthedocs.io/zh_CN/1.0/index.html

 

Scrapy安装 ：  pip install Scrapy

 

创建Scrapy项目 ： scrapy startproject scrapyspider(projectname)

该命令创建包涵下列内容的目录：

 

这些文件分别是:

scrapy.cfg: 项目的配置文件。
scrapyspider/: 该项目的python模块。之后您将在此加入代码。
scrapyspider/items.py: 项目中的item文件。
scrapyspider/pipelines.py: 项目中的pipelines文件，用来执行保存数据的操作。
scrapyspider/settings.py: 项目的设置文件。
scrapyspider/spiders/: 放置爬虫代码的目录。
 

  

 

 

 编写爬虫：以爬取豆瓣电影TOP250为例展示一个完整但简单的Scrapy爬虫的流程

 

首先，在items.py文件中声明需要提取的数据，Item 对象是种简单的容器，保 存了爬取到得数据。 其提供了 类似于词典(dictionary-like) 的API以及用于声明可 用字段的简单语法。许多Scrapy组件使用了Item提供的额外信息: exporter根据 Item声明的字段来导出 数据、 序列化可以通过Item字段的元数据(metadata) 来 定义、trackref 追踪Item 实例来帮助寻找内存泄露 (see 使用 trackref 调试内 存泄露) 等等。
Item使用简单的class定义语法以及Field对象来声明。我们打开scrapyspider目录下的items.py文件写入下列代码声明Item：

为了创建一个爬虫，首先需要继承scrapy.Spider类，定义以下三个属性：

1、name : 用于区别不同的爬虫，名字必须是唯一的。

2、start_urls: 包含了Spider在启动时进行爬取的url列表。

3、parse() 是spider的一个函数。 被调用时，每个初始URL完成下载后生成的 Response 对象将会作为唯一的参数传递给该函数，然后解析提取数据。

 

 

在scrapyspider/spiders目录下创建douban_spider.py文件，并写入初步的代码：

 

from scrapy import Request

from scrapy.spiders import Spider

from scrapyspider.items import DoubanMovieItem

 

 

class DoubanMovieTop250Spider(Spider):

    name = 'douban_movie_top250'

    headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.143 Safari/537.36',}#设置请求头文件，模拟浏览器访问（涉及反爬机制）

 

    def start_requests(self):#该函数定义需要爬取的初始链接并下载网页内容

        url = 'https://movie.douban.com/top250'

        yield Request(url, headers=self.headers)

 

    def parse(self, response):

        item = DoubanMovieItem()

        movies = response.xpath('//ol[@class="grid_view"]/li') #使用xpath对下载到的网页源码进行解析

        for movie in movies:

            item['ranking'] = movie.xpath(

                './/div[@class="pic"]/em/text()').extract()[0]

            item['movie_name'] = movie.xpath(

                './/div[@class="hd"]/a/span[1]/text()').extract()[0]

            item['score'] = movie.xpath(

                './/div[@class="star"]/span[@class="rating_num"]/text()'

            ).extract()[0]

            item['score_num'] = movie.xpath(

                './/div[@class="star"]/span/text()').re(ur'(\d+)人评价')[0]

            yield item

        next_url =  response.xpath('//span[@class="next"]/a/@href').extract() #解析得到下一页的链接

        if next_url:

            next_url = 'https://movie.douban.com/top250' + next_url[0]

            yield Request(next_url, headers=self.headers) #下载下一页的网页内容

 

运行爬虫： 在项目文件夹内打开cmd运行下列命令：

 

此处的douban_movie_top250即为我们刚刚写的爬虫的name, 而-o douban.csv是scrapy提供的功能，将item输出为csv格式的文件，存储到douban.csv中。

得到数据：

 

 

 

到此我们已经可以解决一般普通网站的抓取任务，普通网站是指网页源码之中包含了所有我们想要抓取的内容，但是有的时候一些网站采用了Ajax异步加载的方法，导致以上介绍的方法无法使用。

 

2、关于抓取Ajax异步加载的网站

 

Ajax是什么：

 

AJAX即“Asynchronous Javascript And XML”（异步JavaScript和XML），是指一种创建交互式网页应用的网页开发技术。

通过在后台与服务器进行少量数据交换，AJAX 可以使网页实现异步更新。这意味着可以在不重新加载整个网页的情况下，对网页的某部分进行更新。

通过Ajax异步加载的网页内容在网页源码中是没有的，也就是之前介绍的方法中下载到的response中是解析不到我们想要的内容的。

 

如何抓取AJAX异步加载页面：

 

对于这类网页，我们一般采用两种方法：

1、通过抓包找到异步加载请求的真正地址

2、通过PhantomJS等无头浏览器执行JS代码后再抓取

但是通常采取第一种方法，因为第二种方法使用无头浏览器会大大降低抓取的效率。

 

异步加载网站抓取示例 ：

使用豆瓣电影分类排行榜作为抓取示例，链接为

https://movie.douban.com/typerank?type_name=%E5%8A%A8%E4%BD%9C&type=5&interval_id=100:90&action=

电影信息网页源码中没有，并且采用鼠标下拉更新页面，这时需要我们在需要抓取的页面打开Chrome的开发者工具，选择network，实现一次下拉刷新

 



 

发现新增了一个get请求，并且响应为JSON格式。观察JSON的内容，发现正是需要抓取的内容。

抓取内容的问题解决了，接下来处理多页抓取问题，因为请求为get形式，所以首先进行几次下拉刷新，观察请求链接的变化，会发现请求的地址中只有start的值在变化，并且每次刷新增加20，其他都不变，所以我们更改这个参数就可以实现翻页。

由于之前已经在items.py中对需要抓取的数据做了声明，所以只需要在scraoyspider/spiders目录下创建一个新的爬虫文件douban_actions.py，代码如下：

import re

import json

 

from scrapy import Request

from scrapy.spiders import Spider

from scrapyspider.items import DoubanMovieItem

 

class DoubanAJAXSpider(Spider):

    name = 'douban_ajax' #设置爬虫名字为douban_ajax

    headers = {

        'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.143 Safari/537.36',

    } #设置请求头文件，模拟浏览器访问（涉及反爬机制）

    def start_requests(self):

        url = 'https://movie.douban.com/j/chart/top_list?type=5&interval_id=100%3A90&action=&start =0&limit=20'

        yield Request(url, headers=self.headers)

 

    def parse(self, response):

        datas = json.loads(response.body)#将Json格式数据处理为字典类型

        item = DoubanMovieItem()

        if datas:

            for data in datas:

                item['ranking'] = data['rank']

                item['movie_name'] = data['title']

                item['score'] = data['score']

                item['score_num'] = data['vote_count']

                yield item

 

            # 如果datas存在数据则对下一页进行采集

            page_num = re.search(r'start=(\d+)', response.url).group(1)

            page_num = 'start=' + str(int(page_num)+20)

            next_url = re.sub(r'start=\d+', page_num, response.url)

　　　　#处理链接

            yield Request(next_url, headers=self.headers)

　　　　#请求下一页

 

运行爬虫 ：

 

　　scrapy crawl douban_ajax -o douban_movie.csv

 

 

然而，很多时候ajax请求都会经过后端鉴权，不能直接构造URL获取。这时就可以通过PhantomJS、chromedriver等配合Selenium模拟浏览器动作，抓取经过js渲染后的页面。

使用这种方法有时会遇到定位网页页面元素定位不准的情况，这时就要注意网页中的frame标签，frame标签有frameset、frame、iframe三种，frameset跟其他普通标签没有区别，不会影响到正常的定位，而frame与iframe对selenium定位而言是一样的，需要进行frame的跳转。（这两点暂不展开，在抓取中财网—数据引擎网站时就遇到此类问题）

 

 

彩蛋：

两个提高效率的Chrome插件：

　　Toggle JavaScript  （检测网页哪些内容使用了异步加载）  

　　JSON-handle （格式化Json串）

 

 

3、关于突破爬虫反爬机制

 

   目前使用到的反反爬手段主要有三个：

 

1、在请求之间设置延时，限制请求速度。（使用python time库）

 

2、在每次请求时都随机使用用户代理User-Agent，为了方便，在scrapy框架中，可以使用fake-useragent这个开源库。

Github地址：https://github.com/hellysmile/fake-useragent

 

3、使用高匿代理ip，同样为了方便，在scrapy框架中编写爬虫，建议使用开源库 scrapy-proxies。

Github地址：https://github.com/aivarsk/scrapy-proxies
