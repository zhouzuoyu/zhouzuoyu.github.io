---
title: PM(1)_Overview
date: 2018-02-3 12:00:00
categories:
- Kernel
- Power
---

1. kernel/power/

  >   main.c：/sys/power/下注册了各类接口

  >   {% post_link Kernel/power/suspend流程 suspend.c %}：阐述linux suspend流程

  > {% post_link Kernel/power/autosleep autosleep.c %}：提供有机会就suspend的feature

  <!-- more -->

2. drivers/base/power/
  > {% post_link Kernel/power/wakeup_event wakeup.c %}：保证suspend的同步机制

  > {% post_link Kernel/power/runtime runtime.c %}：Device RPM

  > {% post_link Kernel/power/wakelock wakelock.c %}：提供阻止进入suspend的接口
