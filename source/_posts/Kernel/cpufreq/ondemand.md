---
title: cpufreq(6)_ondemand
date: {{ date }}
categories:
- Kernel
- cpufreq
---
# 源码
drivers\cpufreq\cpufreq_ondemand.c

# 功能
根据以下两个公式来计算合适的频率
load = 100 * (time_elapsed - idle_time) / time_elapsed;
freq_next = min_f + load * (max_f - min_f) / 100;
<!--more-->
# 源码分析
填充dbs governor结构体，主要是实现gov_dbs_update

```c
static struct dbs_governor od_dbs_gov = {
	.gov = CPUFREQ_DBS_GOVERNOR_INITIALIZER("ondemand"),
	.gov_dbs_update = od_dbs_update,	// FIXME：实现策略
	.init = od_init,	// 回填dbs_data用: 设置参数，不同governor的个性化部分
};
```

设置参数
```c
od_init
	dbs_data->up_threshold = MICRO_FREQUENCY_UP_THRESHOLD;	// 95%, 设置一个截止点，>up_threshold，则满频工作
	dbs_data->min_sampling_rate = MICRO_FREQUENCY_MIN_SAMPLE_RATE;	// 10ms，设置采样率
	dbs_data->io_is_busy = should_io_be_busy();	// 0, 是否需要统计iowait时间
```

采样时间内统计cpu负载并且设置合适频率
```c
od_dbs_update
	od_update(policy);
		// 计算load
		load = dbs_update(policy);	// load对应cpu负载的百分比
			/*
			 * 统计查询时间段内idle_time的占比
			 * 100 * (time_elapsed - idle_time) / time_elapsed;
			 */
			cur_idle_time = get_cpu_idle_time(j, &update_time, io_busy);
			// time_elapsed
			time_elapsed = update_time - j_cdbs->prev_update_time;
			j_cdbs->prev_update_time = update_time;
			// idle_time
			idle_time = cur_idle_time - j_cdbs->prev_cpu_idle;
			j_cdbs->prev_cpu_idle = cur_idle_time;
			// cal load
			load = 100 * (time_elapsed - idle_time) / time_elapsed;
		// 执行不同的策略
		if (load > dbs_data->up_threshold) {
			// >up_threshold, 直接满频
			dbs_freq_increase(policy, policy->max);
				__cpufreq_driver_target(policy, freq, od_tuners->powersave_bias ?
						CPUFREQ_RELATION_L : CPUFREQ_RELATION_H);
		} else {
			/* 
			 * 根据cpu的load设置相应频率
			 */
			freq_next = min_f + load * (max_f - min_f) / 100;
			__cpufreq_driver_target(policy, freq_next, CPUFREQ_RELATION_C);
		}
```
