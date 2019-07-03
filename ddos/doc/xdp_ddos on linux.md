# xdp_ddos on linux
## ddos简介

## 代码说明
***源代码来自***[***此处***](https://github.com/netoptimizer/prototype-kernel/tree/master/kernel/samples/bpf)

### user
***xdp_ddos01_blacklist_user.c文件为加载器***

### cmdline
***xdp_ddos01_blacklist_cmdline.c文件用于为黑名单增/删ip***

### kern

***xdp_ddos01_blacklist_kern.c文件需要加载到内核（生成bpf），功能为拒绝黑名单中的ip(?)访问网络***

接下来是简单的说明：

```
struct bpf_map_def SEC("maps") blacklist = {
	.type        = BPF_MAP_TYPE_PERCPU_HASH,
	.key_size    = sizeof(u32),
	.value_size  = sizeof(u64), /* Drop counter */
	.max_entries = 100000,
	.map_flags   = BPF_F_NO_PREALLOC,
};

struct bpf_map_def SEC("maps") port_blacklist = {
	.type        = BPF_MAP_TYPE_PERCPU_ARRAY,
	.key_size    = sizeof(u32),
	.value_size  = sizeof(u32),
	.max_entries = 65536,
};

```
***以上两个map为黑名单***

其余的map有：

* verdict_cnt:计数每个xdp操作（XDP_DROP/XDP_PASS...）的个数
* port_blacklist_drop_count_tcp:记录tcp drop的个数
* port_blacklist_drop_count__udp:记录udp drop的个数

函数parse_ipv4解析ip，如果在blacklist中查找到就执行XDP_DROP，并计数:
```
	value = bpf_map_lookup_elem(&blacklist, &ip_src);
	if (value) {
		*value += 1; /* Keep a counter for drop matches */
		return XDP_DROP;
	}

```

函数parse_port解析端口，如果在port_blacklist中查找到就执行XDP_DROP，判断tcp/udp并计数:
```
	dport_idx = dport;
	value = bpf_map_lookup_elem(&port_blacklist, &dport_idx);

	if (value) {
		if (*value & (1 << fproto)) {
			struct bpf_map_def *drop_counter = drop_count_by_fproto(fproto);
			if (drop_counter) {
				drops = bpf_map_lookup_elem(drop_counter , &dport_idx);
				if (drops)
					*drops += 1; /* Keep a counter for drop matches */
			}
			return XDP_DROP;
		}
	}
	return XDP_PASS;

```

## 运行展示

