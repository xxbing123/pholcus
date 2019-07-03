# Pholcus爬虫框架（改造版）
原官方爬虫框架[henrylee2cn/pholcus](https://github.com/jason-wj/pholcus)基本停止更新，由于个人项目需要，对本项目做了一些修改和完善。
有兴趣对可以看看原官方项目的介绍。
感谢：[henrylee2cn](https://github.com/jason-wj)

## 使用
需要将项目放在$GOPATH/github.com/jason-wj/目录中

## 后续打算（仅仅只是打算，根据业余时间来安排）
1. 将pipe部分自定义化，能够根据需要灵活实现
2. 添加新的功能，将爬虫结果输出到nsq中

## 该项目个人使用感悟
1. 能够将精力集中在规则的实现和完善中，不用专门去考虑输出结果和方式，已经给出固定模板。
2. 每只爬虫独立化，互不干扰
3. 规则简单易懂，规则格式已确定，任何网站的爬虫规则，都按照规则格式来实现就行，不同的网站也很容易理解该爬虫规则（原先使用过scrapy，随意实现规则，写到后面别人再来看会花费很多不必要的时间去理解）

## 隐藏功能
pholcus官方文档只提供了基本的使用方式，很多隐藏的功能需要自行根据源码来挖掘，这里我罗列一些自己用到的：
1. 增量更新：pholcus是根据链接是否已经爬取过来判断是否继续爬取该链接，因此只要在链接之后加入一个时间戳参数，此时pholcus就会认为这是一个新的链接
2. 业务判断认为失败的链接，想要从pholcus的历史记录中删除（如果不删除，那下次该链接更新了内容就很难判断是否要爬取该页面了），此时只需要在请求结果的规则中加入如下代码即可：
```
ctx.Request.SetReloadable(true)
```

## 当前如下修改和完善
1. 为方便在爬虫规则中调用ctx.Sleep时能够动态切换爬虫间隔频率，在`pholcus/app/crawler/crawler.go`第200行处加入：
```
self.setPauseTime()
```  

2. 该框架是使用map将爬虫结果导入到mongo中的，原先爬虫上下文中的参数默认值不能为空（只能搞成""空字符串），这就回导致json解析数据时候同时解析了该参数，后面不仅浪费空间（etl时候会将该空字段再次加入到mongo，很浪费资源），而且容易造成误理解。  
为此，在`pholcus/app/downloader/request/request.go`第282行将如下代码注释：
```
if defaultValue == nil {
	panic("*Request.GetTemp()的defaultValue不能为nil，错误位置：key=" + key)
}
```  

3. 多次发起请求时候，head会被重复利用，这样有的爬虫规则下， 会造成请求错误，始终无法继续（会误以为是ip被封），为此，注释掉`pholcus/app/downloader/surfer/param.go`第60行如下代码：
```
param.header = req.GetHeader()
```
4. 当页面请求错误，获取不到数据时候（提示`convert err xxx`），此时如果错误数量超过goroutine限制的上限，则会陷入死锁状态，为此需要在`pholcus/app/spider/context.go`第643行加入如下一行代码：
```
self.text = []byte("") //防止self.text为nil
```

5. 新增方法，用于获取请求到的页面的原始[]byte数据。原先没有提供时，如果要获取图片或者自行处理数据是很难搞的，为此在`pholcus/app/spider/context.go`第579行处加入如下代码：  
```  
// GetBytes returns plain bytes crawled.
func (self *Context) GetBytes() []byte {
	if self.text == nil {
		self.initText()
	}
	return self.text
}
```

6. 当response编码为"image/jpeg"或者没有指定编码时，不要进行转码操作（默认会转为utf-8，会影响图片等内容等展示）,需要在`pholcus/app/spider/context.go`第641行加入一项：
```  
"image/jpeg",""
```

7. 部分网站可能会发生url变化，此时继续爬取，会被识别为新的url来爬取，会造成和旧的url爬的数据重复。为了解决这个问题，需要在`pholcus/app/downloader/request/request.go`第19行加入：
```  
UrlAlias      string          //url别名，主要是为了防止网站url发生变化，影响去重。（若网站url变化，只需要在此处加入旧的url就行）
```
同时在142行加入：
```  
// 请求的唯一识别码
func (self *Request) Unique() string {
	if self.unique == "" {
		if self.UrlAlias != "" {
			block := md5.Sum([]byte(self.Spider + self.Rule + self.UrlAlias + self.Method))
			self.unique = hex.EncodeToString(block[:])
		} else {
			block := md5.Sum([]byte(self.Spider + self.Rule + self.Url + self.Method))
			self.unique = hex.EncodeToString(block[:])
		}
	}
	return self.unique
}
```
以后只要在Request中，指定`UrlAlias`的旧根地址即可

8. 为更直观展示代理使用时候的错误提示，在`pholcus/app/aid/proxy/proxy.go`第239行加入：
ps：曾犯下一个错误，代理测试始终报错，后来才知道，代理ip需要加上`http://`前缀，就是因为源码中忽略了下面的错误提示
```  
if err != nil {
	logs.Log.Informational(" *     [%v]代理测试发生错误：" + err.Error())
}
```

9. 将[henrylee2cn/teleport](https://github.com/henrylee2cn/teleport)和[henrylee2cn/goutil](https://github.com/henrylee2cn/goutil)两个辅助源码直接放在`/pholcus/common`目录中

10. 加入爬虫规则示例包到项目根目录

剩余调整将会根据后续需要来逐步调整。。。
