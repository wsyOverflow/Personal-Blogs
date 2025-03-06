---
typora-copy-images-to: ..\..\..\images\Rendering Blogs\GPGPU\${filename}.assets
typora-root-url: ..\..\..\
title: SlabHash
keywords: GPGPU, GPU Hash Table
categories:
- [Rendering Blogs, GPGPU]
mathjax: true
---

# 1 源码分析

使用到的warp操作：

- `unsigned __ballot_sync(unsigned mask, predicate)`
  mask中的bit为1 表示对相应id的线程执行warp操作，该线程的predicate非0将返回值对应位置为1
-  `int __ffs (int x)`
  返回最低非0有效位的位置。x为0，则返回0
- `T __shfl_sync(unsigned mask, T var, int srcLane, int width=warpSize)`
  从指定lane拷贝var变量。width控制lane索引，srcLane % width

## 1.1 searchKey

函数声明如下

```c++
GpuSlabHashContext<KeyT, ValueT, SlabHashTypeT::ConcurrentMap>::searchKey(
    bool& to_be_searched,
    const uint32_t& laneId,
    const KeyT& myKey,
    ValueT& myValue,
    const uint32_t bucket_id)
```

给一批keys，以32个为一组，对于一个warp内的lane，一个lane( `laneId` )绑定一个 `myKey` ，查找完成返回 `myValue` 。实际查找过程是先组成一个work_queue，对这个work_queue的每一个key，warp进行逐个key查找，直至work_queue完成。函数实现如下：

- line 5 组装work_queue：worke_queue为32bit整数，每一位表示对于lane是否完成查找。
  __ballot_sync 设置work_queue中bit，1表示对应lane需要查找key，0表示已完成查找或不需要

- line 6~7 设置查找位置。
  如果work_queue发生了变化，表示上一个lane的查找已经完成，那么处理下一个key，则需要在对应bucket从头开始查找。 `SlabHashT::A_INDEX_POINTER` 
  否则，表示需要继续查找下一个slab。

- line 8~17 获取当前迭代的查找目标信息以及查找位置的数据

  - 获取当前warp迭代查找目标信息：lane id，bucket，key
    \_\_ffs得到work_queue中的当前warp迭代的查找目标 `src_lane` (由小到大)，减1是因为 \_\_ffs 的返回值0是保留给参数0的，而bit位从1开始，lane id从0开始
    \_\_shfl_sync从查找目标lane中拷贝其 bucket_id
    __shfl_sync从查找目标lane中拷贝其要查找的key
  - 获取warp每个线程对应的查找位置的数据。
    两种查找位置，一个是bucket，一个是slab，返回的是当前线程的laneId位置处的数据(warp中每个线程负责一个位置)

- line 18~20 比较查找目标与查找位置的key，相同表示匹配，不同表示不匹配。warp是否已有线程查找到目标key
  __ballot_sync 得到warp所有线程的比较结果。注意后面 & 操作是用于处理slab中key-value的存放方式。

  - `REGULAR_NODE_KEY_MASK = 0x15555555` 意味着key-value是连续存放方式，因此一个warp的线程只有REGULAR_NODE_KEY_MASK 中bit为1处才是取得是key，其他取得是value。

  \__ffs 则是获取匹配上的lane id，后面-1为了处理lane id从0开始计数。

- line 21~30 处理本次warp迭代没有找到目标key。
  __shfl_sync拷贝warp最后一个lane(31)读取到的数据，slab的存储是next指针

  - 如果next指针空，表示bucket查找结束，意味着hash table不存在目标key
    查找目标lane将查找结果返回，并结束其查找任务 `to_be_searched = false`
  - next指针不为空，表示需要继续查找bucket的下一个slab

- line 31~38 处理本次warp迭代找到目标key。
  \__shfl_sync拷贝found_lane+1读取到的数据，即目标key对应的value
  查找目标lane将查找结果返回，并结束其查找任务 `to_be_searched = false`

可以看出key-value连续存储，而查找过程中，lane是连续读的，因此32个lane只能处理(16-1)个key-value对，最后一个是next指针。因此每个slab的大小是15个key-value对，末尾加上一个next指针。

```c++
uint32_t work_queue = 0;
uint32_t last_work_queue = work_queue;
uint32_t next = SlabHashT::A_INDEX_POINTER;
 
while ((work_queue = __ballot_sync(0xFFFFFFFF, to_be_searched))) {
    next = (last_work_queue != work_queue) ? SlabHashT::A_INDEX_POINTER
                                           : next;  // a successfull insertion in the warp
    uint32_t src_lane = __ffs(work_queue) - 1;
    uint32_t src_bucket = __shfl_sync(0xFFFFFFFF, bucket_id, src_lane, 32);
    uint32_t wanted_key = __shfl_sync(0xFFFFFFFF,
                                      *reinterpret_cast<const uint32_t*>(
                                          reinterpret_cast<const unsigned char*>(&myKey)),
                                      src_lane,
                                      32);
    const uint32_t src_unit_data = (next == SlabHashT::A_INDEX_POINTER)
                                       ? *(getPointerFromBucket(src_bucket, laneId))
                                       : *(getPointerFromSlab(next, laneId));
    int found_lane = __ffs(__ballot_sync(0xFFFFFFFF, src_unit_data == wanted_key) &
                           SlabHashT::REGULAR_NODE_KEY_MASK) -
                     1;
    if (found_lane < 0) {  // not found
        uint32_t next_ptr = __shfl_sync(0xFFFFFFFF, src_unit_data, 31, 32);
        if (next_ptr == SlabHashT::EMPTY_INDEX_POINTER) {  // not found
            if (laneId == src_lane) {
                myValue = static_cast<ValueT>(SEARCH_NOT_FOUND);
                to_be_searched = false;
            }
        } else {
            next = next_ptr;
        }
    } else {  // found the key:
        uint32_t found_value = __shfl_sync(0xFFFFFFFF, src_unit_data, found_lane + 1, 32);
        if (laneId == src_lane) {
            myValue = *reinterpret_cast<const ValueT*>(
            reinterpret_cast<const unsigned char*>(&found_value));
            to_be_searched = false;
        }
    }
    last_work_queue = work_queue;
}
```

## 1.2 insertPair

查找到empty slot后，将新的key-value放入该位置。该插入方式不会查找被插入key是否已存在。

函数声明

```c++
GpuSlabHashContext<KeyT, ValueT, SlabHashTypeT::ConcurrentMap>::insertPair(
    bool& to_be_inserted,
    const uint32_t& laneId,
    const KeyT& myKey,
    const ValueT& myValue,
    const uint32_t bucket_id,
    AllocatorContextT& local_allocator_ctx)
```

函数实现如下：

- line 5~14 与searchKey过程相同。组装work_queue、获取目标查找数据以及查找位置的数据

- line 17~18 比较warp各线程查找位置是否为空，即比较查找位置数据与EMPTY_KEY

- line 19~40 当前warp迭代没有找到empty slot
  \__shfl_sync 从warp最后一个lane获取next指针

  - line 21~37 如果next指针为空，则申请新的结点

- line 41~57 当前warp迭代找到empty slot
  line 42 \__ffs得到最小empty slot位置对应的lane `dest_lane`

  - line 43~57 查找目标lane在 `dest_lane`的查找位置执行最后的插入操作
    为了保证插入操作的原子性，key-value 组成64 bit，使用 atomicCAS 原语操作

    > 如果key 64bit，value 32bit，此时超过了原子类型长度，应该怎么做？
    >
    > 方案1：if (atomicCAS(dest_key, EMPTY_KEY, key) == EMPTY_KEY)  dest_value = value;
    >
    > 这样不可行，因为 atomicCAS 成功后，又来了另一个对同一key的update操作，会导致数据错误
    >
    > 方案2：key 64bit的最高位作为保留位，1表示锁，0表示未锁
    >
    > ```c++
    > lock_key = key || 0x8FFFFFFFFFFFFFFF;
    > while (dest_key & 0x7FFFFFFFFFFFFFFF == key) {
    >     if (key == atomicCAS(dest_key, key, lock_key)) {	// lock
    >         update value;
    >         atomicCAS(dest_key, lock_key, key);				// unlock
    >     }
    > }
    > ```
    
  - 插入成功则终止本迭代的目标插入操作 `to_be_inserted = false`。否则继续迭代执行

```c++
uint32_t work_queue = 0;
uint32_t last_work_queue = 0;
uint32_t next = SlabHashT::A_INDEX_POINTER;

while ((work_queue = __ballot_sync(0xFFFFFFFF, to_be_inserted))) {
    // to know whether it is a base node, or a regular node
    next = (last_work_queue != work_queue) ? SlabHashT::A_INDEX_POINTER
                                           : next;  // a successfull insertion in the warp
    uint32_t src_lane = __ffs(work_queue) - 1;
    uint32_t src_bucket = __shfl_sync(0xFFFFFFFF, bucket_id, src_lane, 32);

    uint32_t src_unit_data = (next == SlabHashT::A_INDEX_POINTER)
                                 ? *(getPointerFromBucket(src_bucket, laneId))
                                 : *(getPointerFromSlab(next, laneId));
    uint64_t old_key_value_pair = 0;

    uint32_t isEmpty = (__ballot_sync(0xFFFFFFFF, src_unit_data == EMPTY_KEY)) &
                       SlabHashT::REGULAR_NODE_KEY_MASK;
    if (isEmpty == 0) {  // no empty slot available:
      	uint32_t next_ptr = __shfl_sync(0xFFFFFFFF, src_unit_data, 31, 32);
      	if (next_ptr == SlabHashT::EMPTY_INDEX_POINTER) {
        	// allocate a new node:
        	uint32_t new_node_ptr = allocateSlab(local_allocator_ctx, laneId);

        	// TODO: experiment if it's better to use lane 0 instead
        	if (laneId == 31) {
          		const uint32_t* p = (next == SlabHashT::A_INDEX_POINTER)
                                  	? getPointerFromBucket(src_bucket, 31)
                                  	: getPointerFromSlab(next, 31);

          		uint32_t temp = atomicCAS((unsigned int*)p, SlabHashT::EMPTY_INDEX_POINTER, new_node_ptr);
          		// check whether it was successful, and
          		// free the allocated memory otherwise
          		if (temp != SlabHashT::EMPTY_INDEX_POINTER) {
            		freeSlab(new_node_ptr);
          		}
        	}
      	} else {
			next = next_ptr;
      	}
    } else {  // there is an empty slot available
      	int dest_lane = __ffs(isEmpty & SlabHashT::REGULAR_NODE_KEY_MASK) - 1;
      	if (laneId == src_lane) {
        	const uint32_t* p = (next == SlabHashT::A_INDEX_POINTER)
                                	? getPointerFromBucket(src_bucket, dest_lane)
                                	: getPointerFromSlab(next, dest_lane);

        	old_key_value_pair = atomicCAS((unsigned long long int*)p,
                      EMPTY_PAIR_64,
                      ((uint64_t)(*reinterpret_cast<const uint32_t*>(
                           reinterpret_cast<const unsigned char*>(&myValue)))
                       << 32) |
                          *reinterpret_cast<const uint32_t*>(
                              reinterpret_cast<const unsigned char*>(&myKey)));
        	if (old_key_value_pair == EMPTY_PAIR_64)
          		to_be_inserted = false;  // succesfful insertion
      	}
    }
	last_work_queue = work_queue;
}
```



## 1.3 insertPairUnique

查找是否存在目标key，如果不存在插入到empty slot，存在则无操作。

函数声明

```c++
GpuSlabHashContext<KeyT, ValueT, SlabHashTypeT::ConcurrentMap>::insertPairUnique(
    bool& to_be_inserted,
    const uint32_t& laneId,
    const KeyT& myKey,
    const ValueT& myValue,
    const uint32_t bucket_id,
    AllocatorContextT& local_allocator_ctx)
```



函数实现如下：

- line 5~14与searchKey过程相同。组装work_queue，以及获取目标查找数据
  line 20 获取目标查找key src_key 
- line 17~18 获取本次warp迭代，哪些lane查找到了empty slot
  line 21~22 获取本次warp迭代，哪个lane查找到了目标key
- line 23~25 存在目标key时，结束查找任务
- line 26~66 不存在目标key时，选择插入到empty slot
  - line 27~47 没有empty slot
    得到next指针
    - 如果next指针是空，则需要申请新的slab
    - 如果next指针不为空，则移动到next执行下次warp迭代

  - line 48~64 存在empty slot。
    最小索引的empty slot作为目标插入位置，dest_lane


```c++
uint32_t work_queue = 0;
uint32_t last_work_queue = 0;
uint32_t next = SlabHashT::A_INDEX_POINTER;
bool new_insertion = false;
while ((work_queue = __ballot_sync(0xFFFFFFFF, to_be_inserted))) {
	// to know whether it is a base node, or a regular node
	next = (last_work_queue != work_queue) ? SlabHashT::A_INDEX_POINTER
                                           : next;  // a successful insertion in the warp
    uint32_t src_lane = __ffs(work_queue) - 1;
    uint32_t src_bucket = __shfl_sync(0xFFFFFFFF, bucket_id, src_lane, 32);

    uint32_t src_unit_data = (next == SlabHashT::A_INDEX_POINTER)
                                 ? *(getPointerFromBucket(src_bucket, laneId))
                                 : *(getPointerFromSlab(next, laneId));
    uint64_t old_key_value_pair = 0;

    uint32_t isEmpty = (__ballot_sync(0xFFFFFFFF, src_unit_data == EMPTY_KEY)) &
                       SlabHashT::REGULAR_NODE_KEY_MASK;

    uint32_t src_key = __shfl_sync(0xFFFFFFFF, myKey, src_lane, 32);
    uint32_t isExisting = (__ballot_sync(0xFFFFFFFF, src_unit_data == src_key)) &
                          SlabHashT::REGULAR_NODE_KEY_MASK;
    if (isExisting) {  // key exist in the hash table
      	if (laneId == src_lane)
        	to_be_inserted = false;
    } else {
      	if (isEmpty == 0) {  // no empty slot available:
        	uint32_t next_ptr = __shfl_sync(0xFFFFFFFF, src_unit_data, 31, 32);
        	if (next_ptr == SlabHashT::EMPTY_INDEX_POINTER) {
          		// allocate a new node:
          		uint32_t new_node_ptr = allocateSlab(local_allocator_ctx, laneId);

          		if (laneId == 31) {
            		const uint32_t* p = (next == SlabHashT::A_INDEX_POINTER)
                                    		? getPointerFromBucket(src_bucket, 31)
                                    		: getPointerFromSlab(next, 31);

            		uint32_t temp = atomicCAS((unsigned int*)p, SlabHashT::EMPTY_INDEX_POINTER, new_node_ptr);
            		// check whether it was successful, and
            		// free the allocated memory otherwise
            		if (temp != SlabHashT::EMPTY_INDEX_POINTER) {
              			freeSlab(new_node_ptr);
            		}
          		}
        	} else {
          		next = next_ptr;
        	}
      	} else {  // there is an empty slot available
        	int dest_lane = __ffs(isEmpty & SlabHashT::REGULAR_NODE_KEY_MASK) - 1;
        	if (laneId == src_lane) {
          		const uint32_t* p = (next == SlabHashT::A_INDEX_POINTER)
                                  		? getPointerFromBucket(src_bucket, dest_lane)
                                  		: getPointerFromSlab(next, dest_lane);

          		old_key_value_pair = atomicCAS((unsigned long long int*)p,
                        	EMPTY_PAIR_64,
                        	((uint64_t)(*reinterpret_cast<const uint32_t*>(
                             	reinterpret_cast<const unsigned char*>(&myValue))) << 32) |
                            	*reinterpret_cast<const uint32_t*>(reinterpret_cast<const unsigned char*>(&myKey)));
          		if (old_key_value_pair == EMPTY_PAIR_64) {
            		to_be_inserted = false;  // successful insertion
            		new_insertion = true;
          		}
    		}
  		}
	}
	last_work_queue = work_queue;
}
```



























<a name="[1]">[1]</a> https://github.com/owensgroup/SlabHash

<a name="[2]">[2]</a> S. Ashkiani, M. Farach-Colton, and J. D. Owens, “A dynamic hash table for the gpu,” in 2018 IEEE International Parallel and Distributed Processing Symposium (IPDPS). IEEE, 2018, pp. 419–429.   