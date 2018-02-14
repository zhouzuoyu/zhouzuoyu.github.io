---
title: cpufreq(3)_cpufreq_governor框架
date: {{ date }}
categories:
- Kernel
- cpufreq
---
# 源码
框架API
> drivers\cpufreq\cpufreq_governor.c

# 功能
可以往/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor写合法的governor来change(store_scaling_governor)
最开始使用cpufreq_default_governor设置默认governor(cpufreq_online)

# 源码分析
## change cpufreq governor
<!-- more -->
往/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor中写入合法governor
```c
store_scaling_governor
	/*
	 * 1: 用户想要的governor，看系统是否已经支持(是否已经注册到cpufreq_governor_list)
	 * 2: 设置
	 */
	// 解析
	cpufreq_parse_governor(str_governor, &new_policy.policy, &new_policy.governor);
		// search governor
		find_governor(str_governor);
			for_each_governor(t)
				if (!strncasecmp(str_governor, t->name, CPUFREQ_NAME_LEN))
					return t;
		// return
		*governor = t;
	// 设置
	cpufreq_set_policy(policy, &new_policy);
```

## start cpufreq governor
如果该cpu还没有policy管理: 使用默认的governor管理
　　　　　　有　　　　　　: 使用last_governor管理

每个cpu都有一份cpufreq_cpu_data，用于保存该cpu的policy信息
当创建一个per-CPU变量时,系统中的每个处理器都会获得它自己对这个变量的拷贝(副本).
存取per-CPU变量时几乎不需要加锁,因为每个处理器使用的都是它自己的拷贝
> static DEFINE_PER_CPU(struct cpufreq_policy *, cpufreq_cpu_data);

cpufreq_online: 有两种情况下将调用该函数
1: cpufreq_register_driver
2: cpu online
```c
static struct subsys_interface cpufreq_interface = {
	.add_dev	= cpufreq_add_dev,
};
cpufreq_register_driver
	// 只会成功走一次
	if (cpufreq_driver)
		goto out;
	cpufreq_driver = driver_data;
	// call cpufreq_interface.add_dev == cpufreq_add_dev
	subsys_interface_register(&cpufreq_interface);
		sif->add_dev(dev, sif);
		//cpufreq_add_dev
			/*
			 * 检查该cpu是否已经有policy进行管理
			 * 没有则新建一个并且由cpufreq_default_governor()管理
			 * 有则使用last_governor管理
			 */
			cpufreq_online(cpu);
				/*
				 * 检查的部分
				 * 检查该cpu是否已经有policy进行管理
				 */
				policy = per_cpu(cpufreq_cpu_data, cpu);
				if (policy)
					// 有
				else {
					// 没有
					new_policy = true;
					policy = cpufreq_policy_alloc(cpu);
						policy->cpu = cpu;
				}
				/* 以下均讨论没有需要新建一个的情况 */
				// policy->cpus == policy->related_cpus == 1<<cpu
				cpumask_copy(policy->cpus, cpumask_of(cpu));
				cpumask_copy(policy->related_cpus, policy->cpus);
				cpufreq_driver->init(policy);
				
				/*
				 * 设置的部分
				 * 将policy赋值给对应cpu上的cpufreq_cpu_data，表明该cpu由该种policy来管理
				 */
				for_each_cpu(j, policy->related_cpus)
					per_cpu(cpufreq_cpu_data, j) = policy;
				/*
				 * check当前的policy是new/old. last_governor保存cpufreq_offline时的policy，so使用last_governor进行判断
				 * new: 没有last_governor, 使用cpufreq_default_governor
				 * old: 使用last_governor
				 */
				cpufreq_init_policy(policy);
					gov = find_governor(policy->last_governor);
					if (gov) {
					} else {
						gov = cpufreq_default_governor();
					}
					new_policy.governor = gov;
					/*
					 * 1: stop old governor
					 * 2: start new governor
					 */
					cpufreq_set_policy(policy, &new_policy);
						// stop old policy
						old_gov = policy->governor;
						cpufreq_stop_governor(policy);
						cpufreq_exit_governor(policy);
						// start new governor
						policy->governor = new_policy->governor;
						cpufreq_init_governor(policy);
							policy->governor->init(policy);
						cpufreq_start_governor(policy);
							policy->governor->start(policy);
							policy->governor->limits(policy);
```

## cpufreq_offline
offline时，将当前的governor记录到policy->last_governor，以备online时继续使用
```c
cpufreq_offline
	if (policy_is_inactive(policy))
		strncpy(policy->last_governor, policy->governor->name, CPUFREQ_NAME_LEN);
```
