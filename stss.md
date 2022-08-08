# stss

## Bugs
- Improper 32bit multiplication calculation of heap allocation size. 

## Primitives
- Modifiying OOB memory on the heap of size ((uint64_t)nNumSyncSamples * sizeof(uint32_t)) -  (nNumSyncSamples * sizeof(uint32_t))  in units of 4Bytes as following:
  memory[i] = ntohl(memory[i]) - 1;

## Exploitability
- Doesn't seem to be so beneficial unless we wish to bypass some heap mitigiations caused by checks on chunks meta-data

```cpp
        case FOURCC('s', 't', 's', 's'):
        {
            *offset += chunk_size;
            // Ignore stss block for audio even if its present
            // All audio sample are sync samples itself,
            // self decodeable and playable.
            // Parsing this block for audio restricts audio seek to few entries
            // available in this block, sometimes 0, which is undesired.
            const char *mime;
            CHECK(mLastTrack->meta->findCString(kKeyMIMEType, &mime));
            if (strncasecmp("audio/", mime, 6)) {
                status_t err =
                    mLastTrack->sampleTable->setSyncSampleParams(
                            data_offset, chunk_data_size);

                if (err != OK) {
                    return err;
                }
            }

            break;
        }



...


/**
 * Note that the returned pointer becomes invalid when additional metadata is set.
 */
bool MetaData::findCString(uint32_t key, const char **value) {
    uint32_t type;
    const void *data;
    size_t size;
    if (!findData(key, &type, &data, &size) || type != TYPE_C_STRING) {
        return false;
    }

    *value = (const char *)data;

    return true;
}


...



status_t SampleTable::setSyncSampleParams(off64_t data_offset, size_t data_size) {
    if (mSyncSampleOffset >= 0 || data_size < 8) {
        return ERROR_MALFORMED;
    }

    mSyncSampleOffset = data_offset;

    uint8_t header[8];
    if (mDataSource->readAt(
                data_offset, header, sizeof(header)) < (ssize_t)sizeof(header)) {
        return ERROR_IO;
    }

    if (U32_AT(header) != 0) {
        // Expected version = 0, flags = 0.
        return ERROR_MALFORMED;
    }

    mNumSyncSamples = U32_AT(&header[4]);

    if (mNumSyncSamples < 2) {
        ALOGV("Table of sync samples is empty or has only a single entry!");
    }

    /* BUG: (Not a vulnerability)
     * Integer overflow due to truncation of multiplication result.
     nNumSyncSamples is of type uint32_t , it is being multiplied by a constant.
     => No integer promotion is going to happen
     Therefore the result of the multiplication is truncated to 32 bits.
     and then zero-extended to 64 bits.
     So the following if statement will never detect an overflow.
     This assumes SIZE_MAX is 2^32 -1 (4,294,967,295) - Which is true for a 32 bit machine.
    Otherwise (64bit)- this check is even more useless. and the vulnuarability still holds true
    */
    uint64_t allocSize = mNumSyncSamples * sizeof(uint32_t);
    if (allocSize > SIZE_MAX) {
        return ERROR_OUT_OF_RANGE;
    }

    /* In case of an overflow, new will allocate a very small amount of memory.
     * because it does the above calculation to figure out how much memory to allocate.
     */
    mSyncSamples = new uint32_t[mNumSyncSamples];

    /* size_t is of type uint64_t, calculation is pretty much same as before 
    No overflow will occur here...
    */
    size_t size = mNumSyncSamples * sizeof(uint32_t);
    if (mDataSource->readAt(mSyncSampleOffset + 8, mSyncSamples, size)
            != (ssize_t)size) {
        return ERROR_IO;
    }

    /* but, this will modify the memory after the newly allocated space
    ( It might be Not so useful to just change the byte order) */
    for (size_t i = 0; i < mNumSyncSamples; ++i) {
        mSyncSamples[i] = ntohl(mSyncSamples[i]) - 1;
    }

    return OK;
}


...

uint32_t U32_AT(const uint8_t *ptr) {
    return ptr[0] << 24 | ptr[1] << 16 | ptr[2] << 8 | ptr[3];
}


```