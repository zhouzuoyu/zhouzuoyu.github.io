---
title: uevent
date: 2019-07-01 16:43:00
categories:
- Linux
---

# uevent信息
```
KERNEL[258.533019] add		/devices/soc0/soc/2100000.aips-bus/2184000.usb/ci_hdrc.0/usb1/1-1/1-1:1.0/host2/target2:0:0/2:0:0:0/block/sda (block)
ACTION=add
DEVNAME=/dev/sda
DEVPATH=/devices/soc0/soc/2100000.aips-bus/2184000.usb/ci_hdrc.0/usb1/1-1/1-1:1.0/host2/target2:0:0/2:0:0:0/block/sda
DEVTYPE=disk
MAJOR=8
MINOR=0
SEQNUM=883
SUBSYSTEM=block

KERNEL[258.533289] add		/devices/soc0/soc/2100000.aips-bus/2184000.usb/ci_hdrc.0/usb1/1-1/1-1:1.0/host2/target2:0:0/2:0:0:0/block/sda/sda1 (block)
ACTION=add
DEVNAME=/dev/sda1
DEVPATH=/devices/soc0/soc/2100000.aips-bus/2184000.usb/ci_hdrc.0/usb1/1-1/1-1:1.0/host2/target2:0:0/2:0:0:0/block/sda/sda1
DEVTYPE=partition
MAJOR=8
MINOR=1
SEQNUM=884
SUBSYSTEM=block
```
<!--more-->
# 注册uevent
> lib\kobject_uevent.c

```c
static struct pernet_operations uevent_net_ops = {
	.init	= uevent_net_init,
	.exit	= uevent_net_exit,
};

static int __init kobject_uevent_init(void)
{
	return register_pernet_subsys(&uevent_net_ops);
}

postcore_initcall(kobject_uevent_init);
```

```c
static int uevent_net_init(struct net *net)
{
	ue_sk->sk = netlink_kernel_create(net, NETLINK_KOBJECT_UEVENT, &cfg);	// NETLINK_KOBJECT_UEVENT
		return __netlink_kernel_create(net, unit, THIS_MODULE, cfg);
			__netlink_create(&init_net, sock, cb_mutex, unit);
				sk->sk_protocol = protocol;	// 设置protocol: NETLINK_KOBJECT_UEVENT
	list_add_tail(&ue_sk->list, &uevent_sock_list);	// 放到uevent_sock_list链表中
}
```

# 上报uevent
uevent使用netlink(一种socket)发送事件(将相应事件放在接收队列末尾)
```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action)
{
	return kobject_uevent_env(kobj, action, NULL);
		const char *action_string = kobject_actions[action];
		// 设置一些事件字段
		retval = add_uevent_var(env, "ACTION=%s", action_string);
		retval = add_uevent_var(env, "DEVPATH=%s", devpath);
		retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);

		list_for_each_entry(ue_sk, &uevent_sock_list, list) {	// 从链表中取出的就是uevent
			struct sock *uevent_sock = ue_sk->sk;
			len = strlen(action_string) + strlen(devpath) + 2;
			skb = alloc_skb(len + env->buflen, GFP_KERNEL);	// 申请一个skb
			if (skb) {
				/* add header */
				scratch = skb_put(skb, len);
				sprintf(scratch, "%s@%s", action_string, devpath);

				// 发送事件
				retval = netlink_broadcast_filtered(uevent_sock, skb,
									0, 1, GFP_KERNEL,
									kobj_bcast_filter,
									kobj);
					sk_for_each_bound(sk, &nl_table[ssk->sk_protocol].mc_list)	// 根据protocol，得到sock
						do_one_broadcast(sk, &info);
							netlink_broadcast_deliver(sk, p->skb2);
								__netlink_sendskb(sk, skb);
									skb_queue_tail(&sk->sk_receive_queue, skb);	// 将skb放入该netlink套接字(NETLINK_KOBJECT_UEVENT)接收队列末尾
									sk->sk_data_ready(sk);						// 执行sk_data_ready回调通知该套接字有数据可读
		}
}
```

# 使用uevent
注册disk会上报uevent事件
```c
static int sd_probe(struct device *dev)
{
	async_schedule_domain(sd_probe_async, sdkp, &scsi_sd_probe_domain);
		add_disk(gd);
			register_disk(disk);
				kobject_uevent(&ddev->kobj, KOBJ_ADD);
}
```
