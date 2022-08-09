## stbl
Primitives:
- Re-Allocate the mDataSource heap chunk
  - Can be used for heap shaping in such a way that we'll be able to overwrite in a high probability/deterministicly
    useful memory inside mDataSource such a vtable pointer of the **readAt()** function
    Following the allocation, the data will get parsed? (see (***) below)
  - one way to do so is to use it also for heap spraying thanks to the logical bug of un-needed repating caching.
   (Still need to prove that it lets us spray the same chunk an arbitrary number of times)

## trak
- Needed as a prerequisite for exploiting the tx3g vulnerability (see below)
```cpp
 
/* The sample table is a special case of a container box */
if (chunk_type == FOURCC('s', 't', 'b', 'l')) {
    ALOGV("sampleTable chunk is %" PRIu64 " bytes long.", chunk_size);

   
       /* The following condition must be true in order to cache the mDataSource and reallocate memory for the cache.
       flags() is a virtual function that returns the flags of the mDataSource. , it simply checks if at least one of the flags kWantsPrefetching or kIsCachingDataSource is set. 
       KWantPrefetching is set if mDataSource is of type MediaHTTP.
       kIsCachingDataSource is set if mDataSource is of type NuCachedSource2 - we are using this class to cache the data in the first place.
        */
    if (mDataSource->flags()
            & (DataSource::kWantsPrefetching
                | DataSource::kIsCachingDataSource)) {
        /* (Re)Allocates new "data source" based on the current one.
        This can be used for heap shaping purposes. we just need to make the spot it will be allocated at predictable.
        Then we'll be able to overwrite important fields in the dataSource with heap overflow */
        
        /* Note that mDataSource might already be cached from createFromURI(), see createFromURI.md */
        
        /* This line creates a new MPEG4DataSource with a "wrapped atasource reference" sp<DataSource> mSource that references (took ownership of) the original source object (std::move style?).
        I assume the old heap chunk is freed once it detects that it's refcount is decremented to 0.
        I base this assumption on the following:
        sp stands for "Strong pointer/reference" , kind of a smart pointer.
        see https://android.googlesource.com/platform/system/core/+/master/libutils/include/utils/RefBase.h ,
        Especially look at the comments at the beginning.
        TODO: Does transfering the ownership of the reference comes together with copying the source to the new heap chunk?.
        
        */
        sp<MPEG4DataSource> cachedSource =
            new MPEG4DataSource(mDataSource);
        
        /* See setCachedRange.md for information on its implementation , the cache size (allocated heap chunk size) is according to chunk_size of the 'stbl' atom (it clears the old cache before that), then it fills the private mCache field of cachedSource with the stbl box data
        Therfore - it re/allocates a (smaller?) new cache which gets filled with the "To be processes" data.
        (which can contain more nested stbl boxes whereas the last one can/should include actuall data*/
        
        if (cachedSource->setCachedRange(*offset, chunk_size) == OK) {
            /* if caching is successful, we can use the cachedSource for the rest of the parsing as the current mDataSource*/
            mDataSource = cachedSource; /* Important */
        }
    }

    /* Not so interesting, see code at the bottom of the file */
    mLastTrack->sampleTable = new SampleTable(mDataSource);

    /* Not so sure why they didn't put a return statement here...
    The chunk_type won't change at this point */
}


bool isTrack = false;
if (chunk_type == FOURCC('t', 'r', 'a', 'k')) {
    isTrack = true;

    Track *track = new Track;
    track->next = NULL;
    if (mLastTrack) {
        mLastTrack->next = track;
    } else {
        mFirstTrack = track;
    }
    mLastTrack = track; /* exactly what we need */

    track->meta = new MetaData;
    track->includes_expensive_metadata = false;
    track->skipTrack = false;
    track->timescale = 0;
    track->meta->setCString(kKeyMIMEType, "application/octet-stream");
}

/* recursively parse the cached range that we've just set */
off64_t stop_offset = *offset + chunk_size;
*offset = data_offset;
while (*offset < stop_offset) {
    status_t err = parseChunk(offset, depth + 1); /* recursivly parse each sub-atom */
    if (err != OK) {
        return err;
    }
}

if (*offset != stop_offset) {
    return ERROR_MALFORMED;
}
/* We'll reach here only after done with parsing the track container, so we don't realy care 
    what happens here besides the fact we don't want to crash. */
if (isTrack) {
    if (mLastTrack->skipTrack) {
        Track *cur = mFirstTrack;

        if (cur == mLastTrack) {
            delete cur;
            mFirstTrack = mLastTrack = NULL;
        } else {
            while (cur && cur->next != mLastTrack) {
                cur = cur->next;
            }
            cur->next = NULL;
            delete mLastTrack;
            mLastTrack = cur;
        }

        return OK;
    }

    /* note that if mLastTract is NULL at this point, verifyTrack will cause segfault due to
    NULL dereference. 
    TODO: Does this neccessarily mean that we need at least two track atoms? */
    status_t err = verifyTrack(mLastTrack);

    if (err != OK) {
        return err;
    }
} else if (chunk_type == FOURCC('m', 'o', 'o', 'v')) {
....


```cpp

class SampleTable : public RefBase {
public:
    SampleTable(const sp<DataSource> &source);

...


SampleTable::SampleTable(const sp<DataSource> &source)
    : mDataSource(source),
....
    mSampleIterator = new SampleIterator(this);
}

MPEG4DataSource::MPEG4DataSource(const sp<DataSource> &source)
    : mSource(source),
      mCachedOffset(0),
      mCachedSize(0),
      mCache(NULL) {
}


```