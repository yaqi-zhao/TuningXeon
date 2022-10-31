# IAA and AVX512-VBMI Acceleration for ClickHouse on 4th Gen Intel® Xeon® processor


## Introduction

This guide is for users who are already familiar with ClickHouse.  It provides recommendations for configuring hardware and software that will provide the best performance in most situations. However, please note that we rely on the users to carefully consider these settings for their specific scenarios, since ClickHouse can be deployed in multiple ways. 

4th Gen Intel® Xeon® Scalable processors deliver workload-optimized performance with built-in acceleration for AI, encryption, HPC, storage, database systems, and networking. Intel® In-Memory Analytics Accelerator(IAA) is a new built-in accelerator for AI, HPC, Networking, Security, Storage and Analytics. IAA supports (de)compression with the Deflate compression standard described in RFC 1951 and analytics operations.  ClickHouse(https://clickhouse.com/) is a column-oriented database management system (DBMS) for online analytical processing of queries (OLAP). By applying IAA features on CLickHouse the performance will be improved while the CPU utilization becomes less. [Star Schema Benchmark](https://clickhouse.com/docs/en/getting-started/example-datasets/star-schema) is used for benchmark testing. 

Optimization on ClickHouse that can run on Sapphire Rapids including:
 - Deflate compression algorithm implemented by Intel® Query Processing Library including both IAA and AVX512
 - Parquet file decoding by Intel® In-Memory Analytics Accelerator(IAA)
 - ClickHouse filter operator optimized by AVX512_VBMI/AVX512_VBMI2 extensions


### Server Configuration

#### Hardware

The configuration described in this article is based on 4th Generation Intel® Xeon® processor hardware. The server platform, memory, hard drives, and network interface cards can be determined according to your usage requirements.

| Hardware | Model | 
|----------------------------------|------------------------------------|
| Server Platform  | Sapphire Rapids |
| CPU | Intel® SPR E3 stepping, base Frequency 1.6GHz  | 
| BIOS |  EGSDCRB1.86B.0072.D01.2201101353 | 
| Memory | 256GB (16x16GB DRAM 4800 MT/s [4800 MT/s]) | 
| Storage/Disks | 1 * 892G Intel SSDSC2KB960G8 | 

#### Software

| Software | Version |
|------------------|-------------|
| Operating System | CentOS Stream 8 | 
| Kernel | The latest upstream kernel (v6.0+) basically support full functional DSA/IAA host driver| 
| ClickHouse | version 22.9 | 


			
## Hardware Tuning


### Check BIOS configuration

This should be enabled by default in BIOS. Do double-check. This configuration is :
```
EDKII -> Socket Configuration -> IIO Configuration -> Interrupt Remapping Option: No -> Yes
EDKII Menu ->Socket Configuration -> IIO Configuration -> PCIe ENQCMD /ENQCMDS -> Enable
EDKII Menu -> Socket Configuration ->Processor Configuration -> VMX -> Enable
```



### Enable IAA Device

| IAX1.0 | Number |
|------------------|-------------|
| Work Queues | 8 | 
| Engines |8 | 
| Groups | 4 | 

#### Enable/Disable device with <a href="#accel-config">accel-config tool</a> 

Enable device example(enable iax1 and wq1.0).
```
accel-config config-engine iax1/engine1.0 -g 0
accel-config config-wq iax1/wq1.0 -g 0 -s 16 -p 10 -b 1 -t 15 -m shared -y user -n app1 -d user
accel-config enable-device iax1
accel-config enable-wq iax1/wq1.0
```

Disable device example(disable iax1 and wq1.0).

    accel-config disable-wq iax1/wq1.0
    accel-config disable-device iax1


Use the following command to show the enabled device. 
```
accel-config list
```


### Memory Configuration/Settings

No specific workload setting

### Storage/Disk Configuration/Settings

No specific workload setting

### Network Configuration/Setting

No specific workload setting for this topic


## Software Tuning 

Software configuration tuning is essential. From the operating system to ClickHouse configuration settings, they are all designed for general-purpose applications and default settings are rarely tuned for the best performance.

###  ClickHouse with IAA and AVX512_VBMI/AVX512_VBMI2 Architecture

ClickHouse supports multiple storage engines including MergeTree-engine, File-engine, HDFS engine, and S3 engine. During query execution, the data is first read from Storage/Disks and then decompressed or decoded into in-memory ClickHouse columns. Then some operations including filter, aggregator, etc. will be executed according to the query plan. We draw a diagram to show the optimization we did on Sapphire Rapids.



<div align="left">
<img src=./Picture.png width=60%/>
</div>


### ClickHouse Software Tuning 

Deflate compression and parquet file reader optimization are based on <a href="#qpl">Intel® Query Processing Library(QPL)</a> . The index and filter operator are optimized by AVX512_VBMI and AVX512_VBMI2. If you want to enable these features, there are some pre-requisites:

- Default release packages do not support these features, ClickHouse needs to be built on your environment to support AVX512 extensions and the QPL library. Enable the following cmake flags while building ClickHouse:

```
  -DENABLE_QPL=1 
  -DENABLE_QPL_COMPRESSION=1   
  -DENABLE_AVX512=1
```
- DEFLATE_QPL is experimental and can only be used after setting the configuration parameter allow_experimental_codecs=1.

#### 1. Deflate (de)compression using QPL/IAA
This feature is supported after the release version v22.8. ClickHouse Doc has explained how to use DEFLATE_QPL (https://clickhouse.com/docs/en/sql-reference/statements/create/table/#deflate_qpl).
 - You can change the default compression method of a server configuration for the MergeTree-engine family to DEFLATE_QPL (https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings/#server-settings-compression)
  or
 - You can also define the compression method for each column in the CREATE TABLE query (https://clickhouse.com/docs/en/sql-reference/statements/create/table/#column-compression-codecs)


#### 2. Parquet Reader in Arrow using QPL/IAA

This feature is submitted to apache/arrow([#14217](https://github.com/apache/arrow/pull/14217)) but has not been merged yet. Just pick this PR if you want to integrate this feature. Then for Apache [Parquet](https://clickhouse.com/docs/en/sql-reference/formats/#data-format-parquet) format data selection, the bit-packing and RLE decoding work is offloaded to IAA devices.

#### 3. Enable AVX512_VBMI/AVX512_VBMI2 in ClickHouse

This feature is submitted to ClickHouse([#41765](https://github.com/ClickHouse/ClickHouse/pull/41765)) but has not been merged yet. Just pick this PR if you want to utilize the 2 features for accelerating index and filter operation. Dynamic dispatch is already enabled, no extra build flags are required. ClickHouse will use an optimized path if running in Sapphire Rapids platforms.

## Conclusion

INTEL® IN-MEMORY ANALYTICS ACCELERATOR(IAA) and AVX512_VBMI/AVX512_VBMI2 are built-in with Sapphire Rapids, Intel's 4th Gen Xeon Scalable processor. Some optimizations on ClickHouse are based on IAA hardware and AVX512 extensions. Test on our lab environment shows better performance with IAA and AVX512-VBNI accelerators. Here is a detailed performance data of IAA deflate (de)compression as an example for your reference. 


Performance of Deflate (de)compression on ClickHouse shows that:
-  IAA deflate solution provides ~40% better compression and saves corresponding disk IO size compared with widely used LZ4 while having same P95 (average) latency. 
- Clickhouse with IAA deflate (de)compression provides 6% more QPS (queries per second) than LZ4 and 13% better than ZSTD. The below chart shows detailed information. Detailed data are shown in the following chart.
- IAA deflate saves ~20% memory capacity totally after a full table scan

<br> <br/>
<div align = "center">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src=./Picture2.png width=60%>
    <br> 
    <div align = "center" style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Star Schema Benchmark results comparison</div>
</div>

<br> <br/>


## Related Tools and Information 

### <a id="accel-config">accel-config tool</a> 

Accel-config is a utility library for controlling and configuring IAA (In-Memory Analytics Accelerator) sub-system in the Linux kernel.  Please use "accel-config -h" to check if the tool works. If not, please git clone https://github.com/intel/idxd-config.git and follow README to build and install it.

## References 
1. The <a id="qpl">Intel® Query Processing Library (Intel® QPL) </a> is an open-source library to provide high-performance query processing operations on Intel CPUs.  Intel® QPL is aimed to support capabilities of the new Intel® In-Memory Analytics Accelerator (Intel® IAA) available on Next Generation Intel® Xeon® Scalable processors, codenamed Sapphire Rapids processors, such as very high throughput compression and decompression combined with primitive analytic functions, as well as to provide highly-optimized SW fallback on other Intel CPUs

   For more information about QPL, please refer to:

   GitHub Repo: https://github.com/intel/qpl

   Intel® QPL Documentation: https://intel.github.io/qpl/index.html

2. Intel® In-Memory Analytics Accelerator Architecture Specification: https://www.intel.com/content/www/us/en/content-details/721858/intel-in-memory-analytics-accelerator-architecture-specification.html?wapkw=In-memory%20accelerator

## Feedback 

We value your feedback. If you have comments (positive or negative) on this guide or are seeking something that is not part of this guide, please reach out and let us know what you think. 
 

