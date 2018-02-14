---
title: cpufreq(5)_interactive
date: {{ date }}
categories:
- Kernel
- cpufreq
---
# 功能
Android中广泛使用的cpufreq governor
启动定时器计算cpu负载程度决定选用cpufreq档位

# 源码分析
## register
<!--more-->
```c
struct cpufreq_governor cpufreq_gov_interactive = {
	.name = "interactive",
	.governor = cpufreq_governor_interactive,
};
// cpufreq.c中会调用governor->init
cpufreq_governor_interactive
	// init: 设置定时器的func函数
	case CPUFREQ_GOV_POLICY_INIT:
		tunables = alloc_tunable(policy);
			// default settings
			tunables->timer_rate = DEFAULT_TIMER_RATE;	// 20 * 1000
		common_tunables = tunables;
		ppol->policy_timer.function = cpufreq_interactive_timer;
		break;
	// start: 执行定时器的func函数
	case CPUFREQ_GOV_START:
		cpufreq_interactive_timer_start(tunables, policy->cpu);
			// 设置定时器时间
			expires = round_to_nw_start(ppol->last_evaluated_jiffy, tunables);
				ret = jiffies + usecs_to_jiffies(tunables->timer_rate);	// 
			add_timer(&ppol->policy_timer);
		break;
```
```c
cpufreq_interactive_init
	// 创建了一个内核线程
	speedchange_task = kthread_create(cpufreq_interactive_speedchange_task, NULL, "cfinteractive");
	cpufreq_register_governor(&cpufreq_gov_interactive);
```

## policy
从全局来看有一个定时器和内核线程
timer: 计算target
kthread: 设置频率

### part1: timer
计算适当频率，在freq_table中匹配，最后保存在policyinfo->target_freq中
```c
cpufreq_interactive_timer
	/*
	 * 计算合适频率和负载
	 * 合适频率: active_time/delta_time * 100 * cur_freq
	 * 负载: active_time/delta_time * cur_freq/target_freq * 100
	 */
	now = update_load(cpu);	// 当前时间
		now_idle = get_cpu_idle_time(cpu, &now, tunables->io_is_busy);
		pcpu->cputime_speedadj += active_time * ppol->policy->cur;
		pcpu->time_in_idle_timestamp = now;
		return now;
	delta_time = now - pcpu->cputime_speedadj_timestamp;
	cputime_speedadj = pcpu->cputime_speedadj;
	do_div(cputime_speedadj, delta_time);
	// min_freq->max_freq
	t_prevlaf = (unsigned int)cputime_speedadj * 100;
	prev_laf = t_prevlaf;
	// prev_l 0->100: 越小cpu负载越小，越大cpu负载越大
	prev_l = t_prevlaf / ppol->target_freq;
	
	// 选择合适频率
	prev_chfreq = choose_freq(ppol, prev_laf);
	pred_chfreq = choose_freq(ppol, pred_laf);
	chosen_freq = max(prev_chfreq, pred_chfreq);
	new_freq = chosen_freq;
	
	// 遍历整张freq_table，找到最靠近new_freq的频率值，输出给index
	cpufreq_frequency_table_target(&ppol->p_nolim, ppol->freq_table, new_freq, CPUFREQ_RELATION_L, &index);
	new_freq = ppol->freq_table[index].frequency;
	ppol->target_freq = new_freq;	// set target_freq
	cpumask_set_cpu(max_cpu, &speedchange_cpumask);	// set
	
	// 更新时间
	cpufreq_interactive_timer_resched(data, false);
		pcpu->cputime_speedadj = 0;
		pcpu->cputime_speedadj_timestamp = pcpu->time_in_idle_timestamp;
```

### part2: kthread
timer中更新了target_freq后，才会设置频率
```c
cpufreq_interactive_speedchange_task
	while (1) {
		set_current_state(TASK_INTERRUPTIBLE);
		/*
		 * 有cpu出现了speedchange，才设置频率
		 */
		if (cpumask_empty(&speedchange_cpumask))
			schedule();
		
		set_current_state(TASK_RUNNING);
		cpumask_clear(&speedchange_cpumask);	// clear
		// set
		__cpufreq_driver_target(ppol->policy,
							ppol->target_freq,
							CPUFREQ_RELATION_H);
	}
```
