---
title: cpufreq(2)_cpufreq_driver
date: 2018-02-7 12:00:00
categories:
- Linux
- cpufreq
---
# 源码
drivers\cpufreq\s5pv210-cpufreq.c
drivers\cpufreq\cpufreq.c

# 功能
DVFS的具体执行者
<!-- more -->
# 源码分析
## register cpufreq_driver
```c
static struct cpufreq_driver s5pv210_driver = {
	.target_index	= s5pv210_target,	// FIXME，该函数比较关键
	.init		= s5pv210_cpu_init,		// 初始化，比如设置policy->min/max...
};
// 记录最大最小频率
static int s5pv210_cpu_init(struct cpufreq_policy *policy)
	cpufreq_generic_init(policy, s5pv210_freq_table, 40000);
		cpufreq_table_validate_and_show(policy, table);
			// 遍历table，找到最大频率和最小频率
			cpufreq_frequency_table_cpuinfo(policy, table);
				cpufreq_for_each_valid_entry(pos, table) {
					freq = pos->frequency;
					if (freq < min_freq)
						min_freq = freq;
					if (freq > max_freq)
						max_freq = freq;
				}
				policy->min = policy->cpuinfo.min_freq = min_freq;
				policy->max = policy->cpuinfo.max_freq = max_freq;
			// 传入平台table
			policy->freq_table = table;
			/*
			 * 根据cpufreq_driver中cpufreq_frequency_table的排列，确定为升序/降序表
			 * policy->freq_table_sorted = CPUFREQ_TABLE_SORTED_ASCENDING/CPUFREQ_TABLE_SORTED_DESCENDING
			 */
			set_freq_table_sorted(policy);	// freq_table.c
```
```c
s5pv210_cpufreq_probe
	return cpufreq_register_driver(&s5pv210_driver);	// FIXME: 注册平台相关函数: DVFS【执行】函数
```

## dvfs table
平台相关，有如下两张表，电压/频率对应关系
L0: 性能优先
L4: 功耗优先
```c
static struct s5pv210_dvs_conf dvs_conf[] = {
	[L0] = {
		.arm_volt	= 1250000,	// 1.25V
		.int_volt	= 1100000,	// 1.1v
	},
	[L1] = {
		.arm_volt	= 1200000,
		.int_volt	= 1100000,
	},
	[L2] = {
		.arm_volt	= 1050000,
		.int_volt	= 1100000,
	},
	[L3] = {
		.arm_volt	= 950000,
		.int_volt	= 1100000,
	},
	[L4] = {
		.arm_volt	= 950000,	// 0.95v
		.int_volt	= 1000000,	// 1v
	},
};
// 该表必须为升序/降序排列，不能乱排
static struct cpufreq_frequency_table s5pv210_freq_table[] = {
	{0, L0, 1000*1000},	// 1GHz
	{0, L1, 800*1000},
	{0, L2, 400*1000},
	{0, L3, 200*1000},
	{0, L4, 100*1000},	// 100MHz
	{0, 0, CPUFREQ_TABLE_END},
};
```

## dvfs执行
根据index(cpufreq governor计算得到)调整电压和freq
```c
static int s5pv210_target(struct cpufreq_policy *policy, unsigned int index)
	// 调整arm/int电压
	arm_volt = dvs_conf[index].arm_volt;
	int_volt = dvs_conf[index].int_volt;	
	ret = regulator_set_voltage(arm_regulator, arm_volt, arm_volt_max);
	ret = regulator_set_voltage(int_regulator, int_volt, int_volt_max);
	// 调整DRAM刷新率
	s5pv210_set_refresh(DMC1, 83000);
	s5pv210_set_refresh(DMC0, 83000);
	// 修改分频系数来调整各个频率
 	reg |= ((clkdiv_val[index][0] << S5P_CLKDIV0_APLL_SHIFT) |
		(clkdiv_val[index][1] << S5P_CLKDIV0_A2M_SHIFT) |
		(clkdiv_val[index][2] << S5P_CLKDIV0_HCLK200_SHIFT) |
		(clkdiv_val[index][3] << S5P_CLKDIV0_PCLK100_SHIFT) |
		(clkdiv_val[index][4] << S5P_CLKDIV0_HCLK166_SHIFT) |
		(clkdiv_val[index][5] << S5P_CLKDIV0_PCLK83_SHIFT) |
		(clkdiv_val[index][6] << S5P_CLKDIV0_HCLK133_SHIFT) |
		(clkdiv_val[index][7] << S5P_CLKDIV0_PCLK66_SHIFT));
	writel_relaxed(reg, S5P_CLK_DIV0);
```
