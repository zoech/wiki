# 利用selenium编写spider的一些记录

1. 环境依赖
    1. python selenium 模块
    1. geckodriver + firefox, 或者 chrome + chrome driver
    
1. 建立driver并打开网页
```python
from selenium import webdriver

firefox_options = webdriver.FirefoxOptions()
firefox_options.set_headless()
driver = webdriver.Firefox(firefox_options=firefox_options)
dirver.get("https//www.baidu.com")

```

3. driver 建立，打开某个域名之后可以向driver里加cookie，

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
