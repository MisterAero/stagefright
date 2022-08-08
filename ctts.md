## Bugs

- Improper testing for 32bit multiplication overflow for heap allocation size

## Primitives

- OOB Applying of ntohl() per each 4 Bytes starting from the allocated heap chunk 

## Prerequisites
 
```cpp 
1. data_size == (numEntries + 1) * 8)
2. numEntries > SIZE_MAX / (sizeof(uint32_t) * 2 * numEntries) 
```

## Code
```cpp
     case FOURCC('c', 't', 't', 's'):
        {
            *offset += chunk_size;

            status_t err =
                mLastTrack->sampleTable->setCompositionTimeToSampleParams(
                        data_offset, chunk_data_size);

            if (err != OK) {
                return err;
            }

            break;
        }


....


status_t SampleTable::setCompositionTimeToSampleParams(
        off64_t data_offset, size_t data_size) {
    ALOGI("There are reordered frames present.");

    if (mCompositionTimeDeltaEntries != NULL || data_size < 8) {
        return ERROR_MALFORMED;
    }

    uint8_t header[8];
    if (mDataSource->readAt(
                data_offset, header, sizeof(header))
            < (ssize_t)sizeof(header)) {
        return ERROR_IO;
    }

    if (U32_AT(header) != 0) {
        // Expected version = 0, flags = 0.
        return ERROR_MALFORMED;
    }

    size_t numEntries = U32_AT(&header[4]);

    /* Must pass this check */
    if (data_size != (numEntries + 1) * 8) {
        return ERROR_MALFORMED;
    }

    mNumCompositionTimeDeltaEntries = numEntries;

      /* BUG: integer multiplication truncates to 32 bits .
    numEntries (Controlled by us) is of type size_t (uint32_t) ,
     condition won't detect overflow */
    uint64_t allocSize = numEntries * 2 * sizeof(uint32_t);
    if (allocSize > SIZE_MAX) {
        return ERROR_OUT_OF_RANGE;
    }

    /* BUG: Can result in a very small allocation */
    mCompositionTimeDeltaEntries = new uint32_t[2 * numEntries];

    /* Copies data into the heap from our data source
      the const 8 is sizeof(uint32_t) * 2 from the previous formula
      ssize_t is signed 32 bit so same value as what was allocated is about to be copied... */
    if (mDataSource->readAt(
                data_offset + 8, mCompositionTimeDeltaEntries, numEntries * 8)
            < (ssize_t)numEntries * 8) {
        delete[] mCompositionTimeDeltaEntries;
        mCompositionTimeDeltaEntries = NULL;

        return ERROR_IO;
    }

    /* This is where the heap overflow happens 
    it will execute ntohl() on each uint32_t as if enough memory would have been allocated */
    for (size_t i = 0; i < 2 * numEntries; ++i) {
        mCompositionTimeDeltaEntries[i] = ntohl(mCompositionTimeDeltaEntries[i]);
    }

    mCompositionDeltaLookup->setEntries(
            mCompositionTimeDeltaEntries, mNumCompositionTimeDeltaEntries);

    return OK;
}


/* Nothing special */
void SampleTable::CompositionDeltaLookup::setEntries(
        const uint32_t *deltaEntries, size_t numDeltaEntries) {
    Mutex::Autolock autolock(mLock);

    mDeltaEntries = deltaEntries;
    mNumDeltaEntries = numDeltaEntries;
    mCurrentDeltaEntry = 0;
    mCurrentEntrySampleIndex = 0;
}





```