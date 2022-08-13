#### 项目地址：

#### https://github.com/NSSL-SJTU/SaTC



#### 构建环境：

```
docker pull smile0304/satc:V1.0  拉取镜像
docker run -it -v local:docker  smile0304/satc:V1.0   启动容器


```



#### 使用：

```
python satc.py -d /home/test -o /home/test/result --ghidra_script=ref2sink_cmdi --ghidra_script=ref2sink_bof --taint_check   扫描文件系统所有二进制文件是否存在漏洞（命令注入和缓冲区溢出）（时间会比较长，还是检查单个比较靠谱）


python satc.py -d /home/test -o /home/test/result --ghidra_script=ref2sink_cmdi -b prog.cgi --taint_check   扫描单个文件是否存在命令注入漏洞


python satc.py -d /home/test -o /home/test/result --ghidra_script=ref2share -b prog.cgi
python satc.py -d /home/test -o /home/test/result --ghidra_script=share2sink --ref2share_result=/home/test/result/ghidra_extract_result/prog.cgi/prog.cgi_ref2share.result -b rc --taint_check

ref2share: 此脚本用来查找输入等字符串中被写入共享函数等参数，例如:nvram_set, setenv等函数。需要与share2sink来配合使用
share2sink: 此脚本与ref2share功能类似。需要与ref2share来配合使用；使用此脚本的输入为ref2share脚本的输出

```



#### 输出

输出结果目录结构:

```
|-- ghidra_extract_result
|   |-- httpd
|       |-- httpd
|       |-- httpd_ref2sink_bof.result
|       |-- httpd_ref2sink_cmdi.result
|       |-- httpd_ref2sink_cmdi.result-alter2
|-- keyword_extract_result
|   |-- detail
|   |   |-- API_detail.result
|   |   |-- API_remove_detail.result
|   |   |-- api_split.result
|   |   |-- Clustering_result_v2.result
|   |   |-- File_detail.result
|   |   |-- from_bin_add_para.result
|   |   |-- from_bin_add_para.result_v2
|   |   |-- Not_Analysise_JS_File.result
|   |   |-- Prar_detail.result
|   |   |-- Prar_remove_detail.result
|   |-- info.txt
|   |-- simple
|       |-- API_simple.result
|       |-- Prar_simple.result
|-- result-httpd-ref2sink_cmdi-ctW8.txt
```

需要关注的输出结果目录:

- 1.keyword_extract_result/detail/Clustering_result_v2.result : 前端关键字在bin中的匹配情况。为`Input Entry Recognition`模块的输入
- 2.ghidra_extract_result/{bin}/* : ghidra脚本的分析结果。为`Input Sensitive Taint Analysise`模块的输入
- 3.result-{bin}-{ghidra_script}-{random}.txt: 污点分析结果

其他文件说明:

```
|-- ghidra_extract_result # ghidra寻找函数调用路径的分析结果, 启用`--ghidra_script`选项会输出该目录
|   |-- httpd # 每个被分析的bin都会生成一个同名文件夹
|       |-- httpd # 被分析的bin
|       |-- httpd_ref2sink_bof.result # 定位bof类型的sink函数路径
|       |-- httpd_ref2sink_cmdi.result # 定位cmdi类型的sink函数路径
|-- keyword_extract_result  # 关键字提取结果
|   |-- detail  # 前端关键字提取结果(详细分析结果)
|   |   |-- API_detail.result # 提取的API详细结果
|   |   |-- API_remove_detail.result # 被过滤掉的API信息
|   |   |-- api_split.result  # 模糊匹配的API结果
|   |   |-- Clustering_result_v2.result # 详细分析结果(不关心其他过程关心此文件即可)
|   |   |-- File_detail.result  # 记录了从单独文件中提取的关键字
|   |   |-- from_bin_add_para.result # 在二进制匹配过程中新增的关键字
|   |   |-- from_bin_add_para.result_v2 # 同上,V2版本
|   |   |-- Not_Analysise_JS_File.result # 未被分析的JS文件
|   |   |-- Prar_detail.result # 提取的Prar详细结果
|   |   |-- Prar_remove_detail.result # 被过滤掉的Prar结果
|   |-- info.txt  # 记录前端关键字提取时间等信息
|   |-- simple  # 前端关键字提取结果, 比较简单
|       |-- API_simple.result # 在全部二进制中出现的全部API名称
|       |-- Prar_simple.result  # 在全部二进制中出现等的全部Prar
|-- result-httpd-ref2sink_cmdi-ctW8.txt # 污点分析结果,启用`--taint-check` 和 `--ghidra_script`选项才会生成该文件



```



#### 总结：

Satc比其他工具的优势在于：不会盲目的去寻找调用了危险函数的路径，而是从前端获取参数，再而从文件系统寻找参数在二进制文件中的调用路径，避免了上层调用为常数或不可控的情况。