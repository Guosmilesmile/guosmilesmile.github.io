---
title: Flink 读取HDFS中的数据源码解析
date: 2019-06-06 21:13:35
tags:
categories:
    - Flink
---


### FileInputFormat.java


主要是createInputSplits这个函数

源码中会得到文件系统(这里会得到HDFS)，和文件的总大小。然后考虑了输入文件时文件夹，输入文件不可切分等情况，然后根据HDFS的分块(block)大小，文件总大小和Source的并行度来计算每个split的大小，每个split会存储对应的HDFS的block信息，例如block在哪个host上。注意：每个split的大小是不能超过HDFS中分块(block)的大小的。得到每个split的大小后就可以根据HDFS的各个分块信息来构造inputSplits了。


* 获取最小的split，默认是1
* 如果输入的是文件夹，获取文件夹下的所有文件，如果目标是文件，直接加入。并得到总的文件大小
* 判断文件是否能切割读取

```java

@Override
	public FileInputSplit[] createInputSplits(int minNumSplits) throws IOException {
		if (minNumSplits < 1) {
			throw new IllegalArgumentException("Number of input splits has to be at least 1.");
		}
		
		// take the desired number of splits into account
		minNumSplits = Math.max(minNumSplits, this.numSplits);
		
		final List<FileInputSplit> inputSplits = new ArrayList<FileInputSplit>(minNumSplits);

		// get all the files that are involved in the splits
		List<FileStatus> files = new ArrayList<>();
		long totalLength = 0;

        //如果输入的是文件夹，获取文件夹下的所有文件，如果目标是文件，直接加入。并得到总的文件大小
		for (Path path : getFilePaths()) {
			final FileSystem fs = path.getFileSystem();
			final FileStatus pathFile = fs.getFileStatus(path);

			if (pathFile.isDir()) {
				totalLength += addFilesInDir(path, files, true);
			} else {
			    //判断文件是否能切割
				testForUnsplittable(pathFile);

				files.add(pathFile);
				totalLength += pathFile.getLen();
			}
		}

        //判断文件是否能切割读取
		// returns if unsplittable
		if (unsplittable) {
			int splitNum = 0;
			for (final FileStatus file : files) {
				final FileSystem fs = file.getPath().getFileSystem();
				final BlockLocation[] blocks = fs.getFileBlockLocations(file, 0, file.getLen());
				Set<String> hosts = new HashSet<String>();
				for(BlockLocation block : blocks) {
					hosts.addAll(Arrays.asList(block.getHosts()));
				}
				long len = file.getLen();
				if(testForUnsplittable(file)) {
					len = READ_WHOLE_SPLIT_FLAG;
				}
				FileInputSplit fis = new FileInputSplit(splitNum++, file.getPath(), 0, len,
						hosts.toArray(new String[hosts.size()]));
				inputSplits.add(fis);
			}
			return inputSplits.toArray(new FileInputSplit[inputSplits.size()]);
		}
		

		final long maxSplitSize = totalLength / minNumSplits + (totalLength % minNumSplits == 0 ? 0 : 1);

		// now that we have the files, generate the splits
		int splitNum = 0;
		for (final FileStatus file : files) {

			final FileSystem fs = file.getPath().getFileSystem();
			final long len = file.getLen();
			//获取hdfs文件的block
			final long blockSize = file.getBlockSize();
			//minSplitSize默认设置为0，可以通过配置设置，但是要小于block大小，如果设置的比block，会被强制改为bolckSize
			final long minSplitSize;
			if (this.minSplitSize <= blockSize) {
				minSplitSize = this.minSplitSize;
			}
			else {
				if (LOG.isWarnEnabled()) {
					LOG.warn("Minimal split size of " + this.minSplitSize + " is larger than the block size of " + 
						blockSize + ". Decreasing minimal split size to block size.");
				}
				minSplitSize = blockSize;
			}

            //最终切割大小，如果小于block设为该数据，如果大于，以block大小为切割大小
			final long splitSize = Math.max(minSplitSize, Math.min(maxSplitSize, blockSize));
			final long halfSplit = splitSize >>> 1;
            //每个分块的最大大小是splitSize*1.1 
			final long maxBytesForLastSplit = (long) (splitSize * MAX_SPLIT_SIZE_DISCREPANCY);

			if (len > 0) {
                // 将数据切为多个block的数组
				// get the block locations and make sure they are in order with respect to their offset
				final BlockLocation[] blocks = fs.getFileBlockLocations(file, 0, len);
				Arrays.sort(blocks);

				long bytesUnassigned = len;
				long position = 0;

				int blockIndex = 0;
                //开始分配读取的offset，读取完整的splitSize
				while (bytesUnassigned > maxBytesForLastSplit) {
					// get the block containing the majority of the data
					blockIndex = getBlockIndexForPosition(blocks, position, halfSplit, blockIndex);
					// create a new split
					FileInputSplit fis = new FileInputSplit(splitNum++, file.getPath(), position, splitSize,
						blocks[blockIndex].getHosts());
					inputSplits.add(fis);

					// adjust the positions
					position += splitSize;
					bytesUnassigned -= splitSize;
				}
                //读取剩余未不够完整splitSize的数据
				// assign the last split
				if (bytesUnassigned > 0) {
					blockIndex = getBlockIndexForPosition(blocks, position, halfSplit, blockIndex);
					final FileInputSplit fis = new FileInputSplit(splitNum++, file.getPath(), position,
						bytesUnassigned, blocks[blockIndex].getHosts());
					inputSplits.add(fis);
				}
			} else {
				// special case with a file of zero bytes size
				final BlockLocation[] blocks = fs.getFileBlockLocations(file, 0, 0);
				String[] hosts;
				if (blocks.length > 0) {
					hosts = blocks[0].getHosts();
				} else {
					hosts = new String[0];
				}
				final FileInputSplit fis = new FileInputSplit(splitNum++, file.getPath(), 0, 0, hosts);
				inputSplits.add(fis);
			}
		}

		return inputSplits.toArray(new FileInputSplit[inputSplits.size()]);
	}


```

```
举例说明：文件A大小为256M，存储在HDFS上，HDFS的分块大小为64M，则A文件会被分割成4个block，每个block都是64M。
当我们使用flink读取文件A的时候，如果设置的并行度为2，则源码中：
final long splitSize = Math.max(minSplitSize, Math.min(maxSplitSize, blockSize));
minSplitSize默认是0，maxSplitSize = 256 / 2 = 128M(总大小/并行度),blockSize=64M，算出来的splitSize就是64M。
如果我们读取文件A的时候，并行度设置为8:
minSplitSize默认是0，maxSplitSize = 256 / 8 = 32M(总大小/并行度),blockSize=64M，算出来的splitSize就是32M,相当于一个HDFS中的block(64M)会切分成Flink中的两个split(32M)，当然，不是整数倍时，里面也有相应的逻辑来处理。
得到inputSplits后，会根据它来初始化ExecutionJobVertex中的splitAssigner，最终SourceTask执行的时候，就会请求来得到一个split。
```

#### 判断文件是否能切割
获取文件的后缀，根据后缀去判断文件是否能切割，如果文件是的后缀如下，不能切割，只能一个solt读取（几乎都是压缩包）
* xz
* deflate
* gz
* gzip
* bz2
```java
protected boolean testForUnsplittable(FileStatus pathFile) {
		if(getInflaterInputStreamFactory(pathFile.getPath()) != null) {
			unsplittable = true;
			return true;
		}
		return false;
	}


protected static InflaterInputStreamFactory<?> getInflaterInputStreamFactory(String fileExtension) {
		synchronized (INFLATER_INPUT_STREAM_FACTORIES) {
			return INFLATER_INPUT_STREAM_FACTORIES.get(fileExtension);
		}
	}
	
```

### Reference 
https://blog.csdn.net/u013036495/article/details/88349290