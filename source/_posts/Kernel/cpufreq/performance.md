---
title: cpufreq(7)_performance
date: 2018-02-7 12:00:00
categories:
- Kernel
- cpufreq
---
# 源码
drivers\cpufreq\cpufreq_performance.c

# 功能
直接选用cpufreq_driver->target_index的最大档
<!--more-->
# 源码分析
## 注册governor
__cpufreq_governor_list__：所有可用的cpufreq governor
```c
cpufreq_gov_performance_init
	cpufreq_register_governor(&cpufreq_gov_performance);
		list_add(&governor->governor_list, &cpufreq_governor_list);	// 挂载到cpufreq_governor_list
```

## Policy
```c
cpufreq_gov_performance_limits
	__cpufreq_driver_target(policy, policy->max, CPUFREQ_RELATION_H);	// policy->max在cpufreq_driver->init中进行设置
		// 找到cpufreq_frequency_table中最接近target的index
		index = cpufreq_frequency_table_target(policy, target_freq, relation);
			return cpufreq_table_find_index_h(policy, target_freq);
				// 根据cpufreq_frequency_table选择，例如s5pv210-cpufreq.c中该表为降序排列
				return cpufreq_table_find_index_dh(policy, target_freq);
		// call cpufreq_drvier->target_index
		__target_index(policy, index);
			cpufreq_driver->target_index(policy, index);

static struct cpufreq_governor cpufreq_gov_performance = {
	.limits		= cpufreq_gov_performance_limits,
};
```
