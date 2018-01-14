# leveldb_comment
详细且纯粹的leveldb源码注解，注解总结链接：https://TitenWang.github.io

# 注解进度

2017-12-24  注解用于封装leveldb所有操作结果的Status模块(util/status.cc、include/leveldb/status.h)，内存管理模块(util/arena.cc、util/arena.h)，整型序列化模块(util/coding.cc、util/coding.h)。

2017-12-30  注解操作日志模块(db/log_format.h、db/log_writer.cc、db/log_writer.h、db/log_reader.cc、db/log_reader.h)、LRUCache模块(util/cache.cc、include/leveldb/cache.h)

2017-12-31  注解布隆过滤器模块(util/bloom.cc、util/bloom.h)，memtable模块(db/memtable.cc、db/memtable.h)，跳表实现(db/skiplist.h)，leveldb内部多种key封装(db/dbformat.cc、db/dbformat.h)

2018-01-03  注解Block模块(table/block.h、table/block.cc、table/block_builder.h、table/block_builder.cc)，BlockHandle/Footer模块(table/format.h、table/format.cc)

2018-01-06  注解二层迭代器(table/two_level_iterator.h、table/two_level_iterator.cc)，Table模块(table/table.cc、include/leveldb/table.h、table/table_builder.cc、include/leveldb/talbe_builder.h)，FilterBlock模块(table/filter_block.h、table/filter_block.cc)

2018-01-14  注解版本变更信息类VersionEdit(db/version_edit.cc、db/version_edit.h)，用于构造Version的类VersionSet::Builder(db/version_set.cc、db/version_set.h)

# 注解风格
注解风格和leveldb本身的注释风格保持一致。以table/block.h的注释为例，如下：
```c
// Copyright (c) 2011 The LevelDB Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file. See the AUTHORS file for names of contributors.

#ifndef STORAGE_LEVELDB_TABLE_BLOCK_H_
#define STORAGE_LEVELDB_TABLE_BLOCK_H_

#include <stddef.h>
#include <stdint.h>
#include "leveldb/iterator.h"

namespace leveldb {

struct BlockContents;
class Comparator;

// 一个Block的内存布局：
// ------------------
// |    Record 1    |
// ------------------
// |    Record 2    |
// ------------------
// |      ....      |
// ------------------
// |    Record n    |
// ------------------
// |    Restart 1   |
// ------------------
// |    Restart 2   |
// ------------------
// |      ....      |
// ------------------
// |    Restart m   |
// ------------------
// |  num_restarts  |
// ------------------
// 从上面的内存布局可以看到，一个Block内部可以分成两个部分，前面的Record数组
// 存放的是一些K-V记录，其顺序是根据Key值由小到大排列的。后面则是一个称为
// "重启点"的数组，存储是距离block开始处的偏移值，存放了Record数组中一些记录位置。
// "重启点"是干什么的呢？我们知道Block内部的k-v记录是从小到大排列的，这样的话，
// 相邻的两条或者多条记录的key值很有可能存在重叠部分，比如上图中的Record 1的
// key值为"my friend"，Record 2的key值为"my girlfriend"，那么两者就存在重叠部分
// "my "，为了减少key的存储量，Record 2中的key可能只存储和Record 1的key的不同部分，
// 即"girlfriend"，两者的相同部分则可以从Record 1中获取。基于上述的设计，"重启点"的
// 意思是：从这条记录开始，不再采取只存储不同的key部分，而是记录这条记录完整的
// key值，不管该记录的key值是否前一条记录有相同部分，例如Record 3就是一个重启点，
// 且他的key值为"your friend" 或者"my monther"，都将记录完整的key值，而不只是
// 和前一个记录的不同部分。而num_restarts则是"重启点"的个数。

// 前面我们说过，一个Record记录包含了k-v值，并且其中存储的key可能不是完整的key值，
// 而只是key的一部分值，那么为了适应这种设计，Record记录本身的内存布局是怎么样的呢？
// ----------------------------------------------------------
// |key共享长度|key非共享长度|value长度|key非共享内容|value内容|
// ----------------------------------------------------------
// 以上面的Record 1和Record 2为例，对于Record 2来说，key共享长度为"my "的长度，为3：
// key非共享长度为"girlfriend"的长度，为10，value长度的就是k-v的v的长度，key非共享内容
// 就是"girlfriend"，value内容就是k-v的v的内容。
class Block {
 public:
  // Initialize the block with the specified contents.
  explicit Block(const BlockContents& contents);

  ~Block();

  size_t size() const { return size_; }
  Iterator* NewIterator(const Comparator* comparator);

 private:

  // NumRestarts()方法从data_数组中根据block的内存布局获取到重启点的个数。
  uint32_t NumRestarts() const;

  // data_就是block存放具体内容的内存首地址。
  const char* data_;

  // size_就是block的大小，也就是data_所指向的内存的大小。
  size_t size_;

  // restart_offset_就是“重启点”数组首地址在整个Block内的偏移。
  uint32_t restart_offset_;     // Offset in data_ of restart array

  // 用来表示data_数组指向的内存是不是动态申请的，如果说是，则为true。
  bool owned_;                  // Block owns data_[]

  // No copying allowed
  Block(const Block&);
  void operator=(const Block&);

  // Iter是Block的内部迭代器。
  class Iter;
};

}  // namespace leveldb

#endif  // STORAGE_LEVELDB_TABLE_BLOCK_H_
```
