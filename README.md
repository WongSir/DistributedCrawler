DistributedCrawler
=================

# 简介 #

这是一个基于Hadoop的分布式爬虫，目前只支持抓取腾讯新闻中心的新闻内容。支持插件机制，可以通过实现Extractor接口自己编写插件已实现对于各种网站的抓取和内容提取。

# 使用方法 #

1. 在com.gs.main.Test1中设置爬取时需要的各种参数.

	```Java
	private static final String rootPath = "hdfs://gs-pc:9000/home/test/";
	private static final String dst = rootPath + "qq.txt";// 首页的链接暂存地
	private static final int depth = 3;// 深度
	private static final int topN = 80;// 每页抓取的最大链接数
	private static final String outputPath = rootPath + "output";// 最终结果的输出路径
	private static final String jobName = "DistributeCrawler";// Job的名称
	private static final String crawlDBHost = "localhost";//CrawlDB的IP
	private static final int crawlDBPost = 6377;//CrawlDB的端口
	private static final String crawlDBPassword = "******";//CrawlDB的密码
	private static final int crawlDBTimeout = 1000;//CrawlDB的连接超时时间
	private static final int crawlDBToCrawlDB = 0;//CrawlDB的待抓取的数据库编号
	private static final int crawlDBCrawledDB = 1;//CrawlDB的已抓取的数据库编号
	private static final String SolrURL = "http://localhost:8888/solr";//Solr服务器URL
	private static final Logger LOG = LoggerFactory.getLogger(MainClass.class);
	```

2. 启动带有Solr的Tomcat服务器.详细请参看: [tomcat部署solr步骤](http://blog.csdn.net/gsh19940409/article/details/17886567) | [通过Solr4.0.0实现分布式索引](http://blog.csdn.net/gsh19940409/article/details/17915223)

3. 启动Redis服务器,然后FlushAll一下,免得有上次剩下的URL.

# 抓取流程 #

1. 然后通过HDFS的本地计算将一个文件分发到各台Slaves上面。
2. 然后各台Slave机器调用map方法开始抓取,抓取之前先从CrawlDB中索取一些连接,默认数量设为20。数量的设置在`Crawler`里面的`maxGenerate`.若未达到topN和depth限制则将本页面的连接Inject到CrawlDB中。每一个页面抓取完之后转换成json格式都写入context中.
3. 每个页面通过`HttpClient`下载完成后，通过正则表达式进行过滤，只保留正文和标题，同时将这个页面封装为一个`PagePOJO`的POJO。这个POJO包含了正文，标题，URL，该页面的ID号（ID号是通过种子地址在种子文件中的偏移量也就是我们所谓的当前Slaves机器的ID号+一个叠加的计数器，即`ID=CrawlerID+counter`）。
4. 然后将这个POJO交给Json生成器来生成一个Json内容。
5. 最后若当前抓取器队列为空且CrawlDB为空的时候该抓取器退出并返回本次所有抓取的PagePOJO的Set.在后在map方法中通过Solr服务器将其索引.

# 索引流程 #

本项目提供两种索引方式,可在抓取的时候同时进行分布式索引,也可等抓取结束后通过生成的Json格式的数据文件索引.

## 离线索引 ##

1. 首先通过`JsonReader`一行一行的读出json内容（每一行是一个Json表达式）。`JsonReader`是通过`RandomAccessFile`来实现的。因为他既可以满足从`InputStream`中按行读入的要求，还带有获取当前偏移量和skip方法，极为的好用。`JsonReader`返回一个`Hit`类型的封装。`Hit`封装了PagePOJO、文件名和在此文件中的起始偏移量。
2. 提取出每一个`Hit`里面的正文、文件名和偏移量，并用`Lucene`索引。不储存content，但是储存文件名和起始偏移量。这样就可以摆脱对数据库的依赖。

## 分布式索引 ##

- 分布式索引通过`Lucene`其下的一个子项目`Solr`实现.具体由`com.gs.indexer.solr.SolrIndex`负责.

# 贝叶斯分类器 #

- 在搜索方法中加入了分类的逻辑，如果搜索时在参数中声明需要分类
，则通过贝叶斯方法进行分类。
- 使用贝叶斯分类器的时候，要进行参数设置，并通过`MapMaker`生成每一个类的map，即训练分类器。

## 训练贝叶斯分类器 ##

1. 将训练集生成为`TrainingDataManager`识别放map格式。
2. 调用`MapMaker`的make方法，传入训练集的根目录。会在每一个分类的目录下生成一个map文件，里面是存储当前分类的每一个词的词频。**注意：如果训练集的文本不是txt或者TXT格式的话需要在`make`方法里面设置文件类型。**
3. 在`StopWordsHandler`里面设置停用词的路径。
4. 在`TrainingDataManager`中设置训练好的map的路径，也就是训练集的根路径。
5. `BayesClassifier`就是分类器的主控类，里面有一个`zoomFactor`参数的设置，这个主要是保证在训练集大小不一样的情况下保证分类质量。即当通过贝叶斯公式算出的概率太小或者太大时，调节这个参数可使得算出的概率数值在可控范围之内，一旦数值过小（全都为0）或者过大（显示无限大）是每一个分类的概率都是一样的，这样分类之后各个类就无从排序，分的类也就没有任何意义。更换训练集之后，一定记得查看各个分类的概率数值，调整`zoomFactor`。
6. 都设置好之后调用`BayesClassifier`的`classify`方法传入待分类的文本，返回值就是`Strng`类型的分类名。注意：`BayesClassifier`是单例模式不可直接构造需要调用`getInstance`方法。

# 使用Carrot2插件进行聚类 #

在搜索模块包含了一个聚类插件,是通过Carrot2实现的,具体使用方法如下.

## 对所有的文档进行聚类 ##

调用`com.gs.cluster.Cluster`这个类的cluster方法,并传入由Crawler爬取好的Json格式的文档的的路径,即可返回一个`ProcessingResult`类型的Result.`ProcessingResult`里面的内容查看方法请参考[Carrot2文档](http://download.carrot2.org/stable/javadoc/).

## 对指定的文档进行聚类 ##

这个方法适用于搜索时候对于搜索的结果进行聚类.搜索完毕之后将索引的`PagePOJO`封装成一个List传给cluster,聚类完毕之后,返回一个类型为`Map<String,List<String>>`的result.`Map<String,List<String>>`.这个map的key存放的是类型的名称,value的List里面存放的是该类型包含的所有文档的标题.

# WindowSearcher #

这是一个由JavaSwing实现的界面化的搜索工具.

## 站点测试 ##

这个Pane是用来测试实验室的几个网站是否正常.

## Index ##

这个Pane是用来对Hadoop爬虫处理好的Json格式的新闻内容进行索引的工具.主要通过Lucene实现索引.在Json框中输入文件所在的文件夹的位置,index框中输入需要存放index文件的文件夹的路径.

# 下一步的工作 #

- 利用Redis建立一个像Nutch一样的CrawlDB,通过维护CrawlDB来实现分布式的抓取和任务调度.
- 搭建一个分布式的Solr服务器集群
- 通过Solr实现分布式的索引和搜索服务

