---
title: DiskLruCache解析
date: 2018-09-23 17:33:31
tags: [Android]
---

### 概述

>DiskLruCache，是JakeWharton大神开源的作品，用于磁盘缓存，与LruCache内存缓存相对应，都是使用LRU算法。

<!--more-->

### 获取DiskLruCache

> 因为DiskLruCache不是Android官方的，所以在Android SDK中找不到，但是得到官方的推荐。

获取方式一：

https://github.com/JakeWharton/DiskLruCache

获取方式二：

https://android.googlesource.com/platform/libcore/+/jb-mr2.0.0-release/luni/src/main/java/libcore/io/DiskLruCache.java

### DiskLruCache的使用

#### 打开缓存

```java
  /**
   * @param directory    缓存目录
   * @param appVersion   当前应用程序的版本号 
   * @param valueCount   同一个key可以对应多少个缓存文件，基本都是1
   * @param maxSize      最多可以缓存多少字节的数据
   * @throws IOException 如果读写缓存失败会抛出IO异常
   */
  public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize) throws IOException
```

#### 写入缓存

```java
// key会成为缓存文件的文件名，并且必须要和URL是一一对应的，而URL可能包含特殊字符，不能用作文件名，
// 所以对URL进行MD5编码，编码后的字符串是唯一的，并且只会包含0-F字符，符合文件命名规则
String key = generateKey(url);
DiskLruCache.Editor editor = mDiskLruCache.edit(key);
// 通过Editor获取到os是指向缓存文件的输出流，然后把想存的东西写入
OutputStream os = editor.newOutputStream(0);
// ...流操作
// 写完缓存后，调用commit()，来提交缓存；调用abort()，放弃写入的缓存
editor.commit();
// editor.abort();
```

#### 读取缓存

```java
DiskLruCache.Snapshot snapshot = mDiskLruCache.get(key);
// 通过snapshot获取到输入流，然后对流进行操作
InputStream is = snapshot.getInputStream(0);
// ...流操作
```

### journal文件解读

```java
libcore.io.DiskLruCache
1
100
1

DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```

#### journal文件头

- 第一行：固定字符串`libcore.io.DiskLruCache`
- 第二行：DiskLruCache的版本号，这个值恒为1。
- 第三行：应用程序的版本号。每当版本号改变，缓存路径下存储的所有数据都会被清空，因为DiskLruCache认为应用更新，所有的数据都应重新获取。
- 第四行：指每个key对应几个文件，一般为1。
- 第五行：空行

#### journal文件内容

- `DIRTY`：第六行以DIRTY前缀开始，后面跟着缓存文件的key，表示一个entry正在被写入。
- `CLEAN`：当写入成功，就会写入一条CLEAN记录，后面的数字记录文件的长度，如果一个key可以对应多个文件，那么就会有多个数字
- `REMOVE`：表示写入失败，或者调用remove(key)方法的时候都会写入一条REMOVE记录
- `READ`：表示一次读取记录

**NOTE**：当journal文件记录的操作次数达到2000时，就会触发重构journal的事件，来保证journal文件的大小始终在一个合理的范围内。

### DiskLruCache源码解析

#### DiskLruCache#open

```java
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize) throws IOException {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
        throw new IllegalArgumentException("valueCount <= 0");
    }

    // 检查journal.bkp文件是否存在(journal的备份文件)
    File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    if (backupFile.exists()) {
        File journalFile = new File(directory, JOURNAL_FILE);
        // 如果journal文件存在，则删除journal.bkp备份文件
        if (journalFile.exists()) {
            backupFile.delete();
        } else {
            // journal文件不存在，将bkp文件重命名为journal文件
            renameTo(backupFile, journalFile, false);
        }
    }

    // Prefer to pick up where we left off.
    DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    // 判断journal文件是否存在
    if (cache.journalFile.exists()) {
        try {
            // 读取journal文件
            cache.readJournal();
            cache.processJournal();
            cache.journalWriter = new BufferedWriter(
                    new OutputStreamWriter(new FileOutputStream(cache.journalFile, true), Util.US_ASCII));
            return cache;
        } catch (IOException journalIsCorrupt) {
            System.out
                    .println("DiskLruCache "
                            + directory
                            + " is corrupt: "
                            + journalIsCorrupt.getMessage()
                            + ", removing");
            cache.delete();
        }
    }

    // journal文件不存在，则创建缓存目录，重新构造DiskLruCache
    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    // 重新创建journal文件
    cache.rebuildJournal();
    return cache;
}
```

##### rebuildJournal()：

```java
// 重新创建journal文件
private synchronized void rebuildJournal() throws IOException {
    if (journalWriter != null) {
        journalWriter.close();
    }

    // 创建journal.tmp文件
    Writer writer = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
    try {
        // 写入文件头(5行)
        writer.write(MAGIC);
        writer.write("\n");
        writer.write(VERSION_1);
        writer.write("\n");
        writer.write(Integer.toString(appVersion));
        writer.write("\n");
        writer.write(Integer.toString(valueCount));
        writer.write("\n");
        writer.write("\n");

        // 遍历lruEntries
        for (DiskLruCache.Entry entry : lruEntries.values()) {
            if (entry.currentEditor != null) {
                writer.write(DIRTY + ' ' + entry.key + '\n');
            } else {
                writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            }
        }
    } finally {
        writer.close();
    }

    if (journalFile.exists()) {
        renameTo(journalFile, journalFileBackup, true);
    }
    // 将journal文件重命名为journal文件
    renameTo(journalFileTmp, journalFile, false);
    journalFileBackup.delete();

    journalWriter = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```

##### readJournal()：

```java
// 读取journal文件
private void readJournal() throws IOException {
    StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
    try {
        String magic = reader.readLine();
        String version = reader.readLine();
        String appVersionString = reader.readLine();
        String valueCountString = reader.readLine();
        String blank = reader.readLine();
        // 校验journal文件头
        if (!MAGIC.equals(magic)
                || !VERSION_1.equals(version)
                || !Integer.toString(appVersion).equals(appVersionString)
                || !Integer.toString(valueCount).equals(valueCountString)
                || !"".equals(blank)) {
            throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
                    + valueCountString + ", " + blank + "]");
        }

        int lineCount = 0;
        while (true) {
            try {
                // 读取journal文件内容
                readJournalLine(reader.readLine());
                lineCount++;
            } catch (EOFException endOfJournal) {
                break;
            }
        }
        redundantOpCount = lineCount - lruEntries.size();
    } finally {
        Util.closeQuietly(reader);
    }
}
```

##### readJournalLine()：

```java
// 读取journal文件内容
private void readJournalLine(String line) throws IOException {
    int firstSpace = line.indexOf(' ');
    if (firstSpace == -1) {
        throw new IOException("unexpected journal line: " + line);
    }

    int keyBegin = firstSpace + 1;
    int secondSpace = line.indexOf(' ', keyBegin);
    final String key;
    if (secondSpace == -1) {
        // 获取缓存key
        key = line.substring(keyBegin);
        // 如果是REMOVE记录，则调用lruEntries.remove(key)
        if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
            lruEntries.remove(key);
            return;
        }
    } else {
        key = line.substring(keyBegin, secondSpace);
    }

    DiskLruCache.Entry entry = lruEntries.get(key);
    // 如果该key没有加入到lruEntries，则创建并加入
    if (entry == null) {
        entry = new DiskLruCache.Entry(key);
        lruEntries.put(key, entry);
    }
    // 处理CLEAN记录
    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
        String[] parts = line.substring(secondSpace + 1).split(" ");
        entry.readable = true;
        entry.currentEditor = null;
        entry.setLengths(parts);
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
        // 处理DIRTY记录
        entry.currentEditor = new DiskLruCache.Editor(entry);
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
        // This work was already done by calling lruEntries.get().
    } else {
        throw new IOException("unexpected journal line: " + line);
    }
}
```

##### processJournal()：

```java
// 处理journal文件
private void processJournal() throws IOException {
    deleteIfExists(journalFileTmp);
    for (Iterator<DiskLruCache.Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
        DiskLruCache.Entry entry = i.next();
        if (entry.currentEditor == null) {
            for (int t = 0; t < valueCount; t++) {
                // 统计所有可用cache占据的容量
                size += entry.lengths[t];
            }
        } else {
            // 删除非法DIRTY状态的entry,并删除对应的文件
            entry.currentEditor = null;
            for (int t = 0; t < valueCount; t++) {
                deleteIfExists(entry.getCleanFile(t));
                deleteIfExists(entry.getDirtyFile(t));
            }
            i.remove();
        }
    }
}
```

#### DiskLruCache#edit

```java
private synchronized DiskLruCache.Editor edit(String key, long expectedSequenceNumber) throws IOException {
    checkNotClosed();
    // 验证key，必须是字母、数字、下划线、横线(-)组成，且长度在1-120之间
    validateKey(key);
    // 获取实体
    DiskLruCache.Entry entry = lruEntries.get(key);
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
            || entry.sequenceNumber != expectedSequenceNumber)) {
        return null; // Snapshot is stale.
    }
    // 实体不存在，则创建一个Entry并加入到lruEntries
    if (entry == null) {
        entry = new DiskLruCache.Entry(key);
        lruEntries.put(key, entry);
    } else if (entry.currentEditor != null) {
        return null; // 该实体正在被编辑
    }

    // 创建Editor并赋值给entry.currentEditor
    DiskLruCache.Editor editor = new DiskLruCache.Editor(entry);
    entry.currentEditor = editor;

    // Flush the journal before creating files to prevent file leaks.
    journalWriter.write(DIRTY + ' ' + key + '\n');
    journalWriter.flush();
    return editor;
}
```

##### Editor#newOutputStream

```java
/**
 * 获取一个文件输入流
 * @param index 缓存文件索引，一个key可能对应多个文件，当对应一个文件时，只要传0
 */
public OutputStream newOutputStream(int index) throws IOException {
    synchronized (DiskLruCache.this) {
        if (entry.currentEditor != this) {
            throw new IllegalStateException();
        }
        if (!entry.readable) {
            written[index] = true;
        }
        // 获取dirty file对象，这是一个中转文件，文件名格式key.index.tmp
        File dirtyFile = entry.getDirtyFile(index);
        FileOutputStream outputStream;
        try {
            outputStream = new FileOutputStream(dirtyFile);
        } catch (FileNotFoundException e) {
            // Attempt to recreate the cache directory.
            directory.mkdirs();
            try {
                outputStream = new FileOutputStream(dirtyFile);
            } catch (FileNotFoundException e2) {
                // We are unable to recover. Silently eat the writes.
                return NULL_OUTPUT_STREAM;
            }
        }
        return new DiskLruCache.Editor.FaultHidingOutputStream(outputStream);
    }
}
```

##### Editor#commit

```java
public void commit() throws IOException {
    // 判断是否发生错误
    if (hasErrors) {
        completeEdit(this, false);
        remove(entry.key); // The previous entry is stale.
    } else {
        completeEdit(this, true);
    }
    committed = true;
}
```

##### Editor#completeEdit

```java
private synchronized void completeEdit(DiskLruCache.Editor editor, boolean success) throws IOException {
    DiskLruCache.Entry entry = editor.entry;
    if (entry.currentEditor != editor) {
        throw new IllegalStateException();
    }

    // 判断是否写入成功，且是第一次写入
    if (success && !entry.readable) {
        for (int i = 0; i < valueCount; i++) {
            if (!editor.written[i]) {
                editor.abort();
                throw new IllegalStateException("Newly created entry didn't create value for index " + i);
            }
            if (!entry.getDirtyFile(i).exists()) {
                editor.abort();
                return;
            }
        }
    }

    for (int i = 0; i < valueCount; i++) {
        File dirty = entry.getDirtyFile(i);
        if (success) {
            if (dirty.exists()) {
                File clean = entry.getCleanFile(i);
                // 将dirtyFile重命名为cleanFile，更新size的大小
                dirty.renameTo(clean);
                long oldLength = entry.lengths[i];
                long newLength = clean.length();
                entry.lengths[i] = newLength;
                size = size - oldLength + newLength;
            }
        } else {
            deleteIfExists(dirty);
        }
    }

    redundantOpCount++;
    entry.currentEditor = null;
    // 如果成功，写入一条CLEAN记录
    if (entry.readable | success) {
        entry.readable = true;
        journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        if (success) {
            entry.sequenceNumber = nextSequenceNumber++;
        }
    } else {
        // 否则，写入一条REMOVE记录
        lruEntries.remove(entry.key);
        journalWriter.write(REMOVE + ' ' + entry.key + '\n');
    }
    journalWriter.flush();

    if (size > maxSize || journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }
}
```

#### DiskLruCache#get

```java
public synchronized DiskLruCache.Snapshot get(String key) throws IOException {
    checkNotClosed();
    validateKey(key);
    DiskLruCache.Entry entry = lruEntries.get(key);
    if (entry == null) {
        return null;
    }

    if (!entry.readable) {
        return null;
    }

    // Open all streams eagerly to guarantee that we see a single published
    // snapshot. If we opened streams lazily then the streams could come
    // from different edits.
    InputStream[] ins = new InputStream[valueCount];
    try {
        for (int i = 0; i < valueCount; i++) {
            ins[i] = new FileInputStream(entry.getCleanFile(i));
        }
    } catch (FileNotFoundException e) {
        // A file must have been deleted manually!
        for (int i = 0; i < valueCount; i++) {
            if (ins[i] != null) {
                Util.closeQuietly(ins[i]);
            } else {
                break;
            }
        }
        return null;
    }

    redundantOpCount++;
    // 往journal文件写入一条READ记录
    journalWriter.append(READ + ' ' + key + '\n');
    if (journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }
	// 将cleanFile的FileInputStream封装成Snapshot并返回
    return new DiskLruCache.Snapshot(key, entry.sequenceNumber, ins, entry.lengths);
}
```



### 参考链接

1. [Android DiskLruCache 源码解析 硬盘缓存的绝佳方案](https://blog.csdn.net/lmj623565791/article/details/47251585)
2. [Android DiskLruCache完全解析，硬盘缓存的最佳方案](https://blog.csdn.net/guolin_blog/article/details/28863651)
3. [优雅的构建 Android 项目之磁盘缓存](https://www.jianshu.com/p/ed5668590900)