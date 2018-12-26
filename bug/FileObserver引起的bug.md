# FileObserver引起的bug

## 前言

最近做文件下载缓存的时候,有这么一个需求,缓存文件有一个最大值限制,如果文件下载下来要超过缓存的最大值,那么就不进行下载.

## 我的方案

1. 使用固定核心线程数的线程池执行下载任务
2. 每次下载文件之前,先获取文件长度,看当前文件大小加上本地已有的文件大小会不会超出最大缓存大小.
3. 因为三个线程并行下载,可能三个线程同时走到判断大小的位置,如果都判断没有超过最大值就进行下载,那么可能下载后就超出大小了.
4. 为了解决第3点提到的问题,使用了``FileObserver``对缓存文件夹的相关事件进行观察.如果下载完成后的总的文件大小超过了最大缓存size,就删除刚刚下载的文件.

## FileObserver的使用

初始化``FileObserver``对象,复写``onEvent``方法,并开启监听,代码片段如下.

```java
private void initFileObserver() {
    mObserver = new FileObserver(mCacheDir) {
        @Override
        public void onEvent(int event, @Nullable String path) {
            switch (event) {
                case FileObserver.CLOSE_WRITE:
                    Log.i(TAG, "onEvent: CLOSE_WRITE," + path);
                    File newFile = null;
                    if (path != null) {
                        newFile = new File(mCacheDir, path);
                    }
                    mExecutor.submit(new CountFileCallable(newFile));
                    break;
                case FileObserver.DELETE:
                    Log.i(TAG, "onEvent: DELETE," + path);
                    mExecutor.submit(new CountFileCallable(null));
                    break;
                default:
            }
        }
    };
    mObserver.startWatching();
}
```

``FileObserver``的实现基本都是本地方法,大致是开启一个线程对文件的各种事件监听,``java``层就只是实现了观察者模式.

``FileObserver``可以监听的事件类型也很多,大概一般的文件操作都有对应的事件,如下

```java
/** Event type: Data was read from a file */
public static final int ACCESS = 0x00000001;
/** Event type: Data was written to a file */
public static final int MODIFY = 0x00000002;
/** Event type: Metadata (permissions, owner, timestamp) was changed explicitly */
public static final int ATTRIB = 0x00000004;
/** Event type: Someone had a file or directory open for writing, and closed it */
public static final int CLOSE_WRITE = 0x00000008;
/** Event type: Someone had a file or directory open read-only, and closed it */
public static final int CLOSE_NOWRITE = 0x00000010;
/** Event type: A file or directory was opened */
public static final int OPEN = 0x00000020;
/** Event type: A file or subdirectory was moved from the monitored directory */
public static final int MOVED_FROM = 0x00000040;
/** Event type: A file or subdirectory was moved to the monitored directory */
public static final int MOVED_TO = 0x00000080;
/** Event type: A new file or subdirectory was created under the monitored directory */
public static final int CREATE = 0x00000100;
/** Event type: A file was deleted from the monitored directory */
public static final int DELETE = 0x00000200;
/** Event type: The monitored file or directory was deleted; monitoring effectively stops */
public static final int DELETE_SELF = 0x00000400;
/** Event type: The monitored file or directory was moved; monitoring continues */
public static final int MOVE_SELF = 0x00000800;
```

``FileObserver``只传递一个文件路径的构造函数就是监听所有的事件,只要有对应的事件触发,你就可以在``onEvent``方法中收到回调.

这里我监听了写入关闭和删除文件的事件,收到事件便提交了一个任务

```java
private class CountFileCallable implements Callable<Void> {

    private final File addFile;

    CountFileCallable(File addFile) {
        this.addFile = addFile;
    }

    @Override
    public Void call() {
        computeFileSize(addFile);
        return null;
    }
}

 private void computeFileSize(File addFile) {
        long totalSize = getTotalSize();
        boolean accepted = totalSize < mCacheSize;
        Log.i(TAG, "totalSize: " + totalSize + "-- mCacheSize: " + mCacheSize);
        if (!accepted) {
            //这里是保证不会让缓存超出大小
            if (addFile != null) {
                addFile.delete();
                Log.i(TAG, "delete file: " + addFile.getPath());
            }
        }
    }

```

这个任务主要是计算当前缓存文件夹的大小,如果超过缓存最大size,就删除最后添加的文件.整体的思路就是这样的,看起来没什么问题,这是改正后的版本.

我之前监听的是CREATE事件,即文件创建事件,此时如果提交任务,文件可能还没有下载完成,此时是不能计算出准确的缓存文件夹大小的,所以最后是超出了最大缓存大小.被测试提了bug.

## 总结

在使用不是很熟悉的``api``时,还是得先看一遍源码,至少得把主要方法看一遍,尤其是官方注释文档,其实文档对每个事件的定义解释得很清楚(当时没看,想当然了),看不懂尽量使用翻译工具,可以避免很多不必要的坑.

