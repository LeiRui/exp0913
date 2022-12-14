# TsFile读耗时分解实验

## 一、实验设置

### 硬件

- FIT楼166.111.130.101 / 192.168.130.31
- CPU：Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz，(6核12线程)
- L1 cache 284KB, L2 cache 1536KB, L3 cache 12MB
- 内存：16G
- 硬盘：1.8T HDD /dev/sdb1 mounted on /disk
- 操作系统：Ubuntu 16.04.7 LTS
- 工作文件夹：/disk/rl/tsfileReadExp/

### 数据集

- 中车数据：/disk/rl/zc_data
- 人工数据：时间戳从1开始以步长1递增，值在[0,100)随机取整数

### 实验代码

- 测试对象：IoTDBv0.13.1
- 可执行jar包和脚本：https://github.com/LeiRui/exp0913.git
- 实验源码：https://github.com/LeiRui/iotdb/blob/cost2/tsfile/src/main/java/org/apache/iotdb/tsfile/read/RLTsFileReadCostBench.java

#### RLTsFileReadCostBench用法

##### 1. 用人工数据写TsFile

```shell
java -jar RLTsFileReadCostBench-0.13.1-jar-with-dependencies.jar WRITE_SYN [pagePointNum] [numOfPagesInChunk] [chunksWritten] [timeEncoding] [valueDataType] [valueEncoding] [compressionType]
```

- `WRITE_SYN`：用来标识是“写人工数据/写真实数据/读数据”中的“写人工数据”
- `pagePointNum(ppn)`：一个page内的点数
- `numOfPagesInChunk(pic)`：一个chunk内的pages数
- `chunksWritten(cw)`：写的chunks总数
- `timeEncoding(te)`：时间戳列编码方式
- `valueDataType(vt)`：值列数据类型
- `valueEncoding(ve)`：值列编码方式
- `compressionType(co)`：压缩方式

##### 2. 用真实数据集写TsFile

```shell
java -jar RLTsFileReadCostBench-0.13.1-jar-with-dependencies.jar WRITE_REAL [path_of_real_data_csv_to_write] [pagePointNum] [numOfPagesInChunk] [timeEncoding] [valueDataType] [valueEncoding] [compressionType]
```

- `WRITE_REAL`：用来标识是“写人工数据/写真实数据/读数据”中的“写真实数据”
- `path_of_real_data_csv_to_write`：用来写TsFile的真实数据集csv地址
- `pagePointNum(ppn)`：一个page内的点数
- `numOfPagesInChunk(pic)`：一个chunk内的pages数
- `timeEncoding(te)`：时间戳列编码方式
- `valueDataType(vt)`：值列数据类型
- `valueEncoding(ve)`：值列编码方式
- `compressionType(co)`：压缩方式

##### 3. 读实验

```shell
java -jar RLTsFileReadCostBench-0.13.1-jar-with-dependencies.jar READ [path_of_tsfile_to_read] [decomposeMeasureTime] [D_decompose_each_step] (timeEncoding)
```

- `WRITE_REAL`：用来标识是“写人工数据/写真实数据/读数据”中的“读数据”
- `path_of_tsfile_to_read`：要读取的TsFile地址
- `decomposeMeasureTime`：`FALSE` to measure the read process as a whole, in which case `D_decompose_each_step` is useless. `TRUE` to measure the decomposed read process, and the decomposition granularity is controlled by `D_decompose_each_step`.
- `D_decompose_each_step`：When `decomposeMeasureTime` is `TRUE`, `D_decompose_each_step=FALSE` to measure the "(D_1)decompress_pageData" and "(D_2)decode_pageData" steps without further deomposition, `D_decompose_each_step=TRUE` to break down these two steps further and measure substeps inside.
- `timeEncoding(te)`：If `timeEncoding` is not specified, TS_2DIFF will be used by default. `timeEncoding` should be the same with that used to write the TsFile.



| 控制参数         | decomposeMeasureTime=FALSE | decomposeMeasureTime=TRUE, D_decompose_each_step=FALSE       | decomposeMeasureTime=TRUE, D_decompose_each_step=TRUE        |
| ---------------- | -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 测量最小单元步骤 | total_time                 | (A)1_index_read_deserialize_MagicString_FileMetadataSize(us)<br />(A)2_index_read_deserialize_IndexRootNode_MetaOffset_BloomFilter(us)<br />(A)3_2_index_read_deserialize_IndexRootNode_exclude_to_TimeseriesMetadata_forExactGet(us)<br />(B)4_data_read_deserialize_ChunkHeader(us)<br />(B)5_data_read_ChunkData(us)<br />(C)6_data_deserialize_PageHeader(us)<br/>(D-1)7_data_decompress_PageData(us)<br/>(D-2)8_data_decode_PageData(us) | (A)1_index_read_deserialize_MagicString_FileMetadataSize(us)<br/>(A)2_index_read_deserialize_IndexRootNode_MetaOffset_BloomFilter(us)<br/>(A)3_2_index_read_deserialize_IndexRootNode_exclude_to_TimeseriesMetadata_forExactGet(us)<br/>(B)4_data_read_deserialize_ChunkHeader(us)<br/>(B)5_data_read_ChunkData(us)<br/>(C)6_data_deserialize_PageHeader(us)<br/>(D-1)7_1_data_ByteBuffer_to_ByteArray(us)<br/>(D-1)7_2_data_decompress_PageDataByteArray(us)<br/>(D-1)7_3_data_ByteArray_to_ByteBuffer(us)<br/>(D-1)7_4_data_split_time_value_Buffer(us)<br/>(D-2)8_1_createBatchData(us)<br/>(D-2)8_2_timeDecoder_hasNext(us)<br/>(D-2)8_3_timeDecoder_readLong(us)<br/>(D-2)8_4_valueDecoder_read(us)<br/>(D-2)8_5_checkValueSatisfyOrNot(us)<br/>(D-2)8_6_putIntoBatchData(us) |



#### 自动化实验脚本

- RLUnitSynExp.sh：用人工数据写TsFile，清空系统缓存，然后进行一次读TsFile实验。
    - 输入：见RLTsFileReadCostBench的写数据参数和读数据参数
    - 输出：一个TsFile文件、一个TsFile空间统计结果文件（ `*writeResult.csv`）、一个读TsFile耗时结果csv文件（ `*readResult-T*csv`）
- RLUnitRealExp.sh：用真实数据写TsFile，清空系统缓存，然后进行一次读TsFile实验。
    - 输入：见RLTsFileReadCostBench的写数据参数和读数据参数
    - 输出：一个TsFile文件、一个TsFile空间统计结果文件（ `*writeResult.csv`）、一个读TsFile耗时结果csv文件（ `*readResult-T*csv`）

- RLReadExpScripts.sh：重复若干次读实验，把读实验结果进行汇总，把写文件的空间结果和读文件的耗时结果汇总到一起，最后对读实验结果进行平均值和百分比统计计算。
    - 输入：
        - `WRITE_READ_JAR_PATH`：RLTsFileReadCostBench-0.13.1-jar-with-dependencies.jar的地址
        - `Calculator_JAR_PATH`：把若干次重复读实验结果进行平均值和百分比计算的RLRepeatReadResultAvgPercCalculator-0.13.1-jar-with-dependencies.jar的地址
        - `FILE_NAME`：要读取的TsFile的地址
        - `decomposeMeasureTime`：见RLTsFileReadCostBench读数据参数
        - `D_decompose_each_step`：见RLTsFileReadCostBench读数据参数
        - `te`：：见RLTsFileReadCostBench读数据参数
        - `REPEAT`：读实验重复次数
    - 输出：
        - REPEAT个读TsFile耗时结果csv文件 `*readResult-T*csv`
        - 一个把重复读实验结果横向拼接起来的csv文件 `*readResult-combined.csv`
        - 一个把写结果和读结果拼接起来的csv文件 `*allResult-combined.csv`
        - 一个把读结果取平均值并且按照不同粒度统计百分比的csv文件 `*allResult-combined-processed.csv`
- RLCompressionSynExpScripts.sh：在不同的压缩方式参数下（UNCOMPRESSED, SNAPPY, GZIP, LZ4），用人工数据写TsFile，清空系统缓存，然后进行若干次重复读TsFile实验，把读实验结果进行汇总，把写文件的空间结果和读文件的耗时结果汇总到一起，最后对读实验结果进行平均值和百分比统计计算。
    - 输入：
        - 工具地址：
            - `WRITE_READ_JAR_PATH`：RLTsFileReadCostBench-0.13.1-jar-with-dependencies.jar的地址
            - `Calculator_JAR_PATH`：把若干次重复读实验结果进行平均值和百分比统计计算的RLRepeatReadResultAvgPercCalculator-0.13.1-jar-with-dependencies.jar的地址
            - `TOOL_PATH`：用于替换脚本中变量值的自动脚本工具RLtool.sh的地址
            - `READ_SCRIPT_PATH`：RLReadExpScripts.sh的地址
        - 写数据参数：见RLTsFileReadCostBench写数据参数
        - 读数据参数：见RLTsFileReadCostBench读数据参数
        - REPEAT：读实验重复次数
    - 输出：不同压缩方式下的一个TsFile文件、一个TsFile空间统计结果文件（ `*writeResult.csv`）、REPEAT个读TsFile耗时结果csv文件（ `*readResult-T*csv`）、一个把重复读实验结果横向拼接起来的csv文件（`*readResult-combined.csv`）、一个把写结果和读结果拼接起来的csv文件（`*allResult-combined.csv`）、一个把读结果取平均值并且按照不同粒度统计百分比的csv文件（ `*allResult-combined-processed.csv`）



## 二、不同压缩方式的实验

RLCompressionRealExpScripts.sh

RLCompressionSynExpScripts.sh


## 三、不同编码方式的实验

RLValueEncodingRealExpScripts.sh

RLValueEncodingSynExpScripts.sh