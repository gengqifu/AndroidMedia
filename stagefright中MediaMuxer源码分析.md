# stagefright中MediaMuxer源码分析

- 私有属性定义
MediaMuxer.h中定义了一些类的属性：
```
const OutputFormat mFormat;
    sp<MediaWriter> mWriter;
    Vector< sp<MediaAdapter> > mTrackList;  // Each track has its MediaAdapter.
    sp<MetaData> mFileMeta;  // Metadata for the whole file.

    Mutex mMuxerLock;

    enum State {
        UNINITIALIZED,
        INITIALIZED,
        STARTED,
        STOPPED
    };
    State mState;
```
这里分别定义了输出格式mFormat，用于输出帧数据到文件的mWriter，MediaAdapter的数组向量mTrackList，文件的MetaData的mFileMeta，同步锁mMuxerLock。枚举类型State和java代码中的状态对应。

- 构造函数和析构函数
```
MediaMuxer::MediaMuxer(int fd, OutputFormat format)
    : mFormat(format),
      mState(UNINITIALIZED) {
    if (format == OUTPUT_FORMAT_MPEG_4) {
        mWriter = new MPEG4Writer(fd);
    } else if (format == OUTPUT_FORMAT_WEBM) {
        mWriter = new WebmWriter(fd);
    }

    if (mWriter != NULL) {
        mFileMeta = new MetaData;
        mState = INITIALIZED;
    }
}

MediaMuxer::~MediaMuxer() {
    Mutex::Autolock autoLock(mMuxerLock);

    // Clean up all the internal resources.
    mFileMeta.clear();
    mWriter.clear();
    mTrackList.clear();
}
```
初始化输出格式mFormat，默认初始状态为UNINITIALIZED，mWriter不为空的情况下，状态设置为INITIALIZED。
析构函数主要是清理资源。

- addTrack
```
ssize_t MediaMuxer::addTrack(const sp<AMessage> &format) {
    Mutex::Autolock autoLock(mMuxerLock);

    if (format.get() == NULL) {
        ALOGE("addTrack() get a null format");
        return -EINVAL;
    }

    if (mState != INITIALIZED) {
        ALOGE("addTrack() must be called after constructor and before start().");
        return INVALID_OPERATION;
    }

    sp<MetaData> trackMeta = new MetaData;
    convertMessageToMetaData(format, trackMeta);

    sp<MediaAdapter> newTrack = new MediaAdapter(trackMeta);
    status_t result = mWriter->addSource(newTrack);
    if (result == OK) {
        return mTrackList.add(newTrack);
    }
    return -1;
}
```
这里要做同步锁定。如果format为NULL，返回-EINVAL。mState必须是INITIALIZED，否则，返回INVALID_OPERATION。convertMessageToMetaData(format, trackMeta)把AMessage的sp format转换为MetaData的sp，再根据trackMetaData生成一个新的MediaAdapter newTrack，添加到mWrite的source，返回值记录在result中。如果result等于OK，newTrack添加到mTrackList，并返回；否则，返回-1。

- start和stop
```
status_t MediaMuxer::start() {
    Mutex::Autolock autoLock(mMuxerLock);
    if (mState == INITIALIZED) {
        mState = STARTED;
        mFileMeta->setInt32(kKeyRealTimeRecording, false);
        return mWriter->start(mFileMeta.get());
    } else {
        ALOGE("start() is called in invalid state %d", mState);
        return INVALID_OPERATION;
    }
}

status_t MediaMuxer::stop() {
    Mutex::Autolock autoLock(mMuxerLock);

    if (mState == STARTED) {
        mState = STOPPED;
        for (size_t i = 0; i < mTrackList.size(); i++) {
            if (mTrackList[i]->stop() != OK) {
                return INVALID_OPERATION;
            }
        }
        return mWriter->stop();
    } else {
        ALOGE("stop() is called in invalid state %d", mState);
        return INVALID_OPERATION;
    }
}
```
start和stop方法都需要加同步锁。
start方法必须在INITIALIZED状态下调用，否则，返回INVALID_OPERATION。如果状态是INITIALIZED，首先要把状态转变为STARTED，实时录制设置为false，调用mWriter的start方法，并且以该方法的返回值作为返回值。
stop方法必须在STARTED状态下调用，否则，也要返回INVALID_OPERATION。如果状态是STARTED，先把状态转换为STOPPED。依此对mTrackList中的MediaAdatper调用stop，如果任意一次调用不返回OK，返回INVALID_OPERATION。循环正常结束之后，调用mWriter的stop，停止Writer的运行。

- writeSampleData
```
status_t MediaMuxer::writeSampleData(const sp<ABuffer> &buffer, size_t trackIndex,
                                     int64_t timeUs, uint32_t flags) {
    Mutex::Autolock autoLock(mMuxerLock);

    if (buffer.get() == NULL) {
        ALOGE("WriteSampleData() get an NULL buffer.");
        return -EINVAL;
    }

    if (mState != STARTED) {
        ALOGE("WriteSampleData() is called in invalid state %d", mState);
        return INVALID_OPERATION;
    }

    if (trackIndex >= mTrackList.size()) {
        ALOGE("WriteSampleData() get an invalid index %zu", trackIndex);
        return -EINVAL;
    }

    MediaBuffer* mediaBuffer = new MediaBuffer(buffer);

    mediaBuffer->add_ref(); // Released in MediaAdapter::signalBufferReturned().
    mediaBuffer->set_range(buffer->offset(), buffer->size());

    sp<MetaData> sampleMetaData = mediaBuffer->meta_data();
    sampleMetaData->setInt64(kKeyTime, timeUs);
    // Just set the kKeyDecodingTime as the presentation time for now.
    sampleMetaData->setInt64(kKeyDecodingTime, timeUs);

    if (flags & MediaCodec::BUFFER_FLAG_SYNCFRAME) {
        sampleMetaData->setInt32(kKeyIsSyncFrame, true);
    }

    sp<MediaAdapter> currentTrack = mTrackList[trackIndex];
    // This pushBuffer will wait until the mediaBuffer is consumed.
    return currentTrack->pushBuffer(mediaBuffer);
}
```
需要加同步锁。
如果buffer为NULL，返回EINVAL。mState必须为STARTED，否则，返回INVALID_OPERATION。参数trackIndex必须不小于mTrackList的size，否则，返回EINVAL。mediaBuffer类型的指针指向一个新创建的MediaBuffer，增加计数，范围介于buffer的offset和offset+size之间。smapleMetaData是mediaBuffer中的MetaData，设置time和decT，参数flags设置了MediaCodec::BUFFER_FLAG_SYNCFRAME，那么，要把kKeyIsSyncFrame设置为true（sync frame, 一般来说就是关键帧）。从mTrackList中获取trackIndex位置的MediaAdapter currentTrack。调用currentTrack的的pushBuffer，pushBuffer要一直到mediaBuffer被消费掉才返回。