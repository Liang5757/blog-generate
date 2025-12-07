---
title: 为什么MP4视频越长起播越慢
date: 2025-03-20 01:04:57
tags:
  - 视频
categories:
  - 视频
---
## 利用在线网站解析分析

> 在线解析mp4：https://www.onlinemp4parser.com/

<!--more-->

### 用例信息

| Fields               | Value    |
| :------------------- | :------- |
| Movie duration       | 3:08:22  |
| Video duration       | 3:08:22  |
| Audio duration       | 3:08:22  |
| Video resolution     | 1024x576 |
| Frame rate (fps)     | 30.00    |
| Video bitrate (kbps) | 462.71   |

全展开后发现是下图红框数据非常长

![image-20241029174327954](./为什么MP4视频越长越慢/image-20241029174327954.png)

## stbl（Sample Table Box）

- stsd（Sample Description Box）：给出视频、音频的编码、宽高、音量等信息，以及每个sample中包含多少个frame；
- stts（Decoding Time to Sample Box）：每个sample的时长；
- **stss**（Sync Sample Box）：哪些sample是关键帧；
- **ctts**（Composition Time to Sample Box）：帧解码到渲染的时间差值，通常用在B帧的场景；
- **stsc**（Sample To Chunk Box）：每个thunk中包含几个sample；
- **stsz**（Sample Size Boxes）：每个sample的size（单位是字节）；
- **stco**（Chunk Offset Box）：thunk在文件中的偏移；

这边就只详细解释长数据的部分

### stss（Sync Sample Box）

包含了每个关键帧在所有帧中的数组下标+1，即关键帧队列从1开始算起，主要用来推导每个帧的时长。在音频track中不存在这个box。

```cpp
aligned(8) class TimeToSampleBox extends FullBox(’stts’, version = 0, 0) {
	unsigned int(32)  entry_count;
	int i;
	for (i=0; i < entry_count; i++) {
		unsigned int(32)  sample_count;
		unsigned int(32)  sample_delta;
	}
}
```

- entry_count：entry的条目数，可以认为是关键帧的数目；
- sample_number：关键帧对应的sample的序号；（从1开始计算）

可以看到下述例子，有1796个关键帧`，随着视频变长，`stss`数据变大

![image-20241030115838585](./为什么MP4视频越长越慢/image-20241030115838585.png)

### ctts（Composition Time to Sample Box）

用于存储每个采样的构建时间戳（CTS，Composition Time Stamp），即从解码（dts）到渲染（pts）之间的差值，pts（n） = dts（n）+ offset（n） 。

对于只有I帧、P帧的视频来说，解码顺序、渲染顺序是一致的，此时，ctts没必要存在。

```c++
aligned(8) class CompositionOffsetBox extends FullBox(‘ctts’, version = 0, 0) { unsigned int(32) entry_count;
   int i;
   for (i=0; i < entry_count; i++) {
      unsigned int(32)  sample_count;
      unsigned int(32)  sample_offset;
   }
}
```

- entry_count：表示 `ctts` 盒子中`entry`的数量。
- entries
  - sample_count：表示这个条目中连续的样本数量，通常是指具有相同显示时间的帧的数量。
  - sample_offset：表示这些样本的构建时间戳的偏移量，通常指示相对于该样本的解码时间戳的时间差。它可以是正值或负值，表示样本需要提前或延后显示。

下图可以看到有338443个`entry`，随着视频变长，ctts变大

![image-20241030152427905](./为什么MP4视频越长越慢/image-20241030152427905.png)

### stsc（Sample To Chunk Box）

Trunk和Sample的映射关系，这个Box就是说明哪些sample可以划分为一个Trunk

```c++
aligned(8) class SampleToChunkBox
	extends FullBox(‘stsc’, version = 0, 0) { 
	unsigned int(32) entry_count;
	for (i=1; i u entry_count; i++) {
		unsigned int(32) first_chunk;
		unsigned int(32) samples_per_chunk; 
		unsigned int(32) sample_description_index;
	}
}
```

- entry_count：描述样本到块的映射关系的条目数量
- first_chunk：表示块中第一个样本的索引（从 1 开始计数）
- samples_per_chunk：表示每个块中包含的sample数量
- sample_description_index：指向 stsd 中 sample description 的索引值，默认为1

可以看到下述例子，有191384个entry，随着视频变长，`stsc`数据变大

![image-20241030175323810](./为什么MP4视频越长越慢/image-20241030175323810.png)

### stsz（Sample Size Boxes）

用于存储每个sample的大小信息，帮助播放器确定在解码过程中需要读取多少字节的数据。

```c++
aligned(8) class SampleSizeBox extends FullBox(‘stsz’, version = 0, 0) { 
	unsigned int(32) sample_size;
	unsigned int(32) sample_count;
	if (sample_size==0) {
		for (i=1; i u sample_count; i++) {
			unsigned int(32)  entry_size;
		}
	}
}
```

- sample_count：当前track里面的sample数目
- sample_size：sample的大小，每个sample的大小都会列出

可以看到下述例子，有339059个entry，随着视频变长，`stsz`数据变大

![image-20241031105426056](./为什么MP4视频越长越慢/image-20241031105426056.png)

### stco（Chunk Offset Box）

用于存储每个块（chunk）在文件中的偏移量，帮助播放器快速找到chunk的存储位置

```c++
# Box Type: ‘stco’, ‘co64’
# Container: Sample Table Box (‘stbl’) Mandatory: Yes
# Quantity: Exactly one variant must be present

aligned(8) class ChunkOffsetBox
	extends FullBox(‘stco’, version = 0, 0) { 
	unsigned int(32) entry_count;
	for (i=1; i u entry_count; i++) {
		unsigned int(32)  chunk_offset;
	}
}

aligned(8) class ChunkLargeOffsetBox
	extends FullBox(‘co64’, version = 0, 0) { 
	unsigned int(32) entry_count;
	for (i=1; i u entry_count; i++) {
		unsigned int(64)  chunk_offset;
	}
}
```

- entry_count：chunk的数量
- chunk_offset：chunk在文件中的位置

可以看到下述例子，有243367个entry，随着视频变长，`stco`数据变大

![image-20241031193839595](./为什么MP4视频越长越慢/image-20241031193839595.png)

## 总结

moov的大小总共：9305337B（8.87MB）

第一个trak（这个例子有两个）：
stss占：7200B（0.006MB，0.077%）
ctts占：2707560B（2.582MB，29.09%）
stsc占：2296624B（2.190MB，24.68%）
stsz占：1356256B （1.293MB，14.58%）
stco占：973484B（0.928MB，10.46%）

时间越长，会导致moov变大，主要是会有所有chunk和sample的描述

## 参考

[[5分钟入门MP4文件格式](https://www.cnblogs.com/chyingp/p/mp4-file-format.html)](https://www.cnblogs.com/chyingp/p/mp4-file-format.html)