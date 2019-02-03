# libstagefright中MediaCodec源码分析

和前两篇一样，我们按照MediaCodec的各个状态来分析libstagefright中MediaCodec的源代码。

- configure
首先我们看一下configure在libstagefright中MediaCodec中的定义：
```

438status_t MediaCodec::configure(
439        const sp<AMessage> &format,
440        const sp<Surface> &surface,
441        const sp<ICrypto> &crypto,
442        uint32_t flags) {
443    sp<AMessage> msg = new AMessage(kWhatConfigure, this);
444
445    if (mIsVideo) {
446        format->findInt32("width", &mVideoWidth);
447        format->findInt32("height", &mVideoHeight);
448        if (!format->findInt32("rotation-degrees", &mRotationDegrees)) {
449            mRotationDegrees = 0;
450        }
451
452        // Prevent possible integer overflow in downstream code.
453        if (mInitIsEncoder
454                && (uint64_t)mVideoWidth * mVideoHeight > (uint64_t)INT32_MAX / 4) {
455            ALOGE("buffer size is too big, width=%d, height=%d", mVideoWidth, mVideoHeight);
456            return BAD_VALUE;
457        }
458    }
459
460    msg->setMessage("format", format);
461    msg->setInt32("flags", flags);
462    msg->setObject("surface", surface);
463
464    if (crypto != NULL) {
465        msg->setPointer("crypto", crypto.get());
466    }
467
468    // save msg for reset
469    mConfigureMsg = msg;
470
471    status_t err;
472    Vector<MediaResource> resources;
473    MediaResource::Type type = (mFlags & kFlagIsSecure) ?
474            MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
475    MediaResource::SubType subtype =
476            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
477    resources.push_back(MediaResource(type, subtype, 1));
478    // Don't know the buffer size at this point, but it's fine to use 1 because
479    // the reclaimResource call doesn't consider the requester's buffer size for now.
480    resources.push_back(MediaResource(MediaResource::kGraphicMemory, 1));
481    for (int i = 0; i <= kMaxRetry; ++i) {
482        if (i > 0) {
483            // Don't try to reclaim resource for the first time.
484            if (!mResourceManagerService->reclaimResource(resources)) {
485                break;
486            }
487        }
488
489        sp<AMessage> response;
490        err = PostAndAwaitResponse(msg, &response);
491        if (err != OK && err != INVALID_OPERATION) {
492            // MediaCodec now set state to UNINITIALIZED upon any fatal error.
493            // To maintain backward-compatibility, do a reset() to put codec
494            // back into INITIALIZED state.
495            // But don't reset if the err is INVALID_OPERATION, which means
496            // the configure failure is due to wrong state.
497
498            ALOGE("configure failed with err 0x%08x, resetting...", err);
499            reset();
500        }
501        if (!isResourceError(err)) {
502            break;
503        }
504    }
505    return err;
506}
```
首先，生成一个AMessage的心的强引用计数msg，AMessage的构造函数的两个参数，一个是枚举值kWhatConfigure（“init”）,在MediaCodec.h中定义，另一个参数应该是AHandler类型，而MediaCodec就是AHandler的子类。现在来看一下是否是video（mIsVideo)，如果是，要从参数format中取出视频的width和height，分别把值赋给类属性mVideoWidth和mVideoHeight；获取旋转角度，赋值给mRotationsDegrees，如果不存在该属性，就把mRotationsDegrees设成0度。如果encoder，那么宽高相乘要小于INT32_MAX的四分之一。一次设置msg的format，flags和surface。如果参数crypto不为NULL，设置msg的crypto。把msg赋值给类属性mConfigrueMsg。判断是否安全编码，把结果保存在type中；判断是video还是audio，结果保存在subtype中。生成两种MediaResource，根据type，subtype，另一种根据MediaResource::kGraphicMemory，把他们都放入之前生命的MediaResource类型的Vector resources中。进行一个循环，除了第一个循环，每次循环开始，都需要reclaim resource。调用PostAndAwaitResponse(msg, &response)（这会调用AMessage类中的同名方法，最终通过ALooper发送消息），发送msg，并且把响应存储到response，把返回值存储到err，如果err不等于NO_MEMORY，跳出循环。最后，返回err的值。
- start
下面，我们来看看start状态。
```
status_t MediaCodec::start() {
568    sp<AMessage> msg = new AMessage(kWhatStart, this);
569
570    status_t err;
571    Vector<MediaResource> resources;
572    MediaResource::Type type = (mFlags & kFlagIsSecure) ?
573            MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
574    MediaResource::SubType subtype =
575            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
576    resources.push_back(MediaResource(type, subtype, 1));
577    // Don't know the buffer size at this point, but it's fine to use 1 because
578    // the reclaimResource call doesn't consider the requester's buffer size for now.
579    resources.push_back(MediaResource(MediaResource::kGraphicMemory, 1));
580    for (int i = 0; i <= kMaxRetry; ++i) {
581        if (i > 0) {
582            // Don't try to reclaim resource for the first time.
583            if (!mResourceManagerService->reclaimResource(resources)) {
584                break;
585            }
586            // Recover codec from previous error before retry start.
587            err = reset();
588            if (err != OK) {
589                ALOGE("retrying start: failed to reset codec");
590                break;
591            }
592            sp<AMessage> response;
593            err = PostAndAwaitResponse(mConfigureMsg, &response);
594            if (err != OK) {
595                ALOGE("retrying start: failed to configure codec");
596                break;
597            }
598        }
599
600        sp<AMessage> response;
601        err = PostAndAwaitResponse(msg, &response);
602        if (!isResourceError(err)) {
603            break;
604        }
605    }
606    return err;
607}
```
首先以kWhatStart状态创建一个AMessage的强引用计数msg。同configure一样，type和subtype一样，分别表示是否安全编码和音频或视频编码。分别以type和subtype创建MediaResource，存入向量resources中。在循环之中，如果不是首次循环，需要对资源进行回收再利用。每次循环都要进行reset。如果reset失败，终止循环。调用`PostAndAwaitResponse(mConfigureMsg, &response)`发送消息并等待返回，mConfigureMsg在configure阶段已经赋值。如果返回值不等于OK，终止循环。调用`PostAndAwaitResponse(msg, &response)`发送消息并等待返回。如果返回值不是NO_MEMORY，跳出循环。最后，函数返回最后设置的err的值。

- dequeueInputBuffe
现在来看看输入缓冲区的出列处理。
```
status_t MediaCodec::dequeueInputBuffer(size_t *index, int64_t timeoutUs) {
748    sp<AMessage> msg = new AMessage(kWhatDequeueInputBuffer, this);
749    msg->setInt64("timeoutUs", timeoutUs);
750
751    sp<AMessage> response;
752    status_t err;
753    if ((err = PostAndAwaitResponse(msg, &response)) != OK) {
754        return err;
755    }
756
757    CHECK(response->findSize("index", index));
758
759    return OK;
760}
```
同样是，生成AMessage的强引用计数msg和response。调用PostAndAwaitResponse(msg, &response))发送消息并等待返回。如果返回不是OK，就把返回的error code作为函数的返回值。否则，返回OK。

- queueInputBuffer
将数据压入输入缓冲区队列。
```
status_t MediaCodec::queueInputBuffer(
689        size_t index,
690        size_t offset,
691        size_t size,
692        int64_t presentationTimeUs,
693        uint32_t flags,
694        AString *errorDetailMsg) {
695    if (errorDetailMsg != NULL) {
696        errorDetailMsg->clear();
697    }
698
699    sp<AMessage> msg = new AMessage(kWhatQueueInputBuffer, this);
700    msg->setSize("index", index);
701    msg->setSize("offset", offset);
702    msg->setSize("size", size);
703    msg->setInt64("timeUs", presentationTimeUs);
704    msg->setInt32("flags", flags);
705    msg->setPointer("errorDetailMsg", errorDetailMsg);
706
707    sp<AMessage> response;
708    return PostAndAwaitResponse(msg, &response);
709}
```
首先，创建AMessage的强引用计数msg，用参数index，offset，size，presentationTimeUs，flags，errorDetailMsg来初始化msg的index，offset，size，timeUs，flags，errorDetailMsg。PostAndAwaitResponse(msg, &response)发送并等待消息返回，并且，用这个调用的返回值作为函数的返回值。

- stop
stop的代码比较简单。
```
status_t MediaCodec::stop() {
610    sp<AMessage> msg = new AMessage(kWhatStop, this);
611
612    sp<AMessage> response;
613    return PostAndAwaitResponse(msg, &response);
614}
```
同样是，生成AMessage的强引用计数msg和response。调用PostAndAwaitResponse(msg, &response))发送消息并等待返回。

- reset
reset的代码如下：
```
status_t MediaCodec::reset() {
654    /* When external-facing MediaCodec object is created,
655       it is already initialized.  Thus, reset is essentially
656       release() followed by init(), plus clearing the state */
657
658    status_t err = release();
659
660    // unregister handlers
661    if (mCodec != NULL) {
662        if (mCodecLooper != NULL) {
663            mCodecLooper->unregisterHandler(mCodec->id());
664        } else {
665            mLooper->unregisterHandler(mCodec->id());
666        }
667        mCodec = NULL;
668    }
669    mLooper->unregisterHandler(id());
670
671    mFlags = 0;    // clear all flags
672    mStickyError = OK;
673
674    // reset state not reset by setState(UNINITIALIZED)
675    mReplyID = 0;
676    mDequeueInputReplyID = 0;
677    mDequeueOutputReplyID = 0;
678    mDequeueInputTimeoutGeneration = 0;
679    mDequeueOutputTimeoutGeneration = 0;
680    mHaveInputSurface = false;
681
682    if (err == OK) {
683        err = init(mInitName, mInitNameIsType, mInitIsEncoder);
684    }
685    return err;
686}
```
reset首先调用release释放codec。如果mCodec（CodecBase类型）不为NULL，如果mCodecLooper也不为NULL，首先对mCodecLooper反注册Handler，否则，对mLooper反注册Handler，并且把mCodec置为NULL。接下来进行一系列重新赋值操作，都赋值为初始化值。如果err等于OK，调用init进行初始化。

- release
我们看看reset中调用，并且可以做为一个独立状态的release。
```
status_t MediaCodec::release() {
647    sp<AMessage> msg = new AMessage(kWhatRelease, this);
648
649    sp<AMessage> response;
650    return PostAndAwaitResponse(msg, &response);
651}
```
同样是，生成AMessage的强引用计数msg和response。调用PostAndAwaitResponse(msg, &response))发送消息并等待返回。