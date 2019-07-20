# weather

## 背景

最近在计划明年从北京rebase到深圳去，所以最近在看深圳的各个方面。去年在深圳呆过一段时间，印象最深的是，深圳总是突然就下雨，还下好大的雨。对于我这种从小在南方长大但是后面又在北京呆了2年多的人来说，熟悉而又无奈。

今天早上本来想随便浏览浏览一个天气网站，看看深圳的历史天气如何的，但是，一不小心发现，这家网站竟然直接能用API来抓数据，这~~~还不抓一波，省的自己一个月一个月地看。

先上最后的效果图：

![深圳](https://github.com/boydfd/pictures/raw/master/weather/shenzhen.jpg)

所有的code都在我的GitHub上：[boydfd](https://github.com/boydfd/weather)

下面从几个方面讲一讲我是怎么做的：

1. 爬取数据
2. 用pandas显示数据
3. 功能扩展
4. 遇到的坑

## 爬取数据

先是在http://tianqi.2345.com上面浏览了一下深圳的6月份天气。然后发现点切换月份的时候，网址没有变，那应该有请求API吧，看看这个API长啥样吧。
![change month](https://github.com/boydfd/pictures/raw/master/weather/change_month.png)

![change month api](https://github.com/boydfd/pictures/raw/master/weather/change_month_http.png)

发现返回值就是纯JS代码，那就解析一下吧：

1. 去掉`var =`和最后的`;`。
2. 用到`demjson`解析成Python的`List[Dict]`对象。
3. 转成pandas的`DataFrame`
4. 加上我们的date字段

```python
date = '201905'
weather = requests.get('http://tianqi.2345.com/t/wea_history/js/{date}/59493_{date}.js'.format(date=date)).text.split('=')[1][:-1]
weather = demjson.decode(weather)['tqInfo']
pd.DataFrame(weather)
df['month'] = date
```

结果是这样的：

![show weather](https://github.com/boydfd/pictures/raw/master/weather/show_weather_01.png)

## 用Pandas显示数据

### 太多雨天

我们可以看到，有各种雷阵雨啊，阴转雨啊，雨转阴之类的，这样看到的天气太杂了，所以我就统一了一下，按照雨、多云、阴、晴的顺序来排序，先出出现的关键词优先级更高。

写一个函数来处理之：

```python
rain = '雨'
rain_index = ' ' + rain
cloudy = '多云'
cloudy_index = ' ' + cloudy
overcast = '阴'
overcast_index = ' ' + overcast
sunny = '晴'
sunny_index = ' ' + sunny
def weath_category(row):
    tianqi = row['tianqi']
    if tianqi.find(rain) != -1:
        return rain_index
    if tianqi.find(overcast) != -1:
        return overcast_index
    if tianqi.find(cloudy) != -1:
        return cloudy_index
    return sunny_index
```

### 多个月的数据

一个月的数据不够啊，我们想要很多个月的数据，那就写得函数来生成月份吧。

```python
def date_generate(start, end):
    start = datetime.strptime(start, '%Y%m')
    end = datetime.strptime(end, '%Y%m')
    while True:
        next_start = start + relativedelta(months=1)
        yield start.strftime('%Y%m')
        if next_start > end:
            break
        start = next_start
```

### 画图

分好类，爬了多个月份的数据，就剩最终的画图部分了。使用Pandas提供给我们的函数，可以很容易就画出图来。

```python
def plot_weather(start, end):
    df = read_weather(start, end).dropna().reset_index()
    df['weather'] = df.apply(weath_category, axis=1)
    
    from pylab import rcParams
    rcParams['figure.figsize'] = 40, 10
    weather_df = df.groupby(['month', 'weather']).aqi.count().unstack().reset_index()
    weather_df.plot.bar(x='month', y=[rain_index, overcast_index, cloudy_index, sunny_index])
```
![shenzhen](https://github.com/boydfd/pictures/raw/master/weather/shenzhen.png)
## 功能扩展

现在只能收集到一个月的数据，想收集多个月的数据，还都自己去页面上找城市代表的code是啥，太低效了。

这个网站这么容易爬，那就再试试能不能找到调用code的API。

啊哦，一不小心找到了所有的code，哈哈哈。
![change city](https://github.com/boydfd/pictures/raw/master/weather/change_city.png)


![change city http](https://github.com/boydfd/pictures/raw/master/weather/change_city_list.png)

那就在JS里面提取一下。

1. 先把所有的JS代码都复制到浏览器的console里, 结果长这样：

![console](https://github.com/boydfd/pictures/raw/master/weather/code_console1.png)

2. 将其转换成字符串。

```javascript 1.8
provqx.flatMap(a => a).join('|')
```

3. 在Python里处理它。

```python
def line_to_city_code(line):
    return line.split(' ')[1].split('-')

def get_city_to_code():
    city_code_list = list(map(line_to_city_code, city_code.split('|')))
    return {city_code[0]: city_code[1] for city_code in city_code_list if len(city_code) == 2}
```

这样我们就拿到所有的code了，只需要输入城市，开始时间，结束时间，一张漂亮的图就出来了,我还写了个类稍微封装了一下，只需要这样就能使用了：

```python
Weather('深圳').plot_weather('201701', '201906')
```

## 遇到的坑
以前在电脑里面处理过一次，就是matplotlib画图中文乱码的事情，这次换了新电脑又碰到了。所以又搞了一次，

大概的步骤可以参考https://www.jianshu.com/p/8ed59ac76c06

我为了以防下次再经历一次，就写了个脚本自动处理这件事，目前只支持macOS和Python3。
脚本也在我的GitHub：[bash]()，

直接执行下面的bash脚本就可以解决这个问题：

```bash
curl -o- https://raw.githubusercontent.com/boydfd/one_step_solve/master/matplotlib_chinese.sh | bash
```
