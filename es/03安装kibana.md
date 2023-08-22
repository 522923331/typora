Kibana是一个针对Elasticsearch的开源分析及可视化平台，用来搜索、查看交互存储在Elasticsearch索引中的数据。使用Kibana ,可以通过各种图表进行高级数据分析及展示。Kibana让海量数据更容易理解。
它操作简单，基于浏览器的用户界面可以快述创建仪表板( dashboard )实时显示Elasticsearch查询动态。
设置Kibana非常简单,无需编码或者额外的基础架构，几分钟内就可以完成Kibana安装并启动Elasticsearch索引监测。

# 1.下载解压kibana

官网: https://www.elastic.co/cn/kibana

下载好后，进入到bin目录，执行：

```shell
sh ./kibana
#启动后可以看到端口
http server running at http://localhost:5601
```

然后访问 http://localhost:5601

<img src="/Users/yp-23040708/Library/Application Support/typora-user-images/截屏2023-04-24 19.34.59.png" alt="截屏2023-04-24 19.34.59" style="zoom:50%;" />

# 2.kibana开发工具

home菜单栏，management目录下，dev Tools:

![截屏2023-04-24 19.38.05](/Users/yp-23040708/Library/Application Support/typora-user-images/截屏2023-04-24 19.38.05.png)

![截屏2023-04-24 20.05.58](/Users/yp-23040708/Library/Application Support/typora-user-images/截屏2023-04-24 20.05.58.png)

输入 GET /.kibana_8.7.0_001

会返回很多的数据信息

# 3.汉化kibana

修改kibana config 下的kibana.yml

```yaml
# =================== System: Other ===================
# The path where Kibana stores persistent data not saved in Elasticsearch. Defaults to data
#path.data: data

# Specifies the path where Kibana creates the process ID file.
#pid.file: /run/kibana/kibana.pid

# Set the interval in milliseconds to sample system and process performance
# metrics. Minimum is 100ms. Defaults to 5000ms.
#ops.interval: 5000

# Specifies locale to be used for all localizable strings, dates and number formats.
# Supported languages are the following: English (default) "en", Chinese "zh-CN", Japanese "ja-JP", French "fr-FR".
#i18n.locale: "en"  默认值
i18n.locale: "zh-CN"  #修改成中文
# =================== Frequently used (Optional)===================
```

再次访问，发现汉化成功了。