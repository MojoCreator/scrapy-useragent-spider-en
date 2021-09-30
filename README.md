# The whole site crawler
Scrapy + selenium/webdriver + Random request header + IP proxy + twisted ConnectionPool + mysql Crawl the whole site crawler for a certain book

### Environment installation  
```vim
pip install requirements
```
### Database configuration
在setting.py Configure in。
Connect to the database using **pymysql**

```python
    # database config
    # Database host IP
    HOST = '127.0.0.1'
    # database username
    USER = "root"
    # Database password
    PASSWORD = "password"
    # Name database
    DATABASE_NAME = 'jianshu'
```
### selenium + webdriver Add middleware to crawl web pages twenty four Because of the dynamic website, the data is dynamically loaded through js, or there is a powerful anti-crawler technology 25 So here we use selenium + chromedriver (chrome browser) to completely simulate browser operations to crawl.

**tips : selenium + phantomjs Also works well**
```python
  class ChromeDriverDownloaderMiddleware(object):
      '''
      Use selenium + chromedriver to crawl web pages
      '''
      def __init__(self):
          driver_path = r'D:\python\chromedriver\chromedriver.exe'
          self.driver = webdriver.Chrome(executable_path=driver_path)
      #
      def process_request(self,request,spider):
          self.driver.get(request.url)
          source = self.driver.page_source
          response = HtmlResponse(self.driver.current_url,body=source,request=request)
          return response
      def process_response(self,request,response,spider):
          pass
```


### Random request header settings
 http://www.useragentstring.com
 Go to this website, you can find all
 User-Agent
 
 Set by the middle button User_Agent
 
 ```python
     class UserAgentRandomDownloaderMiddleware(object):
         '''
         Set random head
         '''
         USER_AGENT = [
             'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36',
             'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML like Gecko) Chrome/51.0.2704.79 Safari/537.36 Edge/14.14931',
             'Mozilla/5.0 (Windows NT 6.1; WOW64; rv:64.0) Gecko/20100101 Firefox/64.0',
             'Galaxy/1.0 [en] (Mac OS X 10.5.6; U; en)',
             'Mozilla/5.0 (compatible, MSIE 11, Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko'
         ]
     
         def process_request(self, request, spider):
             request.headers['User-Agent'] = random.choice(self.USER_AGENT)
 ```
 ### Set up IP proxy How to get the proxy IP:
    
   - Web crawling, here is an open source IP proxy pool project:
   https://github.com/qiyeboy/IPProxyPool
   - Spend money to buy such as: fast agent, sesame agent, etc
 
**In the project, we must make full use of a proxy IP when we get it. The correct use here means that we must wait for the current IP to expire or be disabled by the website before we get a new IP. Pay special attention to asynchronous crawlers. For multiple threads or processes, for the management of proxy IP, avoid excessive waste。**

This project uses Sesame Proxy to obtain the proxy IP, the code is as follows
 ```python
     class IPProxyDownloaderMiddleware(object):
         '''
         IPproxy ，
         '''
         # Get the proxy IP information address, such as Zhima proxy, fast proxy, etc.
         IP_URL = r'http://xxxxxx.com/getip?xxxxxxxxxx'
     
         def __init__(self):
             # super(IPProxyDownloaderMiddleware, self).__init__(self)
             super(IPProxyDownloaderMiddleware, self).__init__()
     
             self.current_proxy = None
             self.lock = DeferredLock()
     
         def process_request(self, request, spider):
             if 'proxy' not in request.meta or self.current_proxy.is_expire:
                 self.updateProxy()
     
             request.meta['proxy'] = self.current_proxy.address
     
         def process_response(self, request, response, spider):
             if response.status != 200:
                 # If you come here, this request is equivalent to being recognized as a crawler
                 # So this request is abolished 104 If you don't return the request, then the request is not getting the data
                 # If the request is returned, then this request will be added to the governor again
                 if not self.current_proxy.blacked:
                     self.current_proxy.blacked = True
                     print("Blacked out")
                 self.updateProxy()
                 return request
             # Under normal circumstances, return response
             return response
     
         def updateProxy(self):
             '''
             Get new proxy ip
             :return:
             '''
             # Because it is an asynchronous request, in order not to send too many requests to the Zhima proxy at the same time, here is the proxy IP.
             # When you need to lock
             self.lock.acquire()
             if not self.current_proxy or self.current_proxy.is_expire or self.current_proxy.blacked:
                 response = requests.get(self.IP_URL)
                 text = response.text
     
                 # return value {"code":0,"success":true,"msg":"0","data":[{"ip":"49.70.152.188","port":4207,"expire_time":"2019-05-28 18:53:15"}]}
                 jsonString = json.loads(text)
     
                 data = jsonString['data']
                 if len(data) > 0:
                     proxyModel = IPProxyModel(data=data[0])
                     self.current_proxy = proxyModel
             self.lock.release()
 ```
 
  ```python
     class IPProxyModel():
         '''
         Proxy ip model class
         '''
         def __init__(self,data):
             # IP
             self.ip = data['ip']
             # The port number
             self.port = data['port']
             # Expiration
             self.expire_time_str = data['expire_time']
             # Is it blacklisted
             self.blacked = False
     
             date_str,time_str = self.expire_time_str.split(" ")
             year,month,day = date_str.split('-')
             hour,minute,second = time_str.split(":")
             # Expiration
             self.expire_time = datetime(year=int(year),month=int(month),day=int(day),hour=int(hour),minute=int(minute),second=int(second))
     
             self.address = "https://{}:{}".format(self.ip,self.port)
     
         @property
         def is_expire(self):
             '''
             Verify whether the proxy IP has expired
             :return:
             '''
             now = datetime.now()
             if self.expire_time - now < timedelta(seconds=10):
                 return True
             else:
                 return False
  ```
  
  ### Save data to database 173 This project uses the database connection pool provided by twisted to insert data into the database 174 How to use:
  
  Import
```pyhton
    
     from twisted.enterprise import adbapi
     from pymysql import cursors
   ```
  Configure database parameters and obtain connection pool objects:
  
  ```pyhton
      
        db_params = {
           'host': settings.HOST,
           # 'port':'3306'
           'port': 3306,
           'user': settings.USER,
           'password': settings.PASSWORD,
           'database': settings.DATABASE_NAME,
           'charset': 'utf8',
           'cursorclass': cursors.DictCursor
         }
         
         # Use twisted provided ConnectionPool
         self.dbpool = adbapi.ConnectionPool('pymysql', **db_params)
   ```
   
   Insert data into the database:
   
 ```pyhton
    defer = self.dbpool.runInteraction(self.insert_item, item)
  ```
  
  ### Start crawler Run start.py to start the crawler
  
   ```pyhton
      from scrapy import cmdline
      
      # Execute on the command line
 scrapy crawl jianshu Order
      cmdline.execute('scrapy crawl jianshu'.split())
   ```
