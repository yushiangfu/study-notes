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
};
```

```
struct kset {
    struct list_head list;                      // to group the set members
    struct kobject kobj;                        // to manage the kset properties
    const struct kset_uevent_ops *uevent_ops;   // to inform the userspace of the set state
} __randomize_layout;

```

```
struct kobj_type {  // it defines how to export the info to sysfs
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
};
```

## <a name="reference"></a> Reference
