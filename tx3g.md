
1.
Might be the strongest heap overflow vulnerability in this file.
Constraints: 
- Need a legitimate tx3g atom that will cause reaching to (1)
- mLastTrack != NULL && mLastTrack->meta != NULL
    - can be solved by adding a **'trak'** atom somewhere before the tx3g atom (see stbl_trak.md)
- need to reach such a state on the heap where an important chunk will be placed right after where the tx3g chunk will be allocated (the file atoms parsing process causes a series of allocation and/or free operations)
    - can be achieved by inserting atoms that will cause a heap shaping state of [Some Chunks] [Hole that matches for the tx3g small buffer ] [Important struct that controls the flow]. for this we have the pssh (filling holes) and hvcC/avcC (creating holes) primitives.

```cpp
/* tx3g is a format used to store subtitles in mp4 files */
/* The overflow can occur in one or more places in this case, marked by (1), (2)
­this code collects all timed­-text type chunks and combines/appends/ them in a FIFO manner into a one single long buffer.
*/
case FOURCC('t', 'x', '3', 'g'):
        {
            uint32_t type;
            const void *data;
            size_t size = 0; /* initialized to zero. */

            /* parse type ,data and size from the atom 
             of the metadata of the last track? (for this type) 
             find previous timed‐text data
             NOTE: mLastTrack must not be NULL!*/
            if (!mLastTrack->meta->findData(
                    kKeyTextFormatData, &type, &data, &size)) {
                /*no previous timed‐text data found*/
                size = 0;
            }

            /* at this point, **data** and **size** might be NULL and 0 respectively.
            /* dynamically allocate a buffer for the data plus the size of the chunk (atom?), enough memory for both the old buffer and the new buffer*/
            /*Vulnurebility: 
             - no check for integer overflow => Heap overflow

            Constraints:
               A. We want size > chunk_size (for heap overflow), and that 
               B. size(size_t = unsigned int) + chunk_size(uint64_t) > SIZE_MAX/sizeof(uint8_t) = SIZE_MAX (Upper limit on size_t).
                we control both size and chunk_size 
                size is taken from the metaData of the last track (current?), so we can control it.
                chunk_size is derived from the atom header
               C. Need for Heap shaping, so that we can overwrite(overflow) something meaningful in a deterministic way (such as a vtable funcion pointer / libc hook / shellcode) */  
            uint8_t *buffer = new (std::nothrow) uint8_t[size + chunk_size];
            if (buffer == NULL) {
                return ERROR_MALFORMED;
            }

            /*if there was any previous timed‐text data, copy it to the beginning of the buffer */
            if (size > 0) {
                memcpy(buffer, data, size); /* (1) This is the first place where heap overflow can occur, 
                                                and it's enough to do damage */
            }

            /* (2) we control mDataSource, so our malicaious payload at offset "offset" in mDataSource of len **chunk_size**
             will be written starting from &buffer[size] , 
            TODO:
            - can we also control offset as much as we want? 
            Since readAt uses caching we won't deep dive into it's details yet, but check out the appendix "api.md" */
            /* append current timed‐text data to the end of the buffer, this will read from our file at a certain offset
            to the heap buffer*/
            /* behaviour depends on readAt() implementation, unlike the memcpy from (1) */
            /
            if ((size_t)(mDataSource->readAt(*offset, buffer + size, chunk_size))
                    < chunk_size) {
                /* Unexpected error in readAt()*/
                delete[] buffer; 
                buffer = NULL;

                /* advance read pointer so we don't end up reading this part again */
                *offset += chunk_size;
                return ERROR_IO; /*signal io error*/
            }


            /* see api.md for setData implementation*/
            /* Sets the current/last timed‐text data to the new buffer content, this will replace/override the previous one. Therefore - It doesn't write to buffer*/
            mLastTrack->meta->setData(
                    kKeyTextFormatData, 0, buffer, size + chunk_size);

            delete[] buffer; /* NOTE: the buffer is always freed after it served it purpose, 
                                but overflow already happend so we don't mind */

            *offset += chunk_size; /*wether we succeeded or failed, we need to advance the file offset*/
            break;
        }
