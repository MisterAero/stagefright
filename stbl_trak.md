## stbl
Primitives:
- Re-Allocate the mDataSource heap chunk
  - Can be used for heap shaping in such a way that we'll be able to overwrite in a high probability/deterministicly
    useful memory inside mDataSource such a vtable pointer of the **readAt()** function
    Following the allocation, the data will get parsed? (see (***) below)

## trak
- Needed as a prerequisite for exploiting the tx3g vulnerability (see below)
```cpp
...
        case FOURCC('s', 't', 'b', 'l'):
....
        {
            if (chunk_type == FOURCC('s', 't', 'b', 'l')) {
                ALOGV("sampleTable chunk is %" PRIu64 " bytes long.", chunk_size);
                /* The sample table is a special case. It is the only chunk
                 * that can be of variable length. We need to read the entire
                 * chunk to determine the size of the table.
                 */
                if (mDataSource->flags()
                        & (DataSource::kWantsPrefetching
                            | DataSource::kIsCachingDataSource)) {
                    /* Is this useful? */
                    /* (Re)Allocates new "data source" based on the current one.
                   This can be used for heap shaping purposes. we just need
                   to make the spot it will be allocated at predictable.
                   Then we'll be able to overwrite important fields in the dataSource with heap overflow */
                    sp<MPEG4DataSource> cachedSource =
                        new MPEG4DataSource(mDataSource);
                    
                    /* HERE */
                    if (cachedSource->setCachedRange(*offset, chunk_size) == OK) {
                        mDataSource = cachedSource;
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


    switch(chunk_type) {
        case FOURCC('m', 'o', 'o', 'v'):
        case FOURCC('t', 'r', 'a', 'k'):
        case FOURCC('m', 'd', 'i', 'a'):
        case FOURCC('m', 'i', 'n', 'f'):
        case FOURCC('d', 'i', 'n', 'f'):
        case FOURCC('s', 't', 'b', 'l'):
        case FOURCC('m', 'v', 'e', 'x'):
        case FOURCC('m', 'o', 'o', 'f'):
        case FOURCC('t', 'r', 'a', 'f'):
        case FOURCC('m', 'f', 'r', 'a'):
        case FOURCC('u', 'd', 't', 'a'):
        case FOURCC('i', 'l', 's', 't'):
        case FOURCC('s', 'i', 'n', 'f'):
        case FOURCC('s', 'c', 'h', 'i'):
        case FOURCC('e', 'd', 't', 's'):
        {
            if (chunk_type == FOURCC('s', 't', 'b', 'l')) {
                ALOGV("sampleTable chunk is %" PRIu64 " bytes long.", chunk_size);
                /* TODO: understand the meaning of cachedSource + decide when we 
                wish to use such a chunk_type (and how many times)
                 */
                if (mDataSource->flags()
                        & (DataSource::kWantsPrefetching
                            | DataSource::kIsCachingDataSource)) {
                    sp<MPEG4DataSource> cachedSource =
                        new MPEG4DataSource(mDataSource);

                    if (cachedSource->setCachedRange(*offset, chunk_size) == OK) {
                        mDataSource = cachedSource;
                    }
                }

                mLastTrack->sampleTable = new SampleTable(mDataSource);
                /* Note that we don't return after handling this case !*/
            }


 
```


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