## [Linux kernel APIs](https://www.kernel.org/doc/htmldocs/kernel-api/)

- `get_user __get_user`:  copies a single simple variable from user space to kernel space. It supports simple types like char and int, but not larger data types like structures or arrays. `__get_user` with less checking

- `__put_user`: Write a simple value into user space

- `copy_to_user __copy_to_user`: Copy a block of data into user space. `__copy_to_user` with less checking

- `__copy_from_user`: Copy a block of data from user space, with less checking.
