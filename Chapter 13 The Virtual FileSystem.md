## **前言**
    最近在阅读Linux内核设计与实现，由于最近才开始写博客，所以写博客之前章节就留给以后吧。
    在博客中引用书中的内容，尽量用英文，也算是练习自己的英文吧，另一方面也让自己整理一下知识。
    
## **About The Virtual FileSystem**
The *Virtual FileSystem* (sometimes called the *Virtual FileSystem Switch* or more commonly simple *VFS*) is the subsystem of the kernel that implements the file and filesystem-related intefaces provided to user-space programs. All filesystems rely on the VFS to enable them not only to coexist, but also to interoperate.

 It is because modern operating systems, such as Linux, abstract access to the filesystems via a virtual interface that such interoperation and generic access is possible.

Filesystem abstraction layer, such a generic interface for any type of filesystem is feasible only because the kernel implements an abstraction layer around its low-level filesystem interface.The abstraction layer works by defining the basic conceptual interfaces and data structure that all filesystems support.

![这里写图片描述](http://img.blog.csdn.net/20160417205612927)

## **About Unix Filesystems**
Historically, Unix has provided 4 basic filesystem-related abstractions: *files*, *directory entries*,*inodes* and *mount points*.

A *filesystem* is a hierarchical storage of data adhering to a specific structure.In Unix, filesystems are mounted at a specific mount point in a global hierarchy known as *namespace*.This enables all mounted filesystems to appear as entries in a single tree. Contrast this single, unified tree with the behavior of DOS and Windows, which break the file namespace up into drive letters, such as **c:**.This breaks the namespace up among deviceand partition boundaries, "leaking" hardware details into filesystem abstractions.

dentry ---- directory entry
inode ---- index node, file metadata

## **VFS is object-oriented** 
VFS is programed strictly in C, without benefit of a language directly supporting object-oriented paradigms. 
The four primary object types of the VFS are:
 - superblock, which represents a specific mounted  filesystem
 - inode, which represents a specific file
 - dentry, which represents a directory entry
 - file, which represents an open file as associatedm with a process

## **The Superblock Object**
*superblock* or the *filesystem control block*, which is stored in a specific sector on disk.
Filesystem that are no disk-based generate the superblock on-the-fly and store it in memory.

header file: linux/fs.h

```c
	 struct super_block{
	        struct list_head              s_list;           /* list of all superblock */
	        dev_t                         s_dev;            /* identifier */
	        unsigned long                 s_blocksize;      /* block size in bytes */
	        unsigned char                 s_blocksize_bits; /* block size in bits */
	        unsigned char                 s_dirt;           /* dirty flag */
	        unsigned long long            s_maxbytes;       /* max file size */     
	        struct file_system_type       s_type;           /* filesystem type */
	        struct super_operations       s_op;             /* superblock methods */
	        struct dquot_operations       *dq_op;           /* quota methods */
	        struct quotactl_ops           *s_qcop;          /* quota control methods */
	        struct export_operations      *s_export_op;     /* export methods */
	        unsigned long                 s_flags;          /* mount flags */
	        unsigned long                 s_magic;          /* filesystem's magic number */
	        struct dentry                 *s_root;          /* directory mount point */
	        struct rw_semaphore           s_umount;         /* unmount semaphore */
	        struct semaphore              s_lock;           /* superblock semaphore */
	        int                           s_count;          /* superblock ref count */
	        int                           s_need_sync;      /* not-yet-synced flag */
	        atomic_t                      s_active;         /* active reference count */
	        void                          *s_security;      /* security module */
	        struct xattr_handler          **s_xattr;        /* extended attribute handlers */
	        struct list_head              s_inodes;         /* list of inodes */
	        struct list_head              s_dirty;          /* list of dirty inodes */
	        struct list_head              s_io;             /* list of writebacks */
	        struct list_head              s_more_io;        /* list of more writebacks */
	        struct hlist_head             s_anon;           /* anonymous dentries */
	        struct list_head              s_files;          /* list of assigned files */
	        struct list_head              s_dentry_lru;     /* list of unused entries */
	        int                           s_nr_dentry_unused;/* number of dentries on list */
	        struct block_device           *s_bdev;           /* associate block device */
	        struct mtd_info               *s_mtd;            /* memory disk infomation */
	        struct list_head              s_instances;       /* instances of this fs */
	        struct quota_info             s_dquot;           /* quota-specific options */
	        int                           s_frozen;          /* frozen status */
	        wait_queue_head_t             s_wait_unfrozen;   /* wait queue on freeze */
	        char                          s_id[32];          /* text name */
	        void                          *s_fs_info;        /* filesystem-specific info */
	        fmode_t                       s_mode;            /* mount permissons */
	        struct semaphore              s_vfs_rename_sem;  /* rename semaphore */
	        u32                           s_time_gran;       /* granularity of timestamps */
	        char                          *s_subtype;        /* subtype name */
	        char                          *s_options;        /* saved mount options */
	 };
```
superblock operartions:
```c
struct super_operations{
       struct inode *(*alloc_inode)(struct super_block *sb);
       void          (*destory_inode)(struct inode *);
       void          (*dirty_inode)(struct inode *);
       int           (*write_inode)(struct inode *, int);
       void          (*drop_inode)(struct inode *);
       void          (*delete_inode)(struct inode *);
       void          (*put_super)(struct super_block *);
       void          (*write_super)(struct super_block *);
       int           (*sync_fs)(struct super_block* sb, int wait);
       int           (*freeze_fs)(struct super_block *);
       int           (*unfreeze_fs)(struct super_block *);
       int           (*statfs)(struct dentry *, struct kstatfs *);
       int           (*remount_fs)(struct super_block *, int *, char *);
       void          (*clear_inode)(struct inode *);
       void          (*umount_begin)(struct super_block *);
       int           (*show_options)(struct_seq_file *, struct vfsmount *);
       int           (*show_stats)(struct seq_file *, struct vfsmount *);
       ssize_t       (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
       ssize_t       (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
       int           (*bdev_try_to_free_page)(struct super_block *, struct page *, gfp_t);      
};
```
some of these functions are optional; a specific filesystem can then set its value in the superblock operations structure to NULL. If the associated pointer is NULL, the VFS either calls a generic function or does nothing, depending on the operation.

## **The Inode Object**

to be continued.......

---
参考书籍:《Linux Kernel Development (3th)》, (美)Robert Love著.