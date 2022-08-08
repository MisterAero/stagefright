## Prerequisites
- (data_size < 8 + mNumSampleToChunkOffsets * 12)
  which is achieved by forcing **mNumSampleToChunkOffsets > (UINT_MAX - 8) / 12**

## Primitives
- OOB read from heap **INTO** another chunk on the heap which stores the sample table of the currently cached atom (OOB write) at specific offset (mSampleToChunkEntries field)

## Exploitability
**Very low, doesn't seem to be benificial at the moment**
```cpp
  case FOURCC('s', 't', 't', 's'):
        {
            *offset += chunk_size;

            status_t err =
                mLastTrack->sampleTable->setTimeToSampleParams(
                        data_offset, chunk_data_size);

            if (err != OK) {
                return err;
            }

            break;
        }

....

status_t SampleTable::setSampleToChunkParams(
        off64_t data_offset, size_t data_size) {
    if (mSampleToChunkOffset >= 0) {
        return ERROR_MALFORMED;
    }

    mSampleToChunkOffset = data_offset;

    if (data_size < 8) {
        return ERROR_MALFORMED;
    }

    uint8_t header[8];
    if (mDataSource->readAt(
                data_offset, header, sizeof(header)) < (ssize_t)sizeof(header)) {
        return ERROR_IO;
    }

    if (U32_AT(header) != 0) {
        // Expected version = 0, flags = 0.
        return ERROR_MALFORMED;
    }

    mNumSampleToChunkOffsets = U32_AT(&header[4]);

    /* BUG: No check for integer overflow due to multiplication,
    Both variables are uint32_t (size_t is 
    mNumsampleToChunkOffsets is of type uint32_t and data_size is of type size_t which is long unsigned int which is of the same size probabely,
    So the right hand side can turn out to be a very small positive value.
    Which will enable us to read from after the data ends and with that we will overwrite (overflow starting from) the current sampleTable at the offset of the private member array - mSampleToChunkEntries.
    */ 
    if (data_size < 8 + mNumSampleToChunkOffsets * 12) {
        return ERROR_MALFORMED;
    }

    mSampleToChunkEntries =
        new SampleToChunkEntry[mNumSampleToChunkOffsets];

    for (uint32_t i = 0; i < mNumSampleToChunkOffsets; ++i) {
        uint8_t buffer[12];
        if (mDataSource->readAt(
                    mSampleToChunkOffset + 8 + i * 12, buffer, sizeof(buffer))
                != (ssize_t)sizeof(buffer)) {
            return ERROR_IO;
        }

        CHECK(U32_AT(buffer) >= 1);  // chunk index is 1 based in the spec.

        // We want the chunk index to be 0-based.
        /* This is where heap overflow can happen , kind of a a weird info leak which might not be of any value to us*/
        mSampleToChunkEntries[i].startChunk = U32_AT(buffer) - 1;
        mSampleToChunkEntries[i].samplesPerChunk = U32_AT(&buffer[4]);
        mSampleToChunkEntries[i].chunkDesc = U32_AT(&buffer[8]);
    }

    return OK;
}


```