## file system key execution

#### fs/file.c
```
void __fd_install(struct files_struct *files, unsigned int fd,
		struct file *file)
{
	rcu_read_lock_sched();
	smp_rmb();
	fdt = rcu_dereference_sched(files->fdt);
	rcu_assign_pointer(fdt->fd[fd], file);
	rcu_read_unlock_sched();
}
```

#### fs/open.c
```
stream_open
	filp->f_mode &= ~(FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE | FMODE_ATOMIC_POS);
	filp->f_mode |= FMODE_STREAM;

```
