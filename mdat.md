
## mdat

**Seems to be a big rabbit-hole, don't bother for now...**

   - Constraints:
     - mIsDrm flag is set (if the file is DRM protected)
  which is being set only in the following:
```cpp
  src/media/libstagefright/DRMExtractor.cpp:
  (DRMExtractor::DRMExtractor() which can be called only from 
  sp<MediaExtractor> MediaExtractor::Create(
        const sp<DataSource> &source, const char *mime) 
  This path will have to be further explored to understand how it works,
  It may be too much time consuming to explore it.
`      
  235      mOriginalExtractor = MediaExtractor::Create(source, mime);
  236:     mOriginalExtractor->setDrmFlag(true);
  237      mOriginalExtractor->getMetaData()->setInt32(kKeyIsDRM, 1); 

src/media/libstagefright/MediaExtractor.cpp:
  139         if (isDrm) {
  140:            ret->setDrmFlag(true);
  141         } else {

src/media/libstagefright/WVMExtractor.cpp:
  65              CHECK(mImpl != NULL);
  66:             setDrmFlag(true);
  67          } else {              
```
 
```cpp
        case FOURCC('m', 'd', 'a', 't'):
        {
            ALOGV("mdat chunk, drm: %d", mIsDrm);
            if (!mIsDrm) {
                *offset += chunk_size;
                break;
            }

            if (chunk_size < 8) {
                return ERROR_MALFORMED;
            }

            /* We want to reach here */
            return parseDrmSINF(offset, data_offset);
        }

Lets see what parseDrmSINF() does:

/* Haven't found a vulnerability in this function yet. 
But due to it's length, complexity and weird logic, it might contain some useful bugs.
Maybe it can be used to control the value data_offset (Arbitrary read)
parseDrmSINF() is responsible for digital right management validation? */
```cpp
status_t MPEG4Extractor::parseDrmSINF(
        off64_t * /* offset */, off64_t data_offset) {
    uint8_t updateIdTag;
    if (mDataSource->readAt(data_offset, &updateIdTag, 1) < 1) {
        return ERROR_IO;
    }
    data_offset ++;

    if (0x01/*OBJECT_DESCRIPTOR_UPDATE_ID_TAG*/ != updateIdTag) {
        return ERROR_MALFORMED;
    }

    uint8_t numOfBytes;
    
    /* (size is controlled by us, because mDataSource is also controlled by us) */
    int32_t size = readSize(data_offset, mDataSource, &numOfBytes);
    if (size < 0) {
        return ERROR_IO;
    }
    data_offset += numOfBytes;

    while(size >= 11 ) {
        uint8_t descriptorTag;
        if (mDataSource->readAt(data_offset, &descriptorTag, 1) < 1) {
            return ERROR_IO;
        }
        data_offset ++;

        if (0x11/*OBJECT_DESCRIPTOR_ID_TAG*/ != descriptorTag) {
            return ERROR_MALFORMED;
        }

        uint8_t buffer[8];
        //ObjectDescriptorID and ObjectDescriptor url flag
        if (mDataSource->readAt(data_offset, buffer, 2) < 2) {
            return ERROR_IO;
        }
        data_offset += 2;

        if ((buffer[1] >> 5) & 0x0001) { //url flag is set
            return ERROR_MALFORMED;
        }

        if (mDataSource->readAt(data_offset, buffer, 8) < 8) {
            return ERROR_IO;
        }
        data_offset += 8;

        if ((0x0F/*ES_ID_REF_TAG*/ != buffer[1])
                || ( 0x0A/*IPMP_DESCRIPTOR_POINTER_ID_TAG*/ != buffer[5])) {
            return ERROR_MALFORMED;
        }

        SINF *sinf = new SINF;
        sinf->trackID = U16_AT(&buffer[3]);
        sinf->IPMPDescriptorID = buffer[7];
        sinf->next = mFirstSINF;
        mFirstSINF = sinf;

        size -= (8 + 2 + 1); /* for the ??? */
    }

    if (size != 0) {
        return ERROR_MALFORMED;
    }

    if (mDataSource->readAt(data_offset, &updateIdTag, 1) < 1) {
        return ERROR_IO;
    }
    data_offset ++;

    if(0x05/*IPMP_DESCRIPTOR_UPDATE_ID_TAG*/ != updateIdTag) {
        return ERROR_MALFORMED;
    }

    /* Probabely a second size field of the following box.
    we control size*/
    size = readSize(data_offset, mDataSource, &numOfBytes);
    if (size < 0) {
        return ERROR_IO;
    }
    data_offset += numOfBytes;

    while (size > 0) {
        uint8_t tag;
        int32_t dataLen;
        if (mDataSource->readAt(data_offset, &tag, 1) < 1) {
            return ERROR_IO;
        }
        data_offset ++;

        if (0x0B/*IPMP_DESCRIPTOR_ID_TAG*/ == tag) {
            uint8_t id;
            dataLen = readSize(data_offset, mDataSource, &numOfBytes);
            if (dataLen < 0) {
                return ERROR_IO;
            } else if (dataLen < 4) {
                return ERROR_MALFORMED;
            }
            data_offset += numOfBytes;

            if (mDataSource->readAt(data_offset, &id, 1) < 1) {
                return ERROR_IO;
            }
            data_offset ++;

            SINF *sinf = mFirstSINF;
            while (sinf && (sinf->IPMPDescriptorID != id)) {
                sinf = sinf->next;
            }
            if (sinf == NULL) {
                return ERROR_MALFORMED;
            }
            sinf->len = dataLen - 3;

            /* Heap allocation of constant size with a controlled number of allocations (loop iterations)
              which doesn't get immediately freed afterwards => Useful for heap shaping */
            sinf->IPMPData = new (std::nothrow) char[sinf->len];
            if (sinf->IPMPData == NULL) {
                return ERROR_MALFORMED;
            }
            data_offset += 2;

            if (mDataSource->readAt(data_offset, sinf->IPMPData, sinf->len) < sinf->len) {
                return ERROR_IO;
            }
            data_offset += sinf->len;

            /* ???? */
            size -= (dataLen + numOfBytes + 1);
        }
    }

    if (size != 0) {
        return ERROR_MALFORMED;
    }

    return UNKNOWN_ERROR;  // Return a dummy error.
}


```