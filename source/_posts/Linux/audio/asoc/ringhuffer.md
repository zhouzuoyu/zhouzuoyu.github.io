---
title: Ringbuffer
date: 2020-07-16 12:00:00
categories:
- Linux
- asoc
---

# period_size/buffer_size
period_size和buffer_size都是上层设置下来的参数
period_size：指定了底层每次dma传输的数据量，frame为单位，例如1024 frames
buffer_size：指定了上层每次传递给底层的数据量

# appl_ptr/hw_ptr
上层和底层共用了一块内存，并且是producer/consumer的关系，使用了ringbuffer的方式进行内存管理
需要两个指针：appl_ptr和hw_ptr
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;buffer_size
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;\-\-\-\-\-\-\-\-\-\- appl_ptr
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;...|&nbsp;&nbsp;|&nbsp;&nbsp;|&nbsp;&nbsp;|&nbsp;&nbsp;|...
hw_ptr&nbsp;\-\-\-\-\-\-\-\-\-\-


<!--more-->
appl_ptr指示了上层数据的当前位置，hw_ptr指示了底层数据的当前位置
hw_ptr靠dma中断更新，每次+=period_size

appl_ptr>=hw_ptr，一般情况控制在appl_ptr-hw_ptr=buffer_size
限制hw_ptr和appl_ptr在一个buffer_size的距离里，防止appl_ptr写的太快，毕竟是ringbuffer，没有同步限制，早晚会被覆盖掉
检测appl_ptr和hw_ptr的距离，>=buffer_size，appl_ptr就等等，这段时间里dma继续传输，所以hw_ptr会更新上来

## hw_ptr/appl_ptr更新周期
hw_ptr的节奏是确定的，即一次dma搬移的时间=1024/48000=21.3ms
如果appl_ptr写入较慢，>21.3×period_count，则会underrun

appl_ptr写入最快的时间也是确定的，即≈21.3×period_count，底层有等待机制(wait_for_avail)，确保基本同步
上层每次传递buffer_size(例如4*period_size)的数据下来，其实在底层是分了4次执行，主要原因是dma每次只搬period_size的数据
```c
while (size > 0) {	// 循环了4次
	frames = size > avail ? avail : size;	// avail=period_size
	writer(substream, appl_ofs, data, offset, frames, transfer);	// memcpy到dma内存空间
	appl_ptr += frames;	// 更新appl_ptr
	size -= frames;
}
```

# log分析
上层会传递buffer_size(period_size*period_count)的数据下来，tinyplay中默认period_count是4
底层dma每次搬移都是period_size的数据，所以要分4次搬移
因为上层每次给底层都是4个period_size，底层每次搬移1个period_size，所以中间涉及到等待，即上层要等待底层把上一次的buffer_size数据搬移掉上层才能把数据cp进来
```log
[   69.264421] Enter __snd_pcm_lib_xfer					// 一次buffer_size的传输
[   69.266300] avail=0, hw_ptr=45056, appl_ptr=49152	// appl_ptr-hw_ptr=buffer_size: 两者距离有点远了，appl_ptr需要等等hw_ptr
/* 1st */
[   69.267893] Enter while, size=4096					// 等待一次period_size传输
// wait begin(wait_for_avail)
[   69.269005] enter !avail branch
[   69.270244] Enter wait_for_avail
// snd_pcm_playback_avail走了3次，但是if (avail >= runtime->twake)只判断了2次，这个想不明白
[   69.271310] avail=0, hw_ptr=45056, appl_ptr=49152
[   69.274252] avail = 0
[   69.275841] avail=1024, hw_ptr=46080, appl_ptr=49152	// dma transfer finished, hw_ptr updated.
[   69.279535] avail=1024, hw_ptr=46080, appl_ptr=49152
[   69.281461] avail = 1024	// 每次avail都是1024，因为period_size就是1024，底层dma每次搬移都是1024
[   69.282169] Leave wait_for_avail
[   69.283638] leave !avail branch
// wait finish than memcpy, update appl_ptr. appl_ptr+=period_sie
[   69.285008] writer

/* 2nd */
[   69.285831] Enter while, size=3072
[   69.286701] enter !avail branch
[   69.287940] Enter wait_for_avail
[   69.289115] avail=0, hw_ptr=46080, appl_ptr=50176
[   69.290118] avail = 0
[   69.295779] avail=1024, hw_ptr=47104, appl_ptr=50176
[   69.297389] avail=1024, hw_ptr=47104, appl_ptr=50176
[   69.299502] avail = 1024
[   69.300532] Leave wait_for_avail
[   69.301696] leave !avail branch
[   69.303227] writer

// 3rd
[   69.304349] Enter while, size=2048
[   69.305627] enter !avail branch
[   69.306450] Enter wait_for_avail
[   69.307318] avail=0, hw_ptr=47104, appl_ptr=51200
[   69.308808] avail = 0
[   69.315888] avail=1024, hw_ptr=48128, appl_ptr=51200
[   69.317538] avail=1024, hw_ptr=48128, appl_ptr=51200
[   69.318767] avail = 1024
[   69.319643] Leave wait_for_avail
[   69.320783] leave !avail branch
[   69.321772] writer
[   69.322695] Enter while, size=1024

// 4th
[   69.323944] enter !avail branch
[   69.325140] Enter wait_for_avail
[   69.326126] avail=0, hw_ptr=48128, appl_ptr=52224
[   69.327874] avail = 0
[   69.335841] avail=1024, hw_ptr=49152, appl_ptr=52224
[   69.337470] avail=1024, hw_ptr=49152, appl_ptr=52224
[   69.338854] avail = 1024
[   69.339534] Leave wait_for_avail
[   69.340296] leave !avail branch
[   69.341211] writer
[   69.341750] avail=0, hw_ptr=49152, appl_ptr=53248
```
