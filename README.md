# ccp_kernel_fixbug
I found that sometimes the ccp-kernel can not work normally.When I use "dmesg" command,the kernel log showes "[11447.630107] [ccp] ccp_connection not initialized".To fix this bug,change the code of function tcp_ccp_register() in the tcp_ccp.c as follows:

static int __init tcp_ccp_register(void) {
    int ok,sid;
struct ccp_connection *conn;
    getnstimeofday64(&tzero);

#ifdef COMPAT_MODE
    pr_info("[ccp] Compatibility mode: 4.13 <= kernel version <= 4.16\n");
#endif
#ifdef RATESAMPLE_MODE
    pr_info("[ccp] Rate-sample mode: 4.19 <= kernel version\n");
#endif

    kernel_datapath = kmalloc(sizeof(struct ccp_datapath), GFP_KERNEL);
    if(!kernel_datapath) {
        pr_info("[ccp] could not allocate ccp_datapath\n");
        return -4;
    }

    kernel_datapath->max_connections = MAX_ACTIVE_FLOWS;
    kernel_datapath->ccp_active_connections =
        (struct ccp_connection *) kmalloc(sizeof(struct ccp_connection) * MAX_ACTIVE_FLOWS, GFP_KERNEL);
    if(!kernel_datapath->ccp_active_connections) {
        pr_info("[ccp] could not allocate ccp_active_connections\n");
        return -5;
    }

	//folowing 4 line code added by zhujiahua to fix the bug "[ccp] ccp_connection not initialized"!
	for (sid = 0; sid < kernel_datapath->max_connections; sid++) {
	conn = & kernel_datapath->ccp_active_connections[sid];
	conn->index=0;
	}
	
	
    kernel_datapath->max_programs = MAX_DATAPATH_PROGRAMS;
    kernel_datapath->set_cwnd = &do_set_cwnd;
    kernel_datapath->set_rate_abs = &do_set_rate_abs;
    kernel_datapath->now = &ccp_now;
    kernel_datapath->since_usecs = &ccp_since;
    kernel_datapath->after_usecs = &ccp_after;
    kernel_datapath->log = &ccp_log;
#if __IPC__ == IPC_NETLINK
    ok = ccp_nl_sk(&ccp_read_msg);//创建netlink 
    if (ok < 0) {
        return -1;
    }

    kernel_datapath->send_msg = &nl_sendmsg;
    pr_info("[ccp] ipc = netlink\n");
#elif __IPC__ == IPC_CHARDEV
    ok = ccpkp_init(&ccp_read_msg);
    if (ok < 0) {
        return -2;
    }

    kernel_datapath->send_msg = &ccpkp_sendmsg;
    pr_info("[ccp] ipc = chardev\n");
#else
    pr_info("[ccp] ipc =  %s unknown\n", __IPC__);
    return -3;
#endif
	
    ok = ccp_init(kernel_datapath);//检查kernel_datapath的set_cwnd()等函数是否设置
    if (ok < 0) {
        return -6;
    }

    pr_info("[ccp] init\n");
    return tcp_register_congestion_control(&tcp_ccp_congestion_ops);
}
