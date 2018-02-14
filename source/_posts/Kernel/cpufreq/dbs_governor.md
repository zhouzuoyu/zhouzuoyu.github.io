---
title: cpufreq(4)_dbs_governor
date: {{ date }}
categories:
- Kernel
- cpufreq
---
# 功能
将ondemand conservative...共有的部分抽象出来
让cpufreq governor专注于各自的特性

# 源码分析
## 定义
dbs_governor将governor再封装了一层: dbs_governor.governor
将会调用dbs_governor->init：设置参数
<!--more-->
```c
#define CPUFREQ_DBS_GOVERNOR_INITIALIZER(_name_)			\
{								\
	.name = _name_,						\
	.init = cpufreq_dbs_governor_init,			\
}
```
```c
cpufreq_dbs_governor_init
	// 该governor指的是dbs_governor
	struct dbs_governor *gov = dbs_governor_of(policy);
	// check 如果dbs_data信息已经存在，则over
	dbs_data = gov->gdbs_data;
	if (dbs_data)
		goto out;
	// 下面讨论新建一个的情况
	dbs_data = kzalloc(sizeof(*dbs_data), GFP_KERNEL);
	/* 利用container_of，将调用dbs_governor->init(od_init)
	 * 具体的dbs_governor填充不同的参数
	 */
	gov->init(dbs_data);
	// set sampling_rate
	dbs_data->sampling_rate = max(dbs_data->min_sampling_rate,
				      LATENCY_MULTIPLIER * latency);
	// set
	gov->gdbs_data = dbs_data;
	policy_dbs->dbs_data = dbs_data;
	policy->governor_data = policy_dbs;
```

## 调节
cpu负载率有变化时将call dbs_update_util_handler
最终回调dbs_governor->gov_dbs_update
```c
cpufreq_dbs_governor_start
	sampling_rate = dbs_data->sampling_rate;
	gov_set_update_util(policy_dbs, sampling_rate);
		/*
		 * cpu负载率有变化时 将call dbs_update_util_handler
		 */
		cpufreq_add_update_util_hook(cpu, &cdbs->update_util, dbs_update_util_handler);
			// 没有到下一个采样时间，return
			if ((s64)delta_ns < policy_dbs->sample_delay_ns)
				return;
			irq_work_queue(&policy_dbs->irq_work);
				schedule_work_on(smp_processor_id(), &policy_dbs->work);
					// 执行gov->gov_dbs_update(policy), 如od_dbs_update
					gov_update_sample_delay(policy_dbs, gov->gov_dbs_update(policy));
```
