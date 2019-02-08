# MediaMuxer的jni代码分析

## jni方法的定义
MediaMuxer jni方法的定义在frameworks/base/media/jni/android_media_MediaMuxer.cpp中。
```
static const JNINativeMethod gMethods[] = {

    { "nativeAddTrack", "(J[Ljava/lang/String;[Ljava/lang/Object;)I",
        (void *)android_media_MediaMuxer_addTrack },

    { "nativeSetOrientationHint", "(JI)V",
        (void *)android_media_MediaMuxer_setOrientationHint},

    { "nativeSetLocation", "(JII)V",
        (void *)android_media_MediaMuxer_setLocation},

    { "nativeStart", "(J)V", (void *)android_media_MediaMuxer_start},

    { "nativeWriteSampleData", "(JILjava/nio/ByteBuffer;IIJI)V",
        (void *)android_media_MediaMuxer_writeSampleData },

    { "nativeStop", "(J)V", (void *)android_media_MediaMuxer_stop},

    { "nativeSetup", "(Ljava/io/FileDescriptor;I)J",
        (void *)android_media_MediaMuxer_native_setup },

    { "nativeRelease", "(J)V",
        (void *)android_media_MediaMuxer_native_release },

};
```
我们逐一看一看这些native方法的源码。

- android_media_MediaMuxer_addTrack
```
static jint android_media_MediaMuxer_addTrack(
        JNIEnv *env, jclass /* clazz */, jlong nativeObject, jobjectArray keys,
        jobjectArray values) {
    sp<MediaMuxer> muxer(reinterpret_cast<MediaMuxer *>(nativeObject));
    if (muxer == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "Muxer was not set up correctly");
        return -1;
    }

    sp<AMessage> trackformat;
    status_t err = ConvertKeyValueArraysToMessage(env, keys, values,
                                                  &trackformat);
    if (err != OK) {
        jniThrowException(env, "java/lang/IllegalArgumentException",
                          "ConvertKeyValueArraysToMessage got an error");
        return err;
    }

    // Return negative value when errors happen in addTrack.
    jint trackIndex = muxer->addTrack(trackformat);

    if (trackIndex < 0) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "Failed to add the track to the muxer");
        return -1;
    }
    return trackIndex;
}
```
首先，用nativeObject生成一个MediaMuxer（stagefright中，如无特别说明，下同)的强引用计数，如果muxer为NULL，抛出IllegalStateException。调用ConvertKeyValueArraysToMessage把keys和values数组，转换为AMessage的强引用计数trackformat，返回值保存在err中，如果err不等于OK，抛出IllegalArgumentException异常。调用MediaMuxer的addTrack方法，返回值保存在trackIndex中，如果trackIndex小于0，抛出IllegalStateException，返回-1。如果一起正常，就把trackIndex返回。

- android_media_MediaMuxer_setOrientationHint
```
static void android_media_MediaMuxer_setOrientationHint(
        JNIEnv *env, jclass /* clazz */, jlong nativeObject, jint degrees) {
    sp<MediaMuxer> muxer(reinterpret_cast<MediaMuxer *>(nativeObject));
    if (muxer == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "Muxer was not set up correctly");
        return;
    }
    status_t err = muxer->setOrientationHint(degrees);

    if (err != OK) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "Failed to set orientation hint");
        return;
    }

}
```
用nativeObject生成一个MediaMuxer的强引用计数,如果muxer为NULL，抛出IllegalStateException。调用MediaMuxer的setOrientationHint，并且结果保存在err中，抛出IllegalStateException。
start，stop，setLocation的定义和setOrientationHint类似。

- android_media_MediaMuxer_native_setup
```
static jlong android_media_MediaMuxer_native_setup(
        JNIEnv *env, jclass clazz, jobject fileDescriptor,
        jint format) {
    int fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
    ALOGV("native_setup: fd %d", fd);

    MediaMuxer::OutputFormat fileFormat =
        static_cast<MediaMuxer::OutputFormat>(format);
    sp<MediaMuxer> muxer = new MediaMuxer(fd, fileFormat);
    muxer->incStrong(clazz);
    return reinterpret_cast<jlong>(muxer.get());
}
```
根据参数fileDescriptor，获取一个文件描述符fd，输出文件格式就是参数foramt指定的格式。根据fd和format创建MediaMuxer的实例muxer，增加muxer的强引用计数。

- android_media_MediaMuxer_writeSampleData
```
static void android_media_MediaMuxer_writeSampleData(
        JNIEnv *env, jclass /* clazz */, jlong nativeObject, jint trackIndex,
        jobject byteBuf, jint offset, jint size, jlong timeUs, jint flags) {
    sp<MediaMuxer> muxer(reinterpret_cast<MediaMuxer *>(nativeObject));
    if (muxer == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "Muxer was not set up correctly");
        return;
    }

    // Try to convert the incoming byteBuffer into ABuffer
    void *dst = env->GetDirectBufferAddress(byteBuf);

    jlong dstSize;
    jbyteArray byteArray = NULL;

    if (dst == NULL) {

        byteArray =
            (jbyteArray)env->CallObjectMethod(byteBuf, gFields.arrayID);

        if (byteArray == NULL) {
            jniThrowException(env, "java/lang/IllegalArgumentException",
                              "byteArray is null");
            return;
        }

        jboolean isCopy;
        dst = env->GetByteArrayElements(byteArray, &isCopy);

        dstSize = env->GetArrayLength(byteArray);
    } else {
        dstSize = env->GetDirectBufferCapacity(byteBuf);
    }

    if (dstSize < (offset + size)) {
        ALOGE("writeSampleData saw wrong dstSize %lld, size  %d, offset %d",
              (long long)dstSize, size, offset);
        if (byteArray != NULL) {
            env->ReleaseByteArrayElements(byteArray, (jbyte *)dst, 0);
        }
        jniThrowException(env, "java/lang/IllegalArgumentException",
                          "sample has a wrong size");
        return;
    }

    sp<ABuffer> buffer = new ABuffer((char *)dst + offset, size);

    status_t err = muxer->writeSampleData(buffer, trackIndex, timeUs, flags);

    if (byteArray != NULL) {
        env->ReleaseByteArrayElements(byteArray, (jbyte *)dst, 0);
    }

    if (err != OK) {
        jniThrowException(env, "java/lang/IllegalStateException",
                          "writeSampleData returned an error");
    }
    return;
}
```
GetDirectBufferAddress获取byteBuf的共享内存地址dst，如果dst为NULL，CallObjectMethod(byteBuf, gFields.arrayID)调用java/nio/ByteBuffer中的array方法，尝试获取byteArray，如果byteArray依然为NULL，抛出IllegalArgumentException，返回。获取byteArray中的elements和byteArray的长度。如果之前获取的dst不为NULL，直接获取byteBuf的容量。如果dstSize小于offset加size的和，抛出IllegalArgumentException，如果byteArray不为空，需要释放它，返回。调用MediaMuxer类的writeSampleData方法。

- android_media_MediaMuxer_native_release
```
static void android_media_MediaMuxer_native_release(
        JNIEnv* /* env */, jclass clazz, jlong nativeObject) {
    sp<MediaMuxer> muxer(reinterpret_cast<MediaMuxer *>(nativeObject));
    if (muxer != NULL) {
        muxer->decStrong(clazz);
    }
}
```
减少muxer的强引用计数。