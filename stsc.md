# stsc

**Similar behaviour to stts case, go to stts.md for more information**

```cpp
        case FOURCC('s', 't', 's', 'c'):
        {
            status_t err =
                mLastTrack->sampleTable->setSampleToChunkParams(
                        data_offset, chunk_data_size);

            *offset += chunk_size;

            if (err != OK) {
                return err;
            }

            break;
        }
```