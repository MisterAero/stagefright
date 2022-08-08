## setCachedRange

**Still not sure what it can give us other than clearing the cache**
```cpp
status_t MPEG4DataSource::setCachedRange(off64_t offset, size_t size) {
    Mutex::Autolock autoLock(mLock);

    clearCache();

    mCache = (uint8_t *)malloc(size);

    if (mCache == NULL) {
        return -ENOMEM;
    }

    mCachedOffset = offset;
    mCachedSize = size;

    /* Vulnerability? heap overflow controlled by us? */
    ssize_t err = mSource->readAt(mCachedOffset, mCache, mCachedSize);

    if (err < (ssize_t)size) {
        clearCache();

        return ERROR_IO;
    }

    return OK;
}

...

void MPEG4DataSource::clearCache() {
    if (mCache) {
        free(mCache);
        mCache = NULL;
    }

    mCachedOffset = 0;
    mCachedSize = 0;
}

```