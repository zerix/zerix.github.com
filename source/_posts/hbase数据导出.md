title: "hbase数据导出"
date: 2015-04-22 14:25:11
tags:
---

在hbase shell中，scan功能相对强大，rowkey设计的好，可以拿到自己想要的数据。但是其不支持直接重定向到文件中。而使用诸于hbase的mr export工具不能做到scan的强大，由于客户要求，干脆自己动手写了一个。

主要原理是：  
**编写一个spark离线计算任务，使用spark读取hbase数据功能，模拟hbase shell scan的相关命令。支持两种任务类型：1.直接扫描表数据，存储结果.	2.支持扫描结果按表中任意字段归并，并以之为文件名，存储结果.数据导出的格式为每个字段scan的顺序，以逗号为分隔符，所以可以使用excel打开**


具体实现已经在[github](https://github.com/zerix/export_hbase_table)开源,对某些同学用spark读取hbase或归类信息写文件或spark常见配置有参考意义。


这里主要对多文件存储方式做一下讲解，在spark/hadoop中，最后的文件存储由于reduce的数量不止一个，或者spark里面的partitions数量比较多，默认存储的文件会按分成相应的part，其数量和reducer及partitions的数量一致。而使用MultipleTextOutputFormat会默认生成1个文件（使用FSDataOutputStream也可以生成一份文件）。我们探探其实现背后的原理。


MultipleTextOutputFormat调用TextOutputFormat类，TextOutputFormat调用TextOutputFormat.getRecordWriter覆盖基类的RecordWriter函数,其基类为MultipleOutputFormat。在MultipleOutputFormat中，使用TreeMap缓存输出到不同文件的RecordWriter句柄。在TextoutputFormat.getRecordWriter中，返回LineRecordWriter函数，LineRecordWriter调用DataOutputStream作为其输出，使用DataOutputStream.write函数写入到hdfs文件系统中。 


而对于使用者来说，想要输出文件只有一个，只需要继承MultipleTextOutputFormat并覆盖函数名函数即可。
