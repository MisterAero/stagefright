```cpp
parseChunk()
**This function is our "Entry point" for the vulnurability reserach.**
// The MPEG4Source object is created at main() in stagefright.cpp
//  sp<MediaExtractor> extractor = MediaExtractor::Create(dataSource);

/*offset is the offset in the mpeg4 file */
status_t MPEG4Source::parseChunk(off64_t *offset) {
    uint32_t hdr[2]; /*header,normally contains  the chunk data-size & type */
    /* read the first 8 bytes of the box */
    if (mDataSource->readAt(*offset, hdr, 8) < 8) {
        return ERROR_IO;
    }

    /* we control the header (comes from mDataSource), and thus- chunk_size and chunk_type */
    uint64_t chunk_size = ntohl(hdr[0]); 
    uint32_t chunk_type = ntohl(hdr[1]);
    off64_t data_offset = *offset + 8; /*data usually starts immidiately after the header */


    if (chunk_size == 1) {
        /* speical case, the real chunk_size field is of type uint_64t (and not uint_32t) and is
         stored 8 bytes after the typical header */
        if (mDataSource->readAt(*offset + 8, &chunk_size, 8) < 8) {
            return ERROR_IO;
        }
        /* convert from network to host ordering */
        chunk_size = ntoh64(chunk_size); 
        data_offset += 8; 

        if (chunk_size < 16) {
            /* The smallest valid chunk is 16 bytes long in this case.*/
            return ERROR_MALFORMED;
        }
    } else if (chunk_size == 0) {
        
        if (depth == 0) {
            /* ??? IF this is the first-level box, it is going to serve as the main container for
             the whole file, therefore - get the total size and subtract current offset from it*/
            off64_t sourceSize;
            if (mDataSource->getSize(&sourceSize) == OK) {
                /* VULNERABILITY : Possible integer underflow. we control both operands */
                chunk_size = (sourceSize - *offset);
            } else {
                ALOGE("atom size is 0, and data source has no size");
                return ERROR_MALFORMED;
            }
        } else {
            // not allowed for non-toplevel atoms, skip it
            *offset += 4; 
            return OK;
        }
    } else if (chunk_size < 8) {
        // The smallest valid chunk is 8 bytes long.
        ALOGE("invalid chunk size: %" PRIu64, chunk_size);
        return ERROR_MALFORMED;
    }

    char chunk[5];
    MakeFourCCString(chunk_type, chunk);
    ALOGV("chunk: %s @ %lld, %d", chunk, *offset, depth);

....

    PathAdder autoAdder(&mPath, chunk_type); 

    /* TODO: VULNERABILITY ? : Possible integer underflow???
       We control chunk_size, but I'm not sure if there is any benefit for faking it (declraring a chunk_size which is much larger then the length of supplied data + header)
    /* this variable is used to store the size of only the data in for current box
      Note that chunk_data_size is used in some of the switch cases (perhaps those who deals with the data?*/
    off64_t chunk_data_size = *offset + chunk_size - data_offset;

    /* TODO: Do we have to get into this condition on the first box ? (we want to put an
    'ftyp' box first in the file so that the file will be parsed as an mpeg4 file).
       (see code below for underMetaDataPath() which looks for a specific structure of file*/
    if (chunk_type != FOURCC('c', 'p', 'r', 't')
            && chunk_type != FOURCC('c', 'o', 'v', 'r')
            && mPath.size() == 5 && underMetaDataPath(mPath)) {

        off64_t stop_offset = *offset + chunk_size; /* to know where to stop parsing current atom */
        *offset = data_offset;
        while (*offset < stop_offset) {
            /* Important: This line (and others like it) tells us that the parser works/can work in a recursive (depth-first?) fashion.
            So this enables us to combine different chunk types into a single file. */
            status_t err = parseChunk(offset, depth + 1); /* each case will advance the offset var */
            if (err != OK) {
                return err;
            }
        }

        if (*offset != stop_offset) {
            return ERROR_MALFORMED;
        }

        return OK;
    }


    switch(chunk_type) {

    ..............
    ..............

```


```cpp
static bool underMetaDataPath(const Vector<uint32_t> &path) {
    return path.size() >= 5
        && path[0] == FOURCC('m', 'o', 'o', 'v')
        && path[1] == FOURCC('u', 'd', 't', 'a')
        && path[2] == FOURCC('m', 'e', 't', 'a')
        && path[3] == FOURCC('i', 'l', 's', 't');
}
```