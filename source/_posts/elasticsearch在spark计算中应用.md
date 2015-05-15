title: "elasticsearch在spark计算中应用"
date: 2015-04-30 14:02:30
tags:
---
在大数据量的计算任务中，结果展示是一个很重要的环节，普通的hbase存储结果，当结果数据量较大时，scan结合filter的方式的查询效率就不能忍受了。我们选用了elasticsearch的搜索引擎解决方案来解决问题，其和solr类似，我们使用其本地磁盘做数据存储的模式，插入时创建索引完成数据结果的实时查询。  

#整体架构#
在实际产品中，我们使用spark streaming消费kafka的数据，做1分钟数据计算。计算完的数据结果，会存储两份,  

> 其中一份丢hbase，以用作离线计算5分钟和1个小时的计算任务数据源。  
> 另一份结果直接插入ES(elasticsearch)中，提供给1分钟webUI查询结果。  
  
因为我们的数据结构满足结合律，所以1分钟的数据结果可以用来复用，减少5分钟和1个小时任务的计算量。由于我们的日志存在超时情况，所以选用hbase是为了给后续的月份报表，年报表时增加超时日志计算来用。但某些需求需要5分钟或1个小时的计算任务需要保证数据准确性，我们在hbase中新增一个表用来存储当天的超时日志，每日添加一个超时计算任务来完成超时日志的计算和结果追加功能。

#Spark中使用#
elasticsearch官方提供spark的API，我们主要使用其写ES并自动创建索引功能。  

		val conf = new SparkConf()
		.set("es.index.auto.create", "true")
		.set("es.node", es_node_ip)

		val ssc = new StreamingContext(conf, Seconds(30))

		val kafka_config = Map(
			"zookeeper.connect" -> zk_quorum,
			"group.id" -> group_id
		)

		val receive_stream = (1 to num_node).map { 
			i => {
				val topic = Map(source_kafka_topic -> 1)
				val kafka_stream = KafkaUtils.createStream[String, String, StringDecoder,   
					StringDecoder] (ssc, kafka_config, topic, StorageLevel.MEMORY_ONLY_SER).map(_._2)
				kafka_stream
			}
		}
			

		val lines = ssc.union(receive_stream)
		
		val store_es_task = lines.map(rdd => {
			val line_fields = rdd.split("\t")
			val log_hash = Map("log_type" -> line_fields(0).toInt,
								"timestamp" -> line_fields(1).toLong,
								"request_num" -> line_fields(2).toInt,
								"domain" -> line_fields(3),
								"cost_time" -> line_fields(4).toInt)
			log_hash
		}).foreachRDD(rdd => {
			rdd.saveToEs("http_request_log/" + (System.currentTimeMillis() / 1000 / 60 / 60 / 24))
		})
此段代码完成了从kafka中读取数据，并解析，将结果存入es中的全过程。我们要注意spark的ES接口提供自动索引功能，上例中，ES中会自动创建log_type, timestamp, request_num, domain, cost_time这几个索引，其类型为第一条数据插入时的数据格式类型。如果你对后续的插入类型有修改，需要将之前创建的索引数据删除掉。saveToEs的参数为ES的index和type字段，index类似普通数据库的概念，type是表或者分区的概念，不知道id的话，其会自动生成id，和rowkey概念一致。index,type,id三个参数都可以自动生成，为了优化搜索，建议根据业务进行相应数据分区，因为查询时加载的数据源即指定index和type。  


插入到ES后，我们可以使用crul -XGET/XDELETE/XPUT等命令来完成对ES的命令行查询工作，也可以使用ES提供的java相关API来完成查询。


#ES原理篇#
由于刚开始接触，仅仅对应用比较熟悉，相关原理可以查阅<http://shgy.gitbooks.io/mastering-elasticsearch/content/chapter-1/README.html>
