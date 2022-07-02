## Index

1. [Introduction](#introduction)
2. [Reference](#reference)

## <a name="introduction"></a> Introduction

```
struct kobject {
    const char      *name;        // the name exported to userspace
    struct list_head    entry;    // kobjects on the list are in the same set
    struct kobject      *parent;  // to support hierarchical structure
    struct kset     *kset;        // also for set management
    struct kobj_type    *ktype;   // provide more info of the outer structure
    struct kernfs_node  *sd;      // sysfs directory entry
    struct kref     kref;         // for reference management
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work release;
#endif
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

## <a name="reference"></a> Reference
