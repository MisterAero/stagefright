**mDataSource** is defined in **GenericSource**.h inside the struct **struct NuPlayer::GenericSource : public NuPlayer::Source {**

```cpp
sp<DataSource> mDataSource;
```

It is being initialized in: 
```cpp
void NuPlayer::GenericSource::onPrepareAsync() {
    // delayed data source creation
    if (mDataSource == NULL) {
        if (!mUri.empty()) {
            const char* uri = mUri.c_str();
            mIsWidevine = !strncasecmp(uri, "widevine://", 11);

            if (!strncasecmp("http://", uri, 7)
                    || !strncasecmp("https://", uri, 8)
                    || mIsWidevine) {
                mHttpSource = DataSource::CreateMediaHTTP(mHTTPService);
                if (mHttpSource == NULL) {
                    ALOGE("Failed to create http source!");
                    notifyPreparedAndCleanup(UNKNOWN_ERROR);
                    return;
                }
            }


            /************* Call to CreateFromURI() *************/
            /* Our point of interest where mDataSource gets initialized */
            mDataSource = DataSource::CreateFromURI(
                   mHTTPService, uri, &mUriHeaders, &mContentType,
                   static_cast<HTTPBase *>(mHttpSource.get()));
                   /****************************************/
        } else {
            // set to false first, if the extractor
            // comes back as secure, set it to true then.
            mIsWidevine = false;

            mDataSource = new FileSource(mFd, mOffset, mLength);
        }

        if (mDataSource == NULL) {
            ALOGE("Failed to create data source!");
            notifyPreparedAndCleanup(UNKNOWN_ERROR);
            return;
        }

        if (mDataSource->flags() & DataSource::kIsCachingDataSource) {
            mCachedSource = static_cast<NuCachedSource2 *>(mDataSource.get());
        }

        if (mIsWidevine || mCachedSource != NULL) {
            schedulePollBuffering();
        }
    }

    // check initial caching status
    status_t err = prefillCacheIfNecessary();
    if (err != OK) {
        if (err == -EAGAIN) {
            (new AMessage(kWhatPrepareAsync, id()))->post(200000);
        } else {
            ALOGE("Failed to prefill data cache!");
            notifyPreparedAndCleanup(UNKNOWN_ERROR);
        }
        return;
    }

    // init extrator from data source
    err = initFromDataSource();

    if (err != OK) {
        ALOGE("Failed to init from data source!");
        notifyPreparedAndCleanup(err);
        return;
    }

    if (mVideoTrack.mSource != NULL) {
        sp<MetaData> meta = doGetFormatMeta(false /* audio */);
        sp<AMessage> msg = new AMessage;
        err = convertMetaDataToMessage(meta, &msg);
        if(err != OK) {
            notifyPreparedAndCleanup(err);
            return;
        }
        notifyVideoSizeChanged(msg);
    }

    notifyFlagsChanged(
            (mIsWidevine ? FLAG_SECURE : 0)
            | FLAG_CAN_PAUSE
            | FLAG_CAN_SEEK_BACKWARD
            | FLAG_CAN_SEEK_FORWARD
            | FLAG_CAN_SEEK);

    if ((mWVMExtractor == NULL && mCachedSource == NULL) ||
        mPrepareState == STATE_UNPREPARED_EOS) {
        setPrepareState(STATE_PREPARED);
    } else {
        setPrepareState(STATE_PREPARING);
    }
}
```

So it is created with the following line (it might can be also initialized in some other function called from void NuPlayer::GenericSource::onPrepareAsync() , but they are not our focus here):

```cpp
```cpp
mDataSource = DataSource::CreateFromURI(
                   mHTTPService, uri, &mUriHeaders, &mContentType,
                   static_cast<HTTPBase *>(mHttpSource.get()));
```

```cpp
// static
sp<DataSource> DataSource::CreateFromURI(
        const sp<IMediaHTTPService> &httpService,
        const char *uri,
        const KeyedVector<String8, String8> *headers,
        String8 *contentType,
        HTTPBase *httpSource) {
    if (contentType != NULL) {
        *contentType = "";
    }

    /*
     * Widevine is a proprietary digital rights management (DRM) technology from Google used by the Chromium and Firefox web browsers (including some derivatives) */
    bool isWidevine = !strncasecmp("widevine://", uri, 11);

    sp<DataSource> source; /* The returned data source */
    if (!strncasecmp("file://", uri, 7)) {
        /* file:// URI */
        source = new FileSource(uri + 7);
    } else if (!strncasecmp("http://", uri, 7)
            || !strncasecmp("https://", uri, 8)
            || isWidevine) {
                /* http(s) URI  or Widevine*/
        if (httpService == NULL) {
            ALOGE("Invalid http service!");
            return NULL; /* Error */
        }

        if (httpSource == NULL) {
            /* Create a new http connection using a virtual function */
            sp<IMediaHTTPConnection> conn = httpService->makeHTTPConnection();
            if (conn == NULL) {
                ALOGE("Failed to make http connection from http service!");
                return NULL; /* Error */
            }
            /* Create a new http source using the newly created http connection */
            httpSource = new MediaHTTP(conn);
        }

        String8 tmp;
        if (isWidevine) {
            tmp = String8("http://");
            tmp.append(uri + 11);

            uri = tmp.string();
        }

        String8 cacheConfig;
        bool disconnectAtHighwatermark;
        KeyedVector<String8, String8> nonCacheSpecificHeaders;
        if (headers != NULL) {
            /* Couldn't find an assignment operator for KeyedVector , so I guess it does shallow copy. 
            This initializes the nonCacheSpecificHeaders with the headers from the caller */
            nonCacheSpecificHeaders = *headers; 
            
            /* modifies all sent arguments */
            NuCachedSource2::RemoveCacheSpecificHeaders(
                    &nonCacheSpecificHeaders,
                    &cacheConfig,
                    &disconnectAtHighwatermark);
        }

        if (httpSource->connect(uri, &nonCacheSpecificHeaders) != OK) {
            ALOGE("Failed to connect http source!");
            return NULL;
        }

        if (!isWidevine) {
            if (contentType != NULL) {
                *contentType = httpSource->getMIMEType();
            }
            /* 1st time we cache the original httpSource */
           /* Create a cached source version of the http source */
           /* We were given a hint regarding caching, so this is the only place that some form of caching is used before assigning value to mDataSource (gets set to "source") */ 
            source = new NuCachedSource2(
                    httpSource,
                    cacheConfig.isEmpty() ? NULL : cacheConfig.string(),
                    disconnectAtHighwatermark);
            /********************************************************/
        } else {
            // We do not want that prefetching, caching, datasource wrapper
            // in the widevine:// case.
            source = httpSource; /* probabely for DRM reasons */
        }
    } else if (!strncasecmp("data:", uri, 5)) {
        source = DataURISource::Create(uri);
    } else {
        // Assume it's a filename.
        source = new FileSource(uri);
    }

    if (source == NULL || source->initCheck() != OK) {
        return NULL;
    }

    return source;
}

sp<DataSource> DataSource::CreateMediaHTTP(const sp<IMediaHTTPService> &httpService) {
    if (httpService == NULL) {
        return NULL;
    }

    sp<IMediaHTTPConnection> conn = httpService->makeHTTPConnection();
    if (conn == NULL) {
        return NULL;
    } else {
        return new MediaHTTP(conn);
    }
}

String8 DataSource::getMIMEType() const {
    return String8("application/octet-stream");
}

```

```cpp

// static
void NuCachedSource2::RemoveCacheSpecificHeaders(
        KeyedVector<String8, String8> *headers,
        String8 *cacheConfig,
        bool *disconnectAtHighwatermark) {
    *cacheConfig = String8();
    *disconnectAtHighwatermark = false;

    if (headers == NULL) {
        return;
    }

    ssize_t index;
    if ((index = headers->indexOfKey(String8("x-cache-config"))) >= 0) {
        /* read the "special" cache config from the headers */
        *cacheConfig = headers->valueAt(index);
        
        /* remove it from the headers */
        headers->removeItemsAt(index);

        ALOGV("Using special cache config '%s'", cacheConfig->string());
    }

    if ((index = headers->indexOfKey(
                    String8("x-disconnect-at-highwatermark"))) >= 0) {
        *disconnectAtHighwatermark = true;
        headers->removeItemsAt(index);

        ALOGV("Client requested disconnection at highwater mark");
    }
}

```

```cpp

status_t MediaHTTP::connect(
        const char *uri,
        const KeyedVector<String8, String8> *headers,
        off64_t /* offset */) {
    if (mInitCheck != OK) {
        return mInitCheck;
    }

    KeyedVector<String8, String8> extHeaders;
    if (headers != NULL) {
        extHeaders = *headers;
    }
    extHeaders.add(String8("User-Agent"), String8(MakeUserAgent().c_str()));

    bool success = mHTTPConnection->connect(uri, &extHeaders);

    mLastHeaders = extHeaders;
    mLastURI = uri;

    mCachedSizeValid = false;

    return success ? OK : UNKNOWN_ERROR;
}

```


```cpp

NuCachedSource2::NuCachedSource2(
        const sp<DataSource> &source,
        const char *cacheConfig,
        bool disconnectAtHighwatermark)
    : mSource(source),
      mReflector(new AHandlerReflector<NuCachedSource2>(this)),
      mLooper(new ALooper),
      mCache(new PageCache(kPageSize)),
      mCacheOffset(0),
      mFinalStatus(OK),
      mLastAccessPos(0),
      mFetching(true),
      mDisconnecting(false),
      mLastFetchTimeUs(-1),
      mNumRetriesLeft(kMaxNumRetries),
      mHighwaterThresholdBytes(kDefaultHighWaterThreshold),
      mLowwaterThresholdBytes(kDefaultLowWaterThreshold),
      mKeepAliveIntervalUs(kDefaultKeepAliveIntervalUs),
      mDisconnectAtHighwatermark(disconnectAtHighwatermark),
      mSuspended(false) {
    // We are NOT going to support disconnect-at-highwatermark indefinitely
    // and we are not guaranteeing support for client-specified cache
    // parameters. Both of these are temporary measures to solve a specific
    // problem that will be solved in a better way going forward.

    updateCacheParamsFromSystemProperty();

    if (cacheConfig != NULL) {
        updateCacheParamsFromString(cacheConfig);
    }

    if (mDisconnectAtHighwatermark) {
        // Makes no sense to disconnect and do keep-alives...
        mKeepAliveIntervalUs = 0;
    }

    mLooper->setName("NuCachedSource2");
    mLooper->registerHandler(mReflector);

    // Since it may not be obvious why our looper thread needs to be
    // able to call into java since it doesn't appear to do so at all...
    // IMediaHTTPConnection may be (and most likely is) implemented in JAVA
    // and a local JAVA IBinder will call directly into JNI methods.
    // So whenever we call DataSource::readAt it may end up in a call to
    // IMediaHTTPConnection::readAt and therefore call back into JAVA.
    mLooper->start(false /* runOnCallingThread */, true /* canCallJava */);

    Mutex::Autolock autoLock(mLock);
    (new AMessage(kWhatFetchMore, mReflector->id()))->post();
}

NuCachedSource2::~NuCachedSource2() {
    mLooper->stop();
    mLooper->unregisterHandler(mReflector->id());

    delete mCache;
    mCache = NULL;
}
```