## Qemu migration code

### Key data structures

#### `migration/vmstate.h`
```
struct VMStateField {

  const VMStateInfo *info;
  enum VMStateFlags flags;
  const VMStateDescription *vmsd;

}

```

#### `migration/qemu-file.c`
```
struct QEMUFile {
    const QEMUFileOps *ops;
    const QEMUFileHooks *hooks;

    struct iovec iov[MAX_IOV_SIZE];
  }


  struct VMStateDescription {

      MigrationPriority priority;
      LoadStateHandler *load_state_old;

      int (*pre_load)(void *opaque);
      int (*post_load)(void *opaque, int version_id);
      int (*pre_save)(void *opaque);
      int (*post_save)(void *opaque);
      bool (*needed)(void *opaque);
      bool (*dev_unplug_pending)(void *opaque);

      const VMStateField *fields;
      const VMStateDescription **subsections;
  }

```

#### `migration/qemu-file.h`
```
typedef struct QEMUFileOps {
    QEMUFileGetBufferFunc *get_buffer;
    QEMUFileCloseFunc *close;
    QEMUFileSetBlocking *set_blocking;
    QEMUFileWritevBufferFunc *writev_buffer;
    QEMURetPathFunc *get_return_path;
    QEMUFileShutdownFunc *shut_down;
} QEMUFileOps;

typedef struct QEMUFileHooks {
    QEMURamHookFunc *before_ram_iterate;
    QEMURamHookFunc *after_ram_iterate;
    QEMURamHookFunc *hook_ram_load;
    QEMURamSaveFunc *save_page;
} QEMUFileHooks;
```

#### `io/channel.h`
```
// inspired by GIOChannel
struct QIOChannel {
  AioContext *ctx;
  Coroutine *read_coroutine;
  Coroutine *write_coroutine;

```

### migration/migration.c

```
migration_object_init

  blk_mig_init
    // see its section

  ram_mig_init

  dirty_bitmap_mig_init
```


### migration/block.c
```
blk_mig_init
  register_savevm_live("block", 0, 1, &savevm_block_handlers, &block_mig_state)

```


### migration/ram.c
```
ram_mig_init
  register_savevm_live("ram", 0, 4, &savevm_ram_handlers, &ram_state)
```


### migration/block-dirty-bitmap.c
```
dirty_bitmap_mig_init
  register_savevm_live("dirty-bitmap", 0, 1, &savevm_dirty_bitmap_handlers, &dbm_state);
```


### migration/savevm.c

```
static SaveState savevm_state = {
    .handlers = QTAILQ_HEAD_INITIALIZER(savevm_state.handlers),
    .handler_pri_head = { [MIG_PRI_DEFAULT ... MIG_PRI_MAX] = NULL },
    .global_section_id = 0,
};
```

```
// Legacy way, VMSTE is the latest
register_savevm_live
  // populate an SaveStateEntry

  savevm_state_handler_insert
    // figure out the right priority, and save the given SaveStateEntry nse
    savevm_state.handler_pri_head[priority] = nse
```

```
vmstate_register
  vmstate_register_with_alias_id
    savevm_state_handler_insert
      // insert the SaveStateEntry to savevm_state.handler_pri_head based on its priority
```

### Live Migration code

```
// migration/ram.c

// mark all RAM dirty
ram_save_setup


// sending dirty RAM pages
ram_save_iterate


// stop guest, transfer remaining dirty RAM, device state
migration_thread

```
