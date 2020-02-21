# Hadoop-cos-DistChecker



## 功能说明



Hadoop-cos-DistChecker 是一个在使用`hadoop distcp`命令从HDFS迁移数据到COS上后，用于校验迁移目录完整性的工具。基于MapReduce的并行能力，可以快速地进行迁移源目录和目的目录的校验比对。



## 前置依赖


- [hadoop-cos-2.x.x-shaded.jar](https://github.com/tencentyun/hadoop-cos/tree/master/dep)

- Hadooop MapReduce的运行环境

NOTE：这里hadoop-cos依赖需要选择最新版本（GitHub Tag为5.8.2以上）才能支持CRC64校验码的获取。

## 使用说明

### 参数概念

#### 源文件路径列表

源文件列表是用户使用`hadoop fs -ls -R hdfs://host:port/{source_dir} | awk '{print $8}' > check_list.txt`导出待检查的子目录和文件列表。示例格式如下：

```txt
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_0
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_1
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_0
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_1
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/_SUCCESS
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/part-00000

```


#### 源目录

源文件列表所在的目录，这个目录通常也是`distcp`命令进行目录级别迁移时的源路径。例如，`hadoop distcp hdfs://host:port/source_dir cosn://bucket-appid/dest_dir`，则`hdfs://host:port/source_dir`为源目录。

这个路径也是源文件路径列表中公共父目录，例如：上述的源文件列表的公共父目录就是：`/benchmarks`


#### 目的目录

待比较目的目录。

#### 源路径和目的路径规则

目的路径：由目的目录的绝对路径${TARGET_WORK_DIR_PATH}加上源文件的相对路径${SOURCE_FILE_RELATIVE_PATH}拼接得到，即${TARGET_WORK_DIR_PATH}/${SOURCE_FILE_RELATIVE_PATH}。

源文件的相对路径${SOURCE_FILE_RELATIVE_PATH}则是由源文件的绝对路径去除源目录后得到。


### 命令行格式

hadoop-cos-distchecker是一个MapReduce作业程序，按照MapReduce作业的提交流程运行即可：

```shell
hadoop jar hadoop-cos-distchecker-2.8.5-1.0-SNAPSHOT.jar com.qcloud.cos.hadoop.distchecker.App <源文件列表的绝对路径> <源目录的绝对路径表示> <目的目录的绝对路径表示> [Hadoop 可选参数]

```

### 使用步骤

下面以校验` hdfs://10.0.0.3:9000/benchmarks`和`cosn://hdfs-test-1250000000/benchmarks`为例，介绍工具的使用步骤。


首先，执行`hadoop fs -ls -R hdfs://10.0.0.3:9000/benchmarks | awk '{print $8}' > check_list.txt`，将待检查源路径导出到一个check_list.txt的文件中，这个文件里面保存的就是源文件路径列表了：

![导出文件列表](resources/导出文件列表.PNG)


![文件列表](resources/文件列表.PNG)


然后，将check_list.txt放到HDFS中：`hadoop fs -put check_list.txt hdfs://10.0.0.3:9000/`；



![](resources/将check_list放到HDFS.PNG)





最后，执行Hadoop-cos-DistChecker，将hdfs://10.0.0.3:9000/benchmarks://hdfs-test-1250000000/benchmarks进行对比，然后输出结果保存到cosn://hdfs-test-1250000000/check_result路径下，命令格式如下：



```shell
hadoop jar hadoop-cos-distchecker-2.8.5-1.0-SNAPSHOT.jar com.qcloud.cos.hadoop.distchecker.App hdfs://10.0.0.3:9000/check_list.txt hdfs://10.0.0.3:9000/benchmarks cosn://hdfs-test-1250000000/benchmarks cosn://hdfs-test-1250000000/check_result

```

![运行过程](resources/运行过程.PNG)

distchecker会读取源文件列表和源目录执行MapReduce作业，进行分布式地检查，最后的检查报告会输出到`cosn:///hdfs-test-1250000000/check_result`路径下。

![检查结果的输出路径](resources/检查结果.PNG)


检查报告如下：

```text
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO,None,None,None,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_control,None,None,None,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_0	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_0,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_control/in_file_test_io_0,MD5,dee27f089393936ef42dbd3ebd85750b,dee27f089393936ef42dbd3ebd85750b,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_1	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_control/in_file_test_io_1,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_control/in_file_test_io_1,MD5,526560d99bd99476e5a8e68f0ce87326,526560d99bd99476e5a8e68f0ce87326,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_data,None,None,None,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_0	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_0,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_data/test_io_0,CRC64,-1057373059199797567,-1057373059199797567,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_1	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_data/test_io_1,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_data/test_io_1,CRC64,-1057373059199797567,-1057373059199797567,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_write,None,None,None,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/_SUCCESS	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/_SUCCESS,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_write/_SUCCESS,MD5,d41d8cd98f00b204e9800998ecf8427e,d41d8cd98f00b204e9800998ecf8427e,SUCCESS,'The source file and the target file are the same.'
hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/part-00000	hdfs://10.0.0.3:9000/benchmarks/TestDFSIO/io_write/part-00000,cosn://hdfs-test-1250000000/benchmarks/TestDFSIO/io_write/part-00000,MD5,5f91c70529f8c9974bf7730c024c867f,5f91c70529f8c9974bf7730c024c867f,SUCCESS,'The source file and the target file are the same.'


```



## 检查报告格式


检查报告是以如下格式展示：

```TEXT
check_list.txt中的源文件路径 源文件绝对路径,目的文件绝对路径,Checksum算法,源文件的checksum值,目的文件的checksum值,检查结果,检查结果描述

```

其中检查结果分为以下7种：

- SUCCESS：表示源文件和目的文件都存在，且一致；
- MISMATCH：表示源文件和目的文件都存在，但不一致；
- UNCONFIRM：无法确认源文件和目的文件是否一致，这种状态主要是由于COS上的文件可能是CRC64校验码特性上线前就存在的文件，无法获取到其CRC64的校验值
- UNCHECKED：未检查。这种状态主要是由于源文件无法读取或无法源文件的checksum值
- SOURCE_FILE_MISSING：源文件不存在
- TARGET_FILE_MISSING：目的文件不存在
- TARGET_FILESYSTEM_ERROR：目的文件系统不是CosN文件系统；


## 运行性能

由于工具默认会使用输入的源文件列表路径作为InputPath，因此，若这个文件长度未达到map的split size阈值，则只会启动一个map做源路径的顺序校验。此时，若需要利用MapReduce的并行能力，可以适当调整MapReduce的`mapreduce.input.fileinputformat.split.minsize` 和 `mapreduce.input.fileinputformat.split.maxsize`两个运行参数，使得源文件路径列表能够被切分成多片。


## FAQ

1.**为什么检查报告的CRC64值出现负数？**

因为CRC64值，有可能是20位的值，此时已超过Java long型的表示范围，但是其底层字节是一致的，而打印long型时，会出现负数表示；

2.**为什么既有MD5值校验又有CRC64值校验？**

目前,COS的简单上传依然使用MD5值作为校验码，而CRC64校验码只用于分块上传文件的校验码。因此，这里针对两种不同类型的文件，使用的是不同的校验码来检查。