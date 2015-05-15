title: "elasticsearch模板"
date: 2015-05-06 11:01:18
tags:
---

在日志分析系列产品中，我们将ES主要用来做结果存储查询使用，而日志统计结果都是一些固定的字段内容，不需要使用到其分词功能。ES的默认分词会导致使用term接口查询domain时，无法匹配整个url的情况。我们使用  

> curl -XGET http://esserver:9200/indexname/\_analyze?text=domain  

可以查询到domain被分词解析成了几个字段。  

在ES的自动创建索引过程中，会去匹配模板，来生成索引。如果我们设置好默认模板，就可以控制自动创建索引的过程。我们主要目的是将String类型增加not\_analyzed属性，如此ES在创建索引过程中就不会对String类型进行分词。  

		{
			"template" : "*",
			"settings" : {
				"index.number_of_shards" : 5,
				"number_of_replicas" : 1,
				"index" : {
					"query" : { "default_field" : "message"},
					"store" : { "compress" : { "stored" : true, "tv": true } }
				}
			},
			
			"mappings": {
				"_default_": { 
					"_all": { "enabled": true },
					"_source": { "compress": true },
					"dynamic_templates": [
					{
						"string_template" : { 
							"match" : "*",
							"mapping": { "type": "string", "index": "not_analyzed" },
							"match_mapping_type" : "string"
						} 
					}
					]
				}
			}
		}  

其中tempate表示匹配的索引的type，\*表示所有的索引，支持正则表达式。  


我们使用  

> curl -XPUT http://esserver:9200/\_template/new\_template\_name 'template content'  

就可以新建一个索引模板，新建的索引都会默认读取该模板配置。  
