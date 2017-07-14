---
layout: post
title:  "NSMutableDictionary setObject:forKey:"
date:   2017-07-13 18:09:00 +0800
categories: jekyll update
---

NSMutableDictionary内部其实会维护一个哈希表**(map)**，哈希表会用到三个数据结构分别是**_GSIMapTable**，**_GSIMapBucket**，**_GSIMapNode**：，结构如下：

```
typedef struct _GSIMapTable GSIMapTable_t;
typedef GSIMapTable_t *GSIMapTable;
struct	_GSIMapTable {
  NSZone	*zone;
  uintptr_t	nodeCount;	/* Number of used nodes in map.	*/
  uintptr_t	bucketCount;	/* Number of buckets in map.	*/
  GSIMapBucket	buckets;	/* Array of buckets.		*/
  GSIMapNode	freeNodes;	/* List of unused nodes.	*/
  uintptr_t	chunkCount;	/* Number of chunks in array.	*/
  GSIMapNode	*nodeChunks;	/* Chunks of allocated memory.	*/
  uintptr_t	increment;
  bool	extra;
};

typedef struct _GSIMapBucket GSIMapBucket_t;
typedef GSIMapBucket_t *GSIMapBucket;
struct	_GSIMapBucket {
  uintptr_t	nodeCount;	/* Number of nodes in bucket.	*/
  GSIMapNode	firstNode;	/* The linked list of nodes.	*/
};
typedef struct _GSIMapNode GSIMapNode_t;
typedef GSIMapNode_t *GSIMapNode;
struct	_GSIMapNode {
  GSIMapNode	nextInBucket;	/* Linked list of bucket.	*/
  GSIMapKey	key;
#if	GSI_MAP_HAS_VALUE
  GSIMapVal	value;
#endif
};
```

在NSMutableDictionary初始化时，会初始化这个map，初始完结构如下图：
![初始化](../resources/NSDictionary/01.png)
下面往里面添加一个key-value键值对，如：`[xxx setObject:obj1 forKey:key1]`，执行完之后再看下结构图：
![obj1_key1](../resources/NSDictionary/02.png)
`[xxx setObject:obj2 forKey:key2]` 如图：
![obj2_key2](../resources/NSDictionary/03.png)
`[xxx setObject:obj3 forKey:key3]` 如图：
![obj2_key3](../resources/NSDictionary/04.png)