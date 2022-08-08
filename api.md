```cpp

bool MetaData::setCString(uint32_t key, const char *value) {
    /* calls bool MetaData::setData(uint32_t key, uint32_t type, const void *data, size_t size) */
    return setData(key, TYPE_C_STRING, value, strlen(value) + 1);
}


bool MetaData::setData(
        uint32_t key, uint32_t type, const void *data, size_t size) {
    bool overwrote_existing = true;

    /* each unique atom type (FOURCC) has a unique key
     mItems is defined as   KeyedVector<uint32_t, typed_data> mItems;
    (https://android.googlesource.com/platform/frameworks/native/+/jb-dev/include/utils/KeyedVector.h)
    each atom type will have a unique key  (kKeyAVCC, kKeyHVCC, etc) whose matching value is an item
    that describes the previously used atom of that type.
    */
    ssize_t i = mItems.indexOfKey(key);
    if (i < 0) {
        typed_data item;
        i = mItems.add(key, item);

        overwrote_existing = false;
    }

    /* this call will "reset" the chosen cell in the keyedVector */
    typed_data &item = mItems.editValueAt(i);

    /* fill the typed_data struct with the new data while allocating memory for it if needed */
    item.setData(type, data, size);

    return overwrote_existing;
}

the inner call item.setData(type, data, size)  calls:

void MetaData::typed_data::setData(
        uint32_t type, const void *data, size_t size) {
    clear(); /* test and clear the currently memory allocated to the Data */

    mType = type; /* set type of the data */
    allocateStorage(size);  /* allocate memory for the data */
    memcpy(storage(), data, size);
}

void MetaData::typed_data::clear() {
    freeStorage(); /* frees the storage allocated by allocateStorage() */
    mType = 0; /* reset the type */
}

void MetaData::typed_data::freeStorage() {
    /* if the dynamic external data storage is allocated and it's pointer is valid, free it */
    if (!usesReservoir()) {
        if (u.ext_data) {
            free(u.ext_data); /* free the memory which was previously allocated for the data */
            u.ext_data = NULL; /* reset the pointer to eliminate dangling pointers or UAF bug */
        }
    }

    mSize = 0; /* reset the size */
}

/* mSize is the data size registered in the metadata. 
  sizeof(u.reservoir) is the size of the reservoir field type (float).
   This is a simple way to check if the u.reservoir field is currently USED in the union,
   meaning that a dynamic storage is currently NOT allocated for the data. */ 
bool usesReservoir() const {
    return mSize <= sizeof(u.reservoir);
}
...

Wheras u is simply a private member field of the inner struct MetaData::typed_data

private:
    uint32_t mType;
    size_t mSize;

    union {
        void *ext_data;
        float reservoir;
    } u;
...

void MetaData::typed_data::allocateStorage(size_t size) {
    mSize = size; /* in case size is >= sizeof(u.reservoir), this will make gurantee usesReservoir() returns false */

    /* if msize is not big enough (invalid) or somehow a race condition occured and now the union currently set to use the reservoir field, don't allocate dynamic storage */
    if (usesReservoir()) {
        return;
    }

    /* allocate memory for the data and set the pointer field to the allocated memory */
    u.ext_data = malloc(mSize);
}

...
void *storage() {
    return usesReservoir() ? &u.reservoir : u.ext_data;
}

...


```



```cpp
//TODO: understand how it works exactly
ssize_t MPEG4DataSource::readAt(off64_t offset, void *data, size_t size) {
    Mutex::Autolock autoLock(mLock); //protects data from being accessed by multiple threads at the same time

    if (offset >= mCachedOffset
            && offset + size <= mCachedOffset + mCachedSize) {
       /* we can serve the request from the cache */
        memcpy(data, &mCache[offset - mCachedOffset], size);
        return size;
    }

    /* if the requested data is not in the cache, we need to read it from the source??? */
    return mSource->readAt(offset, data, size);
}
```

```cpp
ABuffer::ABuffer(size_t capacity)
    : mMediaBufferBase(NULL),
      mData(malloc(capacity)),
      mCapacity(capacity),
      mRangeOffset(0),
      mRangeLength(capacity),
      mInt32Data(0),
      mOwnsData(true) {
}
```