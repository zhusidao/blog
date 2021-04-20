CountDownLatch源码分析如下:
核心方法await()、countDown()
`>
/**
 *
 */
public void countDown() {
        sync.releaseShared(1);
    }
`

