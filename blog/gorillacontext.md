
### 导语
做过web开发的同学肯定都知道，我们经常使用 `r *http.Request` 这个变量来获取我们希望获得的参数，但是我们经常遇到这样一个场景，我们需要为我们的`r`设置更多的key-value形式的附加值，一般我们都会存储在一个Map对象中，用的时候再从里面取出，Golang考虑到这样一个场景，为我们提供了一个context，这里的context并不是上下文的意思，属于另外一个这个包`github.com/gorilla/context`
### 安装
使用这个库之前，我们同样用Golang中的经典获取包的方式go get命令来获取
```
go get github.com/gorilla/context
```
### 使用方法
#### 设置
```
context.Set(r, "user", "wuyazi")
context.Set(r, "age", 21)
```
在上面的这个函数里，`r`即是`*http.Request`我们设置了一个key为user，value为wuyazi，另一个key为age，value为21。
#### 获取
```
user := context.Get(r, "user").(string)
age := context.Get(r, "age").(int)
```
获取我们使用一个Get()的函数，该函数会返回一个`interface{}`类型的值，然后我们使用断言，进行类型转换，就能得到我们设置的数据了。
### 底层原理学习——源码剖析
#### Set()函数
这个函数接收了3个参数，这3个参数见名知意，这里不再过多介绍，在这个函数里，我们首先可以看到一个关于写入的锁来保证数据的安全性，接着判断我们传入的`r`是否已经存在了我们的data变量里。这里的data是一个双层嵌套的map，定义如下所示
`make(map[*http.Request]map[interface{}]interface{})`interface{}类型则表示了我们可以存入各种类型的数据，如果判断data[r]没有值，则为它开辟一个map，如果有值则直接存入，最后释放写入锁。
在此函数中我们同样看到了一个变量datat，datat是用来保存每个r对应map的声明周期，下面会讲到。

```
func Set(r *http.Request, key, val interface{}) {
	mutex.Lock()
	if data[r] == nil {
		data[r] = make(map[interface{}]interface{})
		datat[r] = time.Now().Unix()
	}
	data[r][key] = val
	mutex.Unlock()
}
```
#### Get()函数
同样在这个函数中我们用了一个读的锁，来保证读的安全性，里面的逻辑很简单，基本上和操作map一应，有值返回值，没有值则返回nil。
```
func Get(r *http.Request, key interface{}) interface{} {
	mutex.RLock()
	if ctx := data[r]; ctx != nil {
		value := ctx[key]
		mutex.RUnlock()
		return value
	}
	mutex.RUnlock()
	return nil
}
```
#### GetOk()函数
GetOk函数主要用于判断key是否存在于map对象中，但是为什么会有这个函数，可能有的同学会问，直接判断value是否为nil不就行了，其实这里需要注意的是，nil值同样可以当做value存储在map对象里，因此有这样一个函数是有必要的，这里面函数的逻辑同样和操作map类似，用golang中特有的map对象判断value的方式ok来判断是否有值，如果key不存在将会返回`nil,false`
```
func GetOk(r *http.Request, key interface{}) (interface{}, bool) {
	mutex.RLock()
	if _, ok := data[r]; ok {
		value, ok := data[r][key]
		mutex.RUnlock()
		return value, ok
	}
	mutex.RUnlock()
	return nil, false
}
```
#### GetAll()函数
这个函数是用来获取所有的键值对，同样用上面的例子使用方法如下：
```
allParams:=context.GetAll(r)
user:=allParams["user"].(string)
age:=allParams["age"].(int)
```
首先获得map对象，然后使用断言，比较简单。在源码中，需要注意的是在进行map对象返回的时候，我们新创建了一个和context大小一样的map对象，用来进行map对象的copy，为什么用这样的方法，因为原来的context是一个引用的类型，如果返回引用类型，调用者可能会破坏原来的map结构，因此返回一个map的copy保证的map的安全性。我们在实际开发中同样应该考虑到这一点。这样我们的代码才能更加具有健壮性。
```
func GetAll(r *http.Request) map[interface{}]interface{} {
	mutex.RLock()
	if context, ok := data[r]; ok {
		result := make(map[interface{}]interface{}, len(context))
		for k, v := range context {
			result[k] = v
		}
		mutex.RUnlock()
		return result
	}
	mutex.RUnlock()
	return nil
}
```
#### Delete()和Clear()函数
这两个函数的实现原理和map对象相同，比较简单，不在多阐述，源码如下:
```
func Delete(r *http.Request, key interface{}) {
	mutex.Lock()
	if data[r] != nil {
		delete(data[r], key)
	}
	mutex.Unlock()
}

func Clear(r *http.Request) {
	mutex.Lock()
	clear(r)
	mutex.Unlock()
}

func clear(r *http.Request) {
	delete(data, r)
	delete(datat, r)
}
```
#### Purge()函数
上面我们提到了一个datat的变量，在这个函数中我们就用到了，用它来对我们的key-value进行可控的清理,如果maxAge<0则重新进行创建map，实现清理所有，如果当前时间-maxAge的值大于创建的时间戳，则清理掉数据。
```
func Purge(maxAge int) int {
	mutex.Lock()
	count := 0
	if maxAge <= 0 {
		count = len(data)
		data = make(map[*http.Request]map[interface{}]interface{})
		datat = make(map[*http.Request]int64)
	} else {
		min := time.Now().Unix() - int64(maxAge)
		for r := range data {
			if datat[r] < min {
				clear(r)
				count++
			}
		}
	}
	mutex.Unlock()
	return count
}
```
#### ClearHandler()函数
上面讲到了手动清理，这里则是自动清理，因为在我们进行开发时，随着key-value的增加，我们大多数情况下会忘记清理，因此我们可以使用这个函数。函数的逻辑同样很简单，用Golang提供的defer机制，当这个函数结束之前调用defer延迟，进行清理。
```
func ClearHandler(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer Clear(r)
		h.ServeHTTP(w, r)
	})
}
```
### 扩展
在Set()函数中有一个`time.Now().Unix()`的使用，这个函数是获取当前的时间戳，是time包的一个用法，这里总结了time包对于时间的常用操作，如下：
#### 获取当前时间
```
currentTime:=time.Now()     //获取当前时间，类型是Go的时间类型Time
t1:=time.Now().Year()        //年
t2:=time.Now().Month()       //月
t3:=time.Now().Day()         //日
t4:=time.Now().Hour()        //小时
t5:=time.Now().Minute()      //分钟
t6:=time.Now().Second()      //秒
t7:=time.Now().Nanosecond()  //纳秒
currentTimeData:=time.Date(t1,t2,t3,t4,t5,t6,t7,time.Local) //获取当前时间，返回当前时间Time     
```
#### 获取当前时间戳
```
timeUnix:=time.Now().Unix()            //单位s,打印结果:1491888244
timeUnixNano:=time.Now().UnixNano()  //单位纳秒,打印结果：
```
#### 获取当前时间的字符串格式
```
timeStr:=time.Now().Format("2006-01-02 15:04:05") 
```
####  时间戳转时间字符串 (int64 —>  string)
```
timeUnix:=time.Now().Unix()   //已知的时间
formatTimeStr:=time.Unix(timeUnix,0).Format("2006-01-02 15:04:05")
```
#### 时间字符串转时间(string  —>  Time)
```
 formatTimeStr="2017-04-11 13:33:37"
 formatTime,err:=time.Parse("2006-01-02 15:04:05",formatTimeStr)
```
#### 时间字符串转时间戳 (string —>  int64)
```
formatTimeStr="2017-04-11 13:33:37"
formatTime,err:=time.Parse("2006-01-02 15:04:05",formatTimeStr)
formatTime.Unix()
```
### 最后
以上的包是在Go1.7引入之前经常使用，在1.7引入后，自带的Context包，即之前讲过的上下文能够很好的进行代替使用方法如下

**设置数据**
```
userContext:=context.WithValue(context.Background(),"user","wuyazi")
ageContext:=context.WithValue(userContext,"age",21)
rContext:=r.WithContext(ageContext)
```
**获得数据**
```
user:=r.Context().Value("user").(string)
age:=r.Context().Value("age").(int)
```
### 推荐阅读
- [Go Context深入学习笔记](https://mp.weixin.qq.com/s?__biz=MzA4NTIyOTI3NQ==&mid=2247483766&idx=2&sn=79e210c44ff42f0f229e2f078427b8e0&chksm=9fda6ac2a8ade3d445a242fcac527547b80be54116b047f04e789b340400d32eb6cc030ed15a&token=1060988883&lang=zh_CN#rd)
- [基于Nginx和Consul构建高可用及自动发现的Docker服务架构](https://mp.weixin.qq.com/s?__biz=MzA4NTIyOTI3NQ==&mid=2247483755&idx=2&sn=829d984edc9658efa53787a8b3f6f7c3&chksm=9fda6adfa8ade3c936eab878666a890f7f432bcedd5537839db36a523b41e5a214e11d5449b5&token=1060988883&lang=zh_CN#rd)
- [关于log日志的深入学习笔记](https://mp.weixin.qq.com/s?__biz=MzA4NTIyOTI3NQ==&mid=2247483739&idx=2&sn=85786a5c43ebdba3fb040a75475d18ec&chksm=9fda6aefa8ade3f9b60e7ad8b025e33516f5e9b07b8ac7f4ad9defc0050c619ac54b547f9236&token=1060988883&lang=zh_CN#rd)

----------------------------------------------------------------------------------------------- 
本文欢迎转载，转载请联系作者，谢谢！
公众号【常更新】：无崖子天下无敌
GitHub：[https://github.com/yuwe1](https://github.com/yuwe1)
博客地址【不定期更新】：[https://yuwe1.github.io/](https://yuwe1.github.io/)

-----------------------------------------------------------------------------------------------                                                                            
![打开微信扫一扫，关注微信公众号](http://pvl456y5x.bkt.clouddn.com/FgUj0g5T2WhzsfGzk2BLtopzd_pz)
