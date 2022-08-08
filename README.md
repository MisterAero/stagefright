Code of the framework that contains Stagefright can be found here:
https://github.com/aospo/platform_frameworks_av

MPEG-4 is made up of units such as atoms or boxes. These boxes start with header , which looks like this: A BoxType consists of a four-letter code. It's called FourCC,

BOXHEADER structure ()
Field           Type                    Comment
(chunk_size)
TotalSize       UI32                    The total size of the box in bytes, including this header

(chunk_type)
BoxType         UI32                    The type of atom

(sourceSize)
ExtendedSize    If TotalSize equals 1   The total 64-bit length of the box in bytes,
                UI64                    including this header

## Key points to look for

- Flow of exploit [+ mpeg4 file content]  
  Eventually mDataSource is describing a **single** mpeg4 file.
  First(?) we'll need to start with an **ftyp** atom in order to treat the file as an mpeg4 and as result - **MPEG4Extractor::parseChunk()** will get called afterwards
  - mDataSource should be initialized before calling parseChunk() ?
  - [Might need to add some more metaData related atoms here]
  - Heap shaping atoms
  - tx3g atom (invoke overflow)
  - invoke the overwriten hooked function (vtable attack)
  
- Constraints 
- Potential for exploitability (risk / exploitability chances)


**TL;DR : Search for the word "VULNERABILITY" or "BUG" for key points**

We were asked to inspect the code starting from  Mpeg4Extractor.cpp file.
And were told that our attack vector is mDataSource,
and that our entry point is Mpeg4Extractor::parseChunk() function.

**I recommend reading the documents in the following order:**
**(New files were added to the repo)**

- classes_and_definitions.md (Taken from Mpeg4Extractor.cpp)
- parse_chunk.md (for first impression on how this function works)

### Vulnureabilities:
- tx3g.md (for tx3g case, Heap overflow caused by integer overflow)
- covr.md (for covr case, Heap overflows caused by integer overflow & underflow)
- parse3gpp.md (for 3gpp case, info leak (on heap) caused by integer underflow)
  
### Primitives due to bugs:
- Heap shaping 
  - pssh.md (for pssh case, Heap shaping, one allocation of chosen size)
  - stbl.md (for stbl case, Heap shaping : Overwrite memory content of mDataSource object)

- OOB heap modification
  - ctts.md - OOB Applying of ntohl() per each 4 Bytes starting from the allocated heap chunk 
  - stss.md (same as stsc.md) - doesn't seem to be beneficial
  - stts.md - OOB read from heap **INTO** another chunk on the heap which stores the sample table of the currently cached atom (OOB write) at specific offset (mSampleToChunkEntries field) - doesn't seem to be beneficial

### General primitives:
- setCachedRange.md - Clearing the cached range of the mDataSource object

....


### Links for resources for further reading:
-------------------
#### Stagefright Vulnerabilities summary & exploitation

https://googleprojectzero.blogspot.com/2015/09/stagefrightened.html  
https://research.nccgroup.com/wp-content/uploads/2020/07/libstagefright-exploit-notes.pdf 
https://www.exploit-db.com/docs/english/39527-metaphor---a-(real)-real-%C2%ADlife-stagefright-exploit.pdf
https://ia903108.us.archive.org/5/items/stagefright_201805/stagefright.pdf
https://web.archive.org/web/20160331185650/http://translate.wooyun.io/2015/08/08/Stagefright-Vulnerability-Disclosure.html
https://web.archive.org/web/20160413230809/http://blog.fortinet.com/post/cryptogirl-on-stagefright-a-detailed-explanation
http://www.cs.toronto.edu/~arnold/427/16s/csc427_16s/indepth/Stagefright/Stagefright-and-Metaphor.pdf
https://raw.githubusercontent.com/NorthBit/Public/master/NorthBit-Metaphor.pdf

