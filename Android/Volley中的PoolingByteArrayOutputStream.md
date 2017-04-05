### Volley中的PoolingByteArrayOutputStream
PoolingByteArrayOutputStream借助一个ByteArrayPool类来维护byte缓存池，因为频繁创建、清除byte数组可能引起严重的内存抖动。
下面是分析，精华都在注释里！

##### ByteArrayPool主要代码：

```java
public class ByteArrayPool {
    //全部byte数组的占用空间限制
    private final int mSizeLimit;
    
    //mBuffersByLastUse维护缓存byte数组的使用顺序，在压缩缓存池的时候将删掉最后的byte数组
    private List<byte[]> mBuffersByLastUse = new LinkedList<byte[]>();
    
    //mBuffersBySize维护缓存byte数组的大小顺序，PoolingByteArrayOutputStream按需提取byte数组
    private List<byte[]> mBuffersBySize = new ArrayList<byte[]>(64);

    public synchronized byte[] getBuf(int len) {
        for (int i = 0; i < mBuffersBySize.size(); i++) {
            byte[] buf = mBuffersBySize.get(i);
            if (buf.length >= len) {
                mCurrentSize -= buf.length;
                mBuffersBySize.remove(i);
                mBuffersByLastUse.remove(buf);
                return buf;
            }
        }
        //可以返回大于mSizeLimit的byte数组，后续不会进入mBuffers
        return new byte[len];
    }

    /**
     * Returns a buffer to the pool, throwing away old buffers if the pool would exceed its allotted
     * size.
     *
     * @param buf the buffer to return to the pool.
     */
    public synchronized void returnBuf(byte[] buf) {
        if (buf == null || buf.length > mSizeLimit) {
            return;
        }
        mBuffersByLastUse.add(buf);
        //二分查找
        int pos = Collections.binarySearch(mBuffersBySize, buf, BUF_COMPARATOR);
        if (pos < 0) {
            pos = -pos - 1;
        }
        mBuffersBySize.add(pos, buf);
        mCurrentSize += buf.length;
        trim();
    }

    /**
     * Removes buffers from the pool until it is under its size limit.
     */
    private synchronized void trim() {
        while (mCurrentSize > mSizeLimit) {
            byte[] buf = mBuffersByLastUse.remove(0);
            mBuffersBySize.remove(buf);
            mCurrentSize -= buf.length;
        }
    }

```

   
##### PoolingByteArrayOutputStream主要代码：

```java
public class PoolingByteArrayOutputStream extends ByteArrayOutputStream {

   public PoolingByteArrayOutputStream(ByteArrayPool pool, int size) {
        mPool = pool;
        buf = mPool.getBuf(Math.max(size, DEFAULT_SIZE));
    }

	private void expand(int i) {
        /* Can the buffer handle @i more bytes, if not expand it */
        if (count + i <= buf.length) {
            return;
        }
        byte[] newbuf = mPool.getBuf((count + i) * 2);
        System.arraycopy(buf, 0, newbuf, 0, count);
        mPool.returnBuf(buf);
        buf = newbuf;
    }

    @Override
    public synchronized void write(byte[] buffer, int offset, int len) {
        //取代父类ByteArrayOutputStream的grow方法实现扩大缓存buf
        //（没有像grow方法那样针对OutOfMemory做处理，虽然一般不会出现）
        expand(len);
        super.write(buffer, offset, len);
    }

    @Override
    public synchronized void write(int oneByte) {
        expand(1);
        super.write(oneByte);
    }
}
```