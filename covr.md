
3. COVR type
   - Two heap overflow vulnerabilities (integer overflow & underflow)
   - Gives us a heap shaping primitive (temporary memory leak)
```cpp
Integer overflow vulnerability that results in buffer overflow in MPEG4Source::parseChunk()
        case FOURCC('c', 'o', 'v', 'r'):
        {
            *offset += chunk_size;

            if (mFileMetaData != NULL) {
                ALOGV("chunk_data_size = %lld and data_offset = %lld",
                        chunk_data_size, data_offset);

                /* - (A) VULNERABILITY: integer overflow 
                 if chunk_data_size >= MAX_SIZE, then the next statement
                 wiil allocate a much smaller buffer than intended.
                   - No immidiate free afterwards => can be used for heap shaping/spraying */
                   /* in case chunk_data_size == SIZE_MAX, the overflow is guranteed to happen,
                   because ABuffer(size_t capacity) initialize mData with  mData(malloc(capacity)).
                   
                   Crafting an atom with chunk_data_size = MAX_SIZE is simple because it's controlled by us
                    (we don't even need to use a special atom with a 64bit size, 
                    32bit is enough for the 32bit process case)*/
                sp<ABuffer> buffer = new ABuffer(chunk_data_size + 1);
                if (mDataSource->readAt(
                    data_offset, buffer->data(), chunk_data_size) != (ssize_t)chunk_data_size) {
                    return ERROR_IO;
                }
                
                const int kSkipBytesOfDataBox = 16;
                /* (B) VULNERABILITY: Integer underflow
                 There's no check for chunk_data_size - kSkipBytesOfDataBox < 0 
                 This results in passing into MetaData::setData a very large value.
                 The reason for this is that setData gets size of type size_t,
                 kSkipBytesOfDataBox is of type int, and fixed to value 16.
                 chunk_data_size is of type off64_t (long int 64bit system) and controlled by us.
                  The result of the subtraction is of type off64_t.
                 Which eventually gets converted into size_t (long unsigned int)
                 => classic signed/unsigned mismatch bug. 
                 
                 Check out api.md to see that setData will eventually allocate a very large space on the heap with such an underflow, and then write to it much less data.
                 */
                mFileMetaData->setData(
                    kKeyAlbumArt, MetaData::TYPE_NONE,
                    buffer->data() + kSkipBytesOfDataBox, chunk_data_size - kSkipBytesOfDataBox);
            }

            break;
        }
```