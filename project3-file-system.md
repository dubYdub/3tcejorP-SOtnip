# PROJECT 3: FILE SYSTEMS 



## Group Members



ZhiJian Wang <11611827@mail.sustech.edu.cn>
Mengwei Guo <11610615@mail.sustech.edu.cn>



## Labor Division



| Division | Task 1 Cache | Task 2 Extensible file | Task 3 Subdirectories |
| -------- | ------------ | ---------------------- | --------------------- |
| *Code*   | Mengwei Guo  | Zhijian Wang           | Zhijian Wang          |
| *Report* | Mengwei Guo  | Zhijian Wang           | Both                  |
|          |              |                        |                       |

## TASK 1 - BUFFER CACHE



To finish this port, we create the **cache.c** and implement the cache based on what we have learnt in class. We have planed to write several tests but gave up finally because of the lack of time. 

### DATA STRUCTURES  



*We create two new files **cache.c** and **cache.h** to place the implementation of memory cache in pintos.*

- For the cache block in cache.c:

```C
/* struct cache block */
struct cache_block {
    int valid;                          /* 1: this block is valid */
    int dirty;                          /* 1: this block has been modified */ 
    block_sector_t sector_index;        /* sector index of the data in block */   
    uint8_t data[BLOCK_SECTOR_SIZE];    /* the size of each block is 512 bytes */
    struct lock single_cache_lock;      /* lock for every block in cache */
    size_t chances_value;               /* for clock algorithm */ 
};
```



- The other attributes in cache.c, which include the cache blocks array, their lock the common lock of whole cache.

```c
/* Array contains all the cache blocks */
static struct cache_block cache_blocks[CACHE_SIZE];

/* Judge if cache has been initialized */
static bool cache_isCreated = false;

/* A lock for operations on the entire cache */
static struct lock cache_lock;

/* the number of valid cache blocks */
static int valid_block_num;
```



### ALGORITHMS & IMPLEMENTATION



We have implemented the following functions to make the cache work well. We will introduce them one by one.

```c
void cache_init (void);
void cache_read(block_sector_t sector,void *buffer, int offset, int read_size);
void cache_write(block_sector_t sector,void *buffer, int offset, int write_size);
int  cache_find(block_sector_t sector);
int  cache_fetch(block_sector_t sector);
int  cache_evict(void);
void cache_write_back(int block_index);
void cache_write_all_back(void);
void cache_clear(int block_index);
void cache_clear_all(void);
```



#### I. Cache_init

To initialize the cache, we need to do the following operations for every cache block:

- Initialize the cache block **lock**
- Set the **valid value** and **dirty value** as false

We also need to mark the cache has been created.



#### II. Cache_read

Cache_read read the indicated data from cache based on offered sector id, offset and read size:

1. Check the status of cache.
2. Use **cache_find** to find the cache block that stored the aimed sector.
3. Use **memcpy** to copy the data from buffer to the cache block.
4. Update the **chance_value** of this cache block as 1 



#### III. Cache_write

Cache_write write the indicated data from cache based on offered sector id, offset and read size:

1. Check the status of cache.
2. Use **cache_find** to find the cache block that stored the aimed sector.
3. Use **memcpy** to copy the data from cache block to the buffer.
4. Update the **chance_value**of this cache block as 1 
5. Mark this block **dirty**



#### IV. Cache_find

**Cache_find** takes sector id as its attribute and find the sector in cache. If the sector is already in the cache, it will return the cache block it(We call it **cache miss**). If the cache is not in, we will use **cache_fetch** to fetch the sector from disk (The details will be introduced in cache_fetch part) and return the cache block id that store it.

1. Use a for loop to scan the whole cache block
2. If the cache block is valid and the sector in this cache is what we want, directly return its cache block id.
3. If the function has not returned after the loop, call **cache_fetch** and use it as a return value.



#### V. Cache_fetch

When a cache miss happened, the cache fetch will be called to fetch sector from the disk to one of the cache block. When the cache is not full, an empty cache block will be used to store the sector, otherwise a current cache block will be kicked out by calling **cache_evict.** After that, we will update the attributes of the new cache block

1. Call cache_evict to find a cache block to put the new sector.
2. Use **block_read** to put the sector to the cache block.
3. Update the **dirty** value and **valid** value of this block.
4. Set the **chance_value** of this block as 0.



#### VI. Cache_evict

We apply the **clock algorithm** in this part to pick a cache to kick out when the cache is fulled. It should be mentioned the cache should be write back to disk if the cache block is dirty. We implement this in function **cache_write_back**. The clock algorithm can be divided into the following steps:

1. Use a fixed circular linked list to represent the cache blocks. The size is the size of cache blocks.
2. In a while loop, we scan the circular linked list. If the current node is an invalid node, then return its cache id.
3. If the current node is valid and it's chance value is 1, then minus 1 to its chance_value, move to next one
4. If the current node is valid and it's chance_value is 0. return its cache_id.



#### VII. Cache_write_back

We need to write back the dirty cache block to disk when it is kicked out. It will take cache block id as argument

1. Make sure the cache block is valid and dirty
2. Use **block_write** write the cache block back to disk according to the attribute of cache block: **sector_index**
3. Set the dirty value of this block as 0.



#### VIII. Cache_write_all_back

Write the whole cache back to disk. All we need to is operating the **cache_write_back**  on all the cache blocks in a for loop.



#### IX. Cache_clear

Cache_clear will simply clear the indicated cache block and don't write back.

1. Make sure the cache is valid.
2. Set the valid value and dirty value as false.



#### X. Cache_clear_all

Clear the whole cache. All we need to is operating the **cache_clear**  on all the cache blocks in a for loop.



### SYNCHRONIZATION

Synchronization is so important in this task. Cache can be viewed as the secondary disk, so the write and read operations should be limited to one thread at the same time. To avoid race condition, we have added :


- **single_cache_lock** for every block in cache.

  It's naturally to add a lock for every cache because we need to only to protect the modified cache when it is read or written. It's a waste of resources to let the cache wait for one cache block. We also acquire the lock before we write back the cache or want to clear it.


- **cache_lock** for the whole cache.

  Although we have the single lock for.every clock, we may meet some condition which can cover the operations on several cache blocks. For example, the **cache_init** and **cache_evict**.  We need to make sure that, this type of complicated can only be performed once.  Unfortunately, **single_cache_lock** can't do that. That's why we introduce cache_lock here.

### RATIONALE

The reason we introduces cache here is the same as what we have learnt from class. The cost of directly accessing the memory is very high, so we need to introduce cache to put the frequently used data and decrease the time we spending on reading and writing on data. The pintos also need this type of strategy to improve the total performance of itself. In our implementation, **cache_write** and **cache_read** can be directly used to operate on disk' s memory. The user can also use **cache_clear_all** and **cache_write_all_back** to clear the cache or write the cache back to disk. It's also a type of protection to the data in the disk.

## TASK 2 - EXTENSIBLE FILES



To finish this port, we have to improve the inode and change the inode_create,inode_read_at and inode_write_at. 

### DATA STRUCTURES


### ALGORITHMS & IMPLEMENTATION

### SYNCHRONIZATION


## SURVEY QUESTIONS



### Question1: 

***When one process is actively reading or writing data in a buffer cache block, how are other processes prevented from evicting that block?***

- First, the whole cache should have a lock to prevent race condition when there are some operations focus on the whole cache, which may include the the process of clock algorithm.
- Second, every cache block should own a lock. While we read data from cache or write data to cache, these locks should be acquired. We don't use global lock here because the operation on single cache may cause a waste of when other processes can't access another cache block. A single lock for a single block make the cache system more efficient. Another reason is that, we need to make sure some complicated process, like cache_init and cache_evict, can be only running be one process at the same time. This type of protection can only be offered by the whole cache lock.

### Question2

***During the eviction of a block from the cache, how are other processes prevented from attempting to access the block?***

As we said in the question 1, a lock for the whole cache has been introduced in our implementation. We acquire this lock before we call the cache_evict function in **cache_fetch**, and release this lock after we finished this function.

### Question3

***If a block is currently being loaded into the cache, how are other processes prevented from also loading it into a different cache entry? How are other processes prevented from accessing the block before it is fully loaded?***

Acquire the single lock that for every cache block. We have talked about it in question1.



### Question4

How will your file system take a relative path like ../my_files/notes.txt and locate the corresponding directory? Also, how will you locate absolute paths like /cs162/solutions.md?

For relative path, our system will split it into directery and file, if system find .. after split, it will go to the parent directery base on currect directery. Then follow the directery order after split to find the goal file. For locate absolute paths, it mean we cannot find .. after split in the begin of the path, so we will change the path to the root and the follow the directory after split.


### Question5

Will a user process be allowed to delete a directory if it is the cwd of a running process? The test suite will accept both “yes” and “no”, but in either case, you must make sure that new files cannot be created in deleted directories.

No, it a directory if it is the cwd of a running process, the directory will be lock, so it cannot be delete. If the directory is been deleted, althouth we have to use time to delete it in the memory, but we can set the is_remove of the directory to be true, and then user cannot go to these directory or create file in these. 

### Question6

How will your syscall handlers take a file descriptor, like 3, and locate the corresponding file or directory struct?



### Question7

You are already familiar with handling memory exhaustion in C, by checking for a NULL return value from malloc. In this project, you will also need to handle disk space exhaustion. When your file system is unable to allocate new disk blocks, you must have a strategy to abort the current operation and rollback to a previous good state.
