
1. pssh
   Primitives:
   - heap allocation primitive (for heap shaping/spraying)
        - controlled size of pssh.datalen (We control the size of the pssh.data)
     How it is useful for us:
        Let us fill any hole in the heap (fragmented memory) with allocations in the size group that 
        we wish that future allocations of it will be highly likely to be contiguous on the heap (for heap shaping purposes).

```cpp
        case FOURCC('p', 's', 's', 'h'):
        {
            *offset += chunk_size;

            PsshInfo pssh;

            if (mDataSource->readAt(data_offset + 4, &pssh.uuid, 16) < 16) {
                return ERROR_IO;
            }

            uint32_t psshdatalen = 0;
            if (mDataSource->readAt(data_offset + 20, &psshdatalen, 4) < 4) {
                return ERROR_IO;
            }
            pssh.datalen = ntohl(psshdatalen);
            ALOGV("pssh data size: %d", pssh.datalen);
            /*NOTE that pssh.datalen is of type uint32_t, 
            chunk_size is of type uint64_t, so the former will be converted to a uint64_t for the comparision by zero-extending it (integer promotion rules) so apperantly no bug in here
            It simply checks validates that pssh.datalen doesn't exceed size of the current box */
            if (pssh.datalen + 20 > chunk_size) {
                // pssh data length exceeds size of containing box
                return ERROR_MALFORMED;
            }
            
            /* No immidiate free afterwards, so this can be used for Heap spary/shaping */
            pssh.data = new (std::nothrow) uint8_t[pssh.datalen];
            if (pssh.data == NULL) {
                return ERROR_MALFORMED;
            }
            ALOGV("allocated pssh @ %p", pssh.data);
      
            /*Note that the value requested is reflecting pssh.datalen (value preserving conversion)
            so there's no possibility for overflow in the following readAt() call */ 
            ssize_t requested = (ssize_t) pssh.datalen;
            
            /* We control the data that will be copied onto the recently allocated heap space */
            if (mDataSource->readAt(data_offset + 24, pssh.data, requested) < requested) {
                return ERROR_IO;
            }
            mPssh.push_back(pssh);

            break;
        }
```