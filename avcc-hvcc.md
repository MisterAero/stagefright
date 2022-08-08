## avcC & hvcC

### Primitives
- Heap allocation & free primitive (for heap shaping/spraying) of size chunk_data_size
  (*) At any time,  only a single chunk of data can is allocated at most for this type of boxes/atoms.
  TODO: check if (*) is true for each one of them seperately or combined - 
  Since setData uses a KeyedVector for the metaData items (see api.md), it makes more sense that the memory is allocated per type of an atom (each of them PROBABELY has a unique key). Thus -  it is unlinkely that the same dynamic memory will be shared between atoms of different types. 

```cpp
/* Both of these functions calls SetData() (see api.md) which allocates memory dynamically and free the currently used chunk if allocated
   (similar to realloc() ) => they be used for heap shaping in such a way that we can control both the allocation and the freeing of the ("same" = same user pointer) memory ,+ controlling the requested size and the copied data */
   case FOURCC('a', 'v', 'c', 'C'):
        {
            *offset += chunk_size;

            /* Calls malloc under the hood */
            sp<ABuffer> buffer = new ABuffer(chunk_data_size);

            if (mDataSource->readAt(
                        data_offset, buffer->data(), chunk_data_size) < chunk_data_size) {
                return ERROR_IO;
            }

            mLastTrack->meta->setData(
                    kKeyAVCC, kTypeAVCC, buffer->data(), chunk_data_size);

            break;
        }


        case FOURCC('h', 'v', 'c', 'C'):
        {
            sp<ABuffer> buffer = new ABuffer(chunk_data_size);

            if (mDataSource->readAt(
                        data_offset, buffer->data(), chunk_data_size) < chunk_data_size) {
                return ERROR_IO;
            }

            mLastTrack->meta->setData(
                    kKeyHVCC, kTypeHVCC, buffer->data(), chunk_data_size);

            *offset += chunk_size;
            break;
        }

....
