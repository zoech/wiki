## select

source: fs/select.c

```c
// fs/select.c
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
		fd_set __user *, exp, struct __kernel_old_timeval __user *, tvp)
{
	return kern_select(n, inp, outp, exp, tvp);
}
```

SYSCALL\_DEFINE5 是一个宏定义，比如上面这个用法就是声明 `sys_select` 系统调用函数，并且有5个参数；

上面主要入口在 `kern_select`:

```c
// fs/select.c
static int kern_select(int n, fd_set __user *inp, fd_set __user *outp,
		       fd_set __user *exp, struct __kernel_old_timeval __user *tvp)
{
	struct timespec64 end_time, *to = NULL;
	struct __kernel_old_timeval tv;
	int ret;

    // 下面这个if 应该主要是设置超时设置的
	if (tvp) {
		if (copy_from_user(&tv, tvp, sizeof(tv)))
			return -EFAULT;

		to = &end_time;
		if (poll_select_set_timeout(to,
				tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),
				(tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))
			return -EINVAL;
	}

    // 主要入口
	ret = core_sys_select(n, inp, outp, exp, to);
	return poll_select_finish(&end_time, tvp, PT_TIMEVAL, ret);
}

```

主要入口 `core_sys_select` :
```c

```
