---
layout: post
title:  "NSMutableDictionary setObject:forKey:"
date:   2017-07-13 18:09:00 +0800
categories: jekyll update
---

## 内存结构
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
![初始化](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/NSDictionary/01.png)
下面往里面添加一个key-value键值对，如：`[xxx setObject:obj1 forKey:key1]`，执行完之后再看下结构图：
![obj1_key1](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/NSDictionary/02.png)
`[xxx setObject:obj2 forKey:key2]` 如图：
![obj2_key2](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/NSDictionary/03.png)
`[xxx setObject:obj3 forKey:key3]` 如图：
![obj2_key3](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/NSDictionary/04.png)

### 伪代码实现
现在用伪代码来实现下里面的流程：

```
- (void)setObject:(id)obj forKey:(id)key
{
    if (!obj || !key) {
        crash();
        return;
    }
    
    GSIMapNode node;
    //第一步 寻找node
    //find bucket by key.hash
    GSIMapBucket bucket = map.bucktes + key.hash % map.bucketCount;
    //find node from bucket by isEqual:
    node = nodeForKeyInBucket(bucket, key);
    
    if (node) {
        //第二步 如果node存在直接赋值。
        [obj retain];
        [node->value release];
        node->value = obj;
    } else {
        //第二步 从未使用的nodes中拿个node出来，并赋值。
        node = map->freeNodes;
        if (!node) {
            //这里没有实现伪代码，主要功能如下：
            //创建新的map->nodeChunks，并将旧的nodeChunks赋值给新的，然后删除旧的。
            //根据新创建的nodeChunks再新创建nodes，并将nodes和nodeChunks一起关联。
            //将map->freeNodes指向新创建出来的nodes。
            GSIMapMoreNodes(map, map->nodeCount < map->increment ? 0: map->increment);
            node = map->freeNodes;
        }
        map->freeNodes = node->nextInBucket;
        
        node->key = [key copyWithZone:map->zone];
        node->value = [value retain];
        node->nextInBucket = 0;
        
        if (map->nodeCount * 3 >= map->bucketCount * 4) {
            //这个代码没有实现伪代码，主要功能如下：
            //这里的意思就是根据传入的数值重新创建bucket数组，并将node->buckets中的值都复制到新创建的bucket的数组中。
            //然后将新创建的bucket数组赋值给node->buckets，并将之前旧的buckets删除。
            GSIMapResize(map, map->nodeCount * 3 / 4 + 1);
        }
        
        //第三步 将node插入到对应的bucket中。
        node->nextInBucket = bucket->firstNode;
        bucket->firstNode = node;
        bucket->nodeCount += 1;
        map->nodeCount++;
    }
}

GSIMapNode nodeForKeyInBucket(GSIMapBucket bucket, id key)
{
    GSIMapNode	node = bucket->firstNode;
    while ((node != 0) && ![node->key isEqual:key])
    {
        node = node->nextInBucket;
    }
    return node;
}
```

### 总结
通过以上的伪代码分析和内存结构分析可以知道：

1. `setObject:forKey:`会根据key和object生成一个node，该node会和map->buckets关联起来。
也就是说map->buckets其实就是负责管理外面设置的对象的。

2. 在存key的时候会用到的方法：hash，copyWithZone:，isEqual:。

3. 在字典销毁时会以map->nodeChunks为开始，把创建的GSIMapNode一个一个free掉。