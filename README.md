
### 1.数据库表创建

首先通过python的sqlalchemy模块，来新建一个表。

creat_lagou_tables.py

```python
from sqlalchemy import create_engine, Integer,String,Float
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column


#创建数据库的连接
engine = create_engine("mysql+pymysql://root:123456@127.0.0.1:3306/lagou?charset=utf8")
#操作数据库，需要我们创建一个session
Session = sessionmaker(bind=engine)

#声明一个基类
Base = declarative_base()

class Lagoutables(Base):
    #表名称
    __tablename__ = 'lagou_data'
    #id,设置为主键和自动增长
    id = Column(Integer,primary_key=True,autoincrement=True)
    #岗位ID,非空字段
    positionID = Column(Integer,nullable=True)
    # 经度
    longitude = Column(Float, nullable=False)
    # 纬度
    latitude = Column(Float, nullable=False)
    # 岗位名称
    positionName = Column(String(length=50), nullable=False)
    # 工作年限
    workYear = Column(String(length=20), nullable=False)
    # 学历
    education = Column(String(length=20), nullable=False)
    # 岗位性质
    jobNature = Column(String(length=20), nullable=True)
    # 公司类型
    financeStage = Column(String(length=30), nullable=True)
    # 公司规模
    companySize = Column(String(length=30), nullable=True)
    # 业务方向
    industryField = Column(String(length=30), nullable=True)
    # 所在城市
    city = Column(String(length=10), nullable=False)
    # 岗位标签
    positionAdvantage = Column(String(length=200), nullable=True)
    # 公司简称
    companyShortName = Column(String(length=50), nullable=True)
    # 公司全称
    companyFullName = Column(String(length=200), nullable=True)
    # 公司所在区
    district = Column(String(length=20), nullable=True)
    # 公司福利标签
    companyLabelList = Column(String(length=200), nullable=True)
    # 工资
    salary = Column(String(length=20), nullable=False)
    # 抓取日期
    crawl_date = Column(String(length=20), nullable=False)

if __name__ == '__main__':
    #创建数据表
    Lagoutables.metadata.create_all(engine)
```

首先在mysql里面新建lagou数据库，然后运行查看查看已经创建好的字段
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200608175858878.png)


### 2.数据爬取入库

通过python中的request模块接口的形式调取数据。

思路：

（1）先获取所有城市信息：需要用request模块中的requests.session()，session对象保存访问接口需要用到的信息：例如cookies等信息。

（2）通过城市分组，再用正则表达式筛选来获取python的岗位信息。其中多次用到列表生成器，以后要多注意这方面的冷知识；不然会有莫名的错误。

代码思路：只要保证可复用即可，其实很简单，毕竟Python是一门”干净“的语言。

（1）先把请求方法抽集到一个方法中：session.get(url（地址），headers（头信息），,timeout（时间），proxies（代理信息）)

（2）先获取所有城市，利用列表生成器生成一个list把数据装进去。

（3）利用循环以城市分组拉去Python岗位信息。for city in lagou.city_list:调用拉取岗位信息的方法。

（4）导入multiprocessing模块，设置多线程加速抓取：multiprocessing.Pool(自定 int or long)，在useragent 里面设置伪用户登录，这里有一个度量，当我爬取数据出现失败的时候，这里要暂停10秒，如果不暂停继续的话，有极大几率ip被封。需要注意的是：必须使用多线程拉取。由于拉勾网近几年反扒措施加强，不加多线程和伪用户登录，以及访问时间的设置，效率会低下，可能导致信息不全，时间太慢

handle_crawl_lagou.py

```python
import json
import re
import time
import requests
import random
import multiprocessing
from fake_useragent import UserAgent
from lagou_spider.handle_insert_data import lagou_mysql
ua = UserAgent()
class HandleLaGou(object):
    def __init__(self):
        #使用session保存cookies信息
        self.lagou_session = requests.session()
        self.header = {

            "Accept": "application/json, text/javascript, */*; q=0.01",
            "Connection": "keep-alive",
            "Host": "www.lagou.com",
            "Referer": 'https://www.lagou.com/jobs/list_python?city=%s&cl=false&fromSearch=true&labelWords=&suginput=',
            "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
            'User-Agent': 'ua.random'}
        self.city_list = ""

    #获取全国所有城市列表的方法
    def handle_city(self):
        city_search = re.compile(r'zhaopin/">(.*?)</a>')
        city_url = "https://www.lagou.com/jobs/allCity.html"
        city_result = self.handle_request(method="GET",url=city_url)
        #使用正则表达式获取城市列表
        self.city_list = city_search.findall(city_result)
        self.lagou_session.cookies.clear()

    def handle_city_job(self,city):
        first_request_url = "https://www.lagou.com/jobs/list_python?city=%s&cl=false&fromSearch=true&labelWords=&suginput="%city
        first_response = self.handle_request(method="GET",url=first_request_url)
        total_page_search = re.compile(r'class="span\stotalNum">(\d+)</span>')
        try:
            total_page = total_page_search.search(first_response).group(1)
            print(total_page)
        #由于没有岗位信息造成的exception
        except:
            return
        else:
            for i in range(1,int(total_page)+1):
                data = {
                    "pn":i,
                    "kd":"python"
                }
                page_url = "https://www.lagou.com/jobs/positionAjax.json?city=%s&needAddtionalResult=false"%city
                referer_url = "https://www.lagou.com/jobs/list_python?city=%s&cl=false&fromSearch=true&labelWords=&suginput="%city
                #referer的URL需要进行encode
                self.header['Referer'] = referer_url.encode()
                response = self.handle_request(method="POST",url=page_url,data=data,info=city)
                lagou_data = json.loads(response)
                job_list = lagou_data['content']['positionResult']['result']
                for job in job_list:
                    lagou_mysql.insert_item(job)


    def handle_request(self,method,url,data=None,info=None):
        while True:
            try:
                if method == "GET":
                    # response = self.lagou_session.get(url=url,headers=self.header,proxies=proxy,timeout=6)
                    response = self.lagou_session.get(url=url,headers=self.header,timeout=6)
                elif method == "POST":
                    # response = self.lagou_session.post(url=url,headers=self.header,data=data,proxies=proxy,timeout=6)
                    response = self.lagou_session.post(url=url,headers=self.header,data=data,timeout=6)
            except:
                # 需要先清除cookies信息
                self.lagou_session.cookies.clear()
                # 重新获取cookies信息
                first_request_url = "https://www.lagou.com/jobs/list_python?city=%s&cl=false&fromSearch=true&labelWords=&suginput=" % info
                self.handle_request(method="GET", url=first_request_url)
                time.sleep(20)
                continue
            response.encoding = 'utf-8'
            if '频繁' in response.text:
                print(response.text)
                #需要先清除cookies信息
                self.lagou_session.cookies.clear()
                # 重新获取cookies信息
                first_request_url = "https://www.lagou.com/jobs/list_python?city=%s&cl=false&fromSearch=true&labelWords=&suginput="%info
                self.handle_request(method="GET",url=first_request_url)
                time.sleep(20)
                continue
            return response.text

if __name__ == '__main__':
    lagou = HandleLaGou()
    # 所有城市的方法
    lagou.handle_city()
    print(lagou.city_list)
    # # 引入多进程加速抓取
    pool = multiprocessing.Pool(2)
    for city in lagou.city_list:
        pool.apply_async(lagou.handle_city_job,args=(city,))
    pool.close()
    pool.join()
    while True:
        city_list = ['北京','上海','广州','深圳','大连','成都','哈尔滨','重庆','无锡','西安','天津','石家庄','青岛','宁波']
        for city in city_list:
            lagou.handle_city_job(city)
            time.sleep(10)

```

查看数据库
![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-wkZ3VnfX-1591610286290)(https://gitee.com/shuaidapao/blogimg4/raw/master/img/20200603203412.png)\]](https://img-blog.csdnimg.cn/20200608175829384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMTY2OTI5,size_16,color_FFFFFF,t_70)
