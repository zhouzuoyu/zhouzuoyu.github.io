---
title: cpufreq(1)_overview
date: 2018-02-7 12:00:00
categories:
- Linux
- cpufreq
---

# cpufreq_policy
每个cpu都有一个cpufreq_cpu_data，该变量记录了该cpu所使用的policy。
policy中会记录governor，so每个cpu都知道管理自己的是哪种governor.
cpu被第一次管理时，会使用默认的governor，后期可以通过节点设置的方式进行调整.
```c
struct cpufreq_policy {
	struct cpufreq_governor	*governor;				// current governor
	void *governor_data;							// policy->governor_data->dbs_data记录着对应dbs_governor的参数
	char last_governor[CPUFREQ_NAME_LEN];			// 记录cpufreq_offline时的governor，cpufreq_online时使用
	struct cpufreq_frequency_table *freq_table;		// 频率表
	enum cpufreq_table_sorting freq_table_sorted;	// 频率表是升序还是降序
};
```
<!-- more -->
# cpufreq_driver
负责策略的执行，具体到平台
cpufreq_driver驱动中要负责两大块
	1：电压和频率两张表的设计
	2：cpufreq_driver->target_index/target的填充：governor计算得到频率具体的执行体。传入频率，自行对应合适的电压。

# cpufreq_governor
cpufreq_governor负责策略的制定，并不负责策略的执行，抽象，不具体到平台
cpufreq_governor从代码框架上大致可以分为两类
1. 一种比较简单，只需要填充[governor]结构体即可
cpufreq_performance.c
直接满载
2. 另一类较为复杂(ondemand)，需要计算cpu负载等等，so抽象共有的 => cpufreq_governor.c
各个驱动中需要填充[dbs_governor]
* cpufreq_governor.c:
	将conservative，ondemand等governor共同的地方抽象出来。
	so每个dbs_governor可以负责实现自己特色的地方，如init，update...
* cpufreq_demand.c
	cpu负载发生变化时，并且到达设定采样频率时(默认10ms)
	cpu负载>95则满载，其他负载则((time_elpsed-idle_time)/time_elapsed * (max_f-min_f)) + min_f
