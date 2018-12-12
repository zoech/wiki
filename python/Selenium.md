# 利用selenium编写spider的一些记录
[TOC]

## 1. 环境依赖
    1. python selenium 模块
    1. geckodriver + firefox, 或者 chrome + chrome driver
    
## 2. 建立driver并打开网页
```python
from selenium import webdriver

firefox_options = webdriver.FirefoxOptions()
firefox_options.set_headless()
driver = webdriver.Firefox(firefox_options=firefox_options)
dirver.get("https//www.baidu.com")

```

## 3. 添加cookie
 driver 建立打开某个域名之后可以向driver里加cookie，需要注意，driver只有在打开了一个网站之后，才能加入cookies，否则会报InvalidCookieDomainException

先建立cookie的一个json文件cookies.json，格式大概如下：

```json
[
	{
		"name":"acw_tc",
		"value":"76b20f7015401937601657397e027aa3417d38f2d6b9ec38a693e1634d2cd2"
	},
	{
		"name":"PHPSESSID",
		"value":"almslc7sid9bbg4npt3mvo3a95"
	},
	{
		"name":"Hm_lvt_ff3eefaf44c797b33945945d0de0e370",
		"value":"1540193760,1540261619,1540449081,1540947498"
	},
	{
		"name":"Hm_lpvt_ff3eefaf44c797b33945945d0de0e370",
		"value":"1540947498"
	},
	{
		"name":"synct",
		"value":"1540947525.363"
	}
]
```

然后在代码里打开这个文件，并将cookie加入到driver中
```python
import codecs

f = codecs.open("./cookies.json", mode='r', encoding = 'utf-8')
  cookies = json.loads(f.read())
  for cookie in cookies:
    driver.add_cookie({
      'name':cookie['name'],
      'value':cookie['value']
      }
    )

f.close()
```

## 4. 模拟页面输入、点击
 可以使用driver.find_element_by_xpath() 来获取页面元素，具体还有哪些方法可以选择页面元素，可以在idle里按tab键查看，例如下面模拟登陆

```python
driver.find_element_by_xpath("//input[@name='username']").send_keys("user")
driver.find_element_by_xpath("//input[@name='password']").send_keys("password")
driver.find_element_by_xpath("//div[@class='ivu-form-item-content']/button[@class='ivu-btn ivu-btn-primary']").click()

```

注意：一般操作页面元素需要等待页面加载完毕，可以了解下WebDriverWait方法

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait


WebDriverWait(driver, 15).until(
                EC.presence_of_element_located((By.CLASS_NAME, 'ivu-form-item-content'))
            )
```

## 5. 解释页面内容
   页面内容可以使用lxml模块来解释，例子如下:

```python
from lxml import etree

body = driver.page_source #获取页面html源码
page = etree.HTML(body) # 构建html对象

# 下面获取class为'ivu-table-tbody'的tbody元素里的tr子元素列表,
# 模式类似上面driver.find_element_by_xpath
tr_list = page.xpath("//tbody[@class='ivu-table-tbody']/tr")
```

## 6. 页面滚动
要模拟页面上下滚动，需要使用dirver的脚本解释接口，即driver.execute_script()  

例子(将页面滚动到高度47000)：
```python
while int(driver.execute_script("return document.body.scrollHeight;")) < 47000:
            driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
```
