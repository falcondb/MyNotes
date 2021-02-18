## Kernel Initialization

### `init/main.c`
```
__init start_kernel

```

```
run_init_process
```

```
do_initcalls
```


```
int __ref kernel_init
  pid = kernel_thread(kernel_init, NULL, CLONE_FS)
```
