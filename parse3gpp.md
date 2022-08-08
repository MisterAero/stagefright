## parse3gpp 

### Primitives:
- Info leak starting from current heap chunk (at offset of 6) which
  will be written to a new heap allocated chunk 
- Such large allocation might also be used for heap shaping
  
```cpp
        case FOURCC('t', 'i', 't', 'l'):
        case FOURCC('p', 'e', 'r', 'f'):
        case FOURCC('a', 'u', 't', 'h'):
        case FOURCC('g', 'n', 'r', 'e'):
        case FOURCC('a', 'l', 'b', 'm'):
        case FOURCC('y', 'r', 'r', 'c'):
        {
            *offset += chunk_size;

            /* Lets inspect this call */
            status_t err = parse3GPPMetaData(data_offset, chunk_data_size, depth);

            if (err != OK) {
                return err;
            }

            break;
        }



status_t MPEG4Extractor::parse3GPPMetaData(off64_t offset, size_t size, int depth) {
    /*This will prevent underflow later on */
    if (size < 4) {
        return ERROR_MALFORMED;
    }
   
    /* Nothing suspicious over here.... */
    uint8_t *buffer = new (std::nothrow) uint8_t[size];
    if (buffer == NULL) {
        return ERROR_MALFORMED;
    }
    if (mDataSource->readAt(
                offset, buffer, size) != (ssize_t)size) {
        delete[] buffer;
        buffer = NULL;

        return ERROR_IO;
    }

    uint32_t metadataKey = 0;
    switch (mPath[depth]) {
        case FOURCC('t', 'i', 't', 'l'):
        {
            metadataKey = kKeyTitle;
            break;
        }
        case FOURCC('p', 'e', 'r', 'f'):
        {
            metadataKey = kKeyArtist;
            break;
        }
        case FOURCC('a', 'u', 't', 'h'):
        {
            metadataKey = kKeyWriter;
            break;
        }
        case FOURCC('g', 'n', 'r', 'e'):
        {
            metadataKey = kKeyGenre;
            break;
        }
        case FOURCC('a', 'l', 'b', 'm'):
        {
            if (buffer[size - 1] != '\0') {
              char tmp[4];
              /* ???? No idea why they built it this way...*/
              sprintf(tmp, "%u", buffer[size - 1]);
              
              /* We'll need to inspect this function call 
                 inspect it's code in api.md and see for yourself that it does the following:
                 num_of_bytes_to_copy = strlen(value) + 1 , so we need to watch for
                 errors to null terminate strings in the code... */
              mFileMetaData->setCString(kKeyCDTrackNumber, tmp);
            }

            metadataKey = kKeyAlbum;
            break;
        }
        case FOURCC('y', 'r', 'r', 'c'):
        {
            char tmp[5];
            uint16_t year = U16_AT(&buffer[4]);

            if (year < 10000) {
                sprintf(tmp, "%u", year);

                mFileMetaData->setCString(kKeyYear, tmp);
            }
            break;
        }

        default:
            break;
    }

    if (metadataKey > 0) {
        bool isUTF8 = true; // Common case
        char16_t *framedata = NULL;
        int len16 = 0; // Number of UTF-16 characters

        /* This following calculation seems to be complicated, lets inspect it */
        // smallest possible valid UTF-16 string w BOM: 0xfe 0xff 0x00 0x00
        
        /*Vulnerability: integer underflow + off by one error*/
        /* BOM stands for Byte Order Mark, it is a way to identify the endianness of the machine that is running the program. ??? */ 
        /* For some reason, they assume that first 6 bytes were already handled, now it expects at least 4 more bytes 
        Note that due to the fact size type is **size_t** (unsigned) , the following condition CAN RESULT in an unsigned integer underflow when size is 4 or 5 (the first if condition mentioned above only check if 0<=size<4 ).
         this will affect len16, which will cause swapping between every two bytes of the upcoming len16 UTF-16 chars (REVERSIBLE, but need to be carful not do abort the program)...
         Later we'll see that this can result in a large info leak */
        /* They forgot to check if 4 < size < 6 ... */ 
        if (size - 6 >= 4) {
            /* calculate the number of UTF-16 characters in the string(not including the terminating null character) */
            /* type(len16) is int (signed) , type(size) is size_t (unsigned int = 4Bytes for a 32bit process),
            /* No comparision is done here, so rhs will be calculated first and then converted to the lhs type (casted),
            but they are of same size so it will just be interpeted as signed
            for size = 5 (or 4) , size - 6 will be a very large unsigned int, when divided by 2 and decremented - it will
            fit within the positive range of signed int (value conserved) => we will end up with a large positive signed int value*/
            len16 = ((size - 6) / 2) - 1; // don't include 0x0000 terminator
            framedata = (char16_t *)(buffer + 6);
            if (0xfffe == *framedata) {
                /*We want to get here so that isUTF8 keeps the value true */
                // endianness marker (BOM) doesn't match host endianness
                for (int i = 0; i < len16; i++) {
                    framedata[i] = bswap_16(framedata[i]);
                }
                // BOM is now swapped to 0xfeff, we will execute next block too
            }

            if (0xfeff == *framedata) {
                // Remove the BOM
                framedata++;
                len16--;
                isUTF8 = false;
            }
            // else normal non-zero-length UTF-8 string
            // we can't handle UTF-16 without BOM as there is no other
            // indication of encoding.
        }

        if (isUTF8) {
            /* BUG: This will results in a LARGE info leak, limited only by the location by the first null character / unmmaped memory access(segfault)
            Therefore, we prefer this route (UTF8) */
            /*Q: Where can we actually see the leaked data if we use this route?
              A: Perhaps in some tool used to play the video (media player/ in-brower player) that presents the metadata of the video to the user, under the relevant metadataKey.
            */
            mFileMetaData->setCString(metadataKey, (const char *)buffer + 6);
        } else {
            // Convert from UTF-16 string to UTF-8 string.
            /* the written string length is limited by len16 value in case it does null termination */
            /* Can't easily find source code for this. but I think I've seen it applies null termination */
            String8 tmpUTF8str(framedata, len16); 
            
            /* This will result in a limited memory leak, which starts from buffer+6 */
            mFileMetaData->setCString(metadataKey, tmpUTF8str.string());
        }
    }

    delete[] buffer;
    buffer = NULL;

    return OK;
}


....

const char *string() const {
    return mPtr;
}

```