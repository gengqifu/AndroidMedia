# android_media_MediaCodec.cpp源码分析

这里我们来分析和MediaCodec.java对应的native层类：android_media_MediaCodec.cpp的源代码。
在该类的最后，我们会看到这样的代码：
```
static const JNINativeMethod gMethods[] = {
1881    { "native_release", "()V", (void *)android_media_MediaCodec_release },
1882
1883    { "native_reset", "()V", (void *)android_media_MediaCodec_reset },
1884
1885    { "native_releasePersistentInputSurface",
1886      "(Landroid/view/Surface;)V",
1887       (void *)android_media_MediaCodec_releasePersistentInputSurface},
1888
1889    { "native_createPersistentInputSurface",
1890      "()Landroid/media/MediaCodec$PersistentSurface;",
1891      (void *)android_media_MediaCodec_createPersistentInputSurface },
1892
1893    { "native_setInputSurface", "(Landroid/view/Surface;)V",
1894      (void *)android_media_MediaCodec_setInputSurface },
1895
1896    { "native_enableOnFrameRenderedListener", "(Z)V",
1897      (void *)android_media_MediaCodec_native_enableOnFrameRenderedListener },
1898
1899    { "native_setCallback",
1900      "(Landroid/media/MediaCodec$Callback;)V",
1901      (void *)android_media_MediaCodec_native_setCallback },
1902
1903    { "native_configure",
1904      "([Ljava/lang/String;[Ljava/lang/Object;Landroid/view/Surface;"
1905      "Landroid/media/MediaCrypto;I)V",
1906      (void *)android_media_MediaCodec_native_configure },
1907
1908    { "native_setSurface",
1909      "(Landroid/view/Surface;)V",
1910      (void *)android_media_MediaCodec_native_setSurface },
1911
1912    { "createInputSurface", "()Landroid/view/Surface;",
1913      (void *)android_media_MediaCodec_createInputSurface },
1914
1915    { "native_start", "()V", (void *)android_media_MediaCodec_start },
1916    { "native_stop", "()V", (void *)android_media_MediaCodec_stop },
1917    { "native_flush", "()V", (void *)android_media_MediaCodec_flush },
1918
1919    { "native_queueInputBuffer", "(IIIJI)V",
1920      (void *)android_media_MediaCodec_queueInputBuffer },
1921
1922    { "native_queueSecureInputBuffer", "(IILandroid/media/MediaCodec$CryptoInfo;JI)V",
1923      (void *)android_media_MediaCodec_queueSecureInputBuffer },
1924
1925    { "native_dequeueInputBuffer", "(J)I",
1926      (void *)android_media_MediaCodec_dequeueInputBuffer },
1927
1928    { "native_dequeueOutputBuffer", "(Landroid/media/MediaCodec$BufferInfo;J)I",
1929      (void *)android_media_MediaCodec_dequeueOutputBuffer },
1930
1931    { "releaseOutputBuffer", "(IZZJ)V",
1932      (void *)android_media_MediaCodec_releaseOutputBuffer },
1933
1934    { "signalEndOfInputStream", "()V",
1935      (void *)android_media_MediaCodec_signalEndOfInputStream },
1936
1937    { "getFormatNative", "(Z)Ljava/util/Map;",
1938      (void *)android_media_MediaCodec_getFormatNative },
1939
1940    { "getOutputFormatNative", "(I)Ljava/util/Map;",
1941      (void *)android_media_MediaCodec_getOutputFormatForIndexNative },
1942
1943    { "getBuffers", "(Z)[Ljava/nio/ByteBuffer;",
1944      (void *)android_media_MediaCodec_getBuffers },
1945
1946    { "getBuffer", "(ZI)Ljava/nio/ByteBuffer;",
1947      (void *)android_media_MediaCodec_getBuffer },
1948
1949    { "getImage", "(ZI)Landroid/media/Image;",
1950      (void *)android_media_MediaCodec_getImage },
1951
1952    { "getName", "()Ljava/lang/String;",
1953      (void *)android_media_MediaCodec_getName },
1954
1955    { "setParameters", "([Ljava/lang/String;[Ljava/lang/Object;)V",
1956      (void *)android_media_MediaCodec_setParameters },
1957
1958    { "setVideoScalingMode", "(I)V",
1959      (void *)android_media_MediaCodec_setVideoScalingMode },
1960
1961    { "native_init", "()V", (void *)android_media_MediaCodec_native_init },
1962
1963    { "native_setup", "(Ljava/lang/String;ZZ)V",
1964      (void *)android_media_MediaCodec_native_setup },
1965
1966    { "native_finalize", "()V",
1967      (void *)android_media_MediaCodec_native_finalize },
1968};
```
这些就是在java层和native层代码的一个对应列表。通过下面这个函数，实现natvie方法的注册：
```
int register_android_media_MediaCodec(JNIEnv *env) {
1971    return AndroidRuntime::registerNativeMethods(env,
1972                "android/media/MediaCodec", gMethods, NELEM(gMethods));
1973}
```
接下来我们逐次分析MediaCodec各个阶段对应的native方法。首先要看的就是configure方法。
- configure
```
static void android_media_MediaCodec_native_configure(
956        JNIEnv *env,
957        jobject thiz,
958        jobjectArray keys, jobjectArray values,
959        jobject jsurface,
960        jobject jcrypto,
961        jint flags) {
962    sp<JMediaCodec> codec = getMediaCodec(env, thiz);
963
964    if (codec == NULL) {
965        throwExceptionAsNecessary(env, INVALID_OPERATION);
966        return;
967    }
968
969    sp<AMessage> format;
970    status_t err = ConvertKeyValueArraysToMessage(env, keys, values, &format);
971
972    if (err != OK) {
973        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
974        return;
975    }
976
977    sp<IGraphicBufferProducer> bufferProducer;
978    if (jsurface != NULL) {
979        sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
980        if (surface != NULL) {
981            bufferProducer = surface->getIGraphicBufferProducer();
982        } else {
983            jniThrowException(
984                    env,
985                    "java/lang/IllegalArgumentException",
986                    "The surface has been released");
987            return;
988        }
989    }
990
991    sp<ICrypto> crypto;
992    if (jcrypto != NULL) {
993        crypto = JCrypto::GetCrypto(env, jcrypto);
994    }
995
996    err = codec->configure(format, bufferProducer, crypto, flags);
997
998    throwExceptionAsNecessary(env, err);
999}
1000
```
通过getMediaCodec方法，获取一个JMediaCodec的强指针codec。如果codec为NULL，就抛出一个INVALID_OPERATION异常。定义AMessage的强引用format，通过ConvertKeyValueArraysToMessage(env, keys, values, &format)，将keys和values转换为配置信息，并且将format信息，存储到format中。如果方法返回error，就抛出IllegalArgumentException异常，并且从android_media_MediaCodec_native_configure返回。定义IGraphicBufferProducer的强引用bufferProducer，如果参数jsurface不等于NULL，通过android_view_Surface_getSurface(env, jsurface)获取一个Surface的强引用surface，如果surface不等于NULL，调用surface->getIGraphicBufferProducer()，把返回值赋值给bufferProducer否则，如果surface等于NULL，抛出IllegalArgumentException异常。定义ICrypto的强引用crypto，如果参数jcrypto不为NULL，调用JCrypto::GetCrypto(env, jcrypto)，把值返回给crypto。这是针对视频加密的情况。最后，通过codec->configure(format, bufferProducer, crypto, flags)实现对MediaCodec的配置，其中，format，bufferProducer，crypto是之前生成的，flags是传入的参数。上边的调用会返回错误信息，throwExceptionAsNecessary(env, err)会根据返回的具体错误信息，决定是否抛出异常，以及抛出何种异常。

- start
下一个阶段是start。
```
static void android_media_MediaCodec_start(JNIEnv *env, jobject thiz) {
1168    ALOGV("android_media_MediaCodec_start");
1169
1170    sp<JMediaCodec> codec = getMediaCodec(env, thiz);
1171
1172    if (codec == NULL) {
1173        throwExceptionAsNecessary(env, INVALID_OPERATION);
1174        return;
1175    }
1176
1177    status_t err = codec->start();
1178
1179    throwExceptionAsNecessary(env, err, ACTION_CODE_FATAL, "start failed");
1180}
```
通过getMediaCodec(env, thiz)获取JMediaCodec的强引用codec。如果codec为NULL，抛出INVALID_OPERATION异常。调用JMediaCodec的start`status_t err = codec->start()`，根据返回的err的状态，决定是否抛出异常，以及抛出何种异常。

- dequeueInputBuffer
下一个状态是dequeueInputBuffer。
```
static jint android_media_MediaCodec_dequeueInputBuffer(
1426        JNIEnv *env, jobject thiz, jlong timeoutUs) {
1427    ALOGV("android_media_MediaCodec_dequeueInputBuffer");
1428
1429    sp<JMediaCodec> codec = getMediaCodec(env, thiz);
1430
1431    if (codec == NULL) {
1432        throwExceptionAsNecessary(env, INVALID_OPERATION);
1433        return -1;
1434    }
1435
1436    size_t index;
1437    status_t err = codec->dequeueInputBuffer(&index, timeoutUs);
1438
1439    if (err == OK) {
1440        return (jint) index;
1441    }
1442
1443    return throwExceptionAsNecessary(env, err);
1444}
```
过程基本和start类似，不再详述。值得注意的是这里会返回input buffer的索引。android_media_MediaCodec_dequeueOutputBuffer方法类似，只是返回output buffer的索引。
- queueInputBuffer
```
static void android_media_MediaCodec_queueInputBuffer(
1235        JNIEnv *env,
1236        jobject thiz,
1237        jint index,
1238        jint offset,
1239        jint size,
1240        jlong timestampUs,
1241        jint flags) {
1242    ALOGV("android_media_MediaCodec_queueInputBuffer");
1243
1244    sp<JMediaCodec> codec = getMediaCodec(env, thiz);
1245
1246    if (codec == NULL) {
1247        throwExceptionAsNecessary(env, INVALID_OPERATION);
1248        return;
1249    }
1250
1251    AString errorDetailMsg;
1252
1253    status_t err = codec->queueInputBuffer(
1254            index, offset, size, timestampUs, flags, &errorDetailMsg);
1255
1256    throwExceptionAsNecessary(
1257            env, err, ACTION_CODE_FATAL, errorDetailMsg.empty() ? NULL : errorDetailMsg.c_str());
1258}
```
这里会的调用JMediaCodec的queueInputBuffer，把缓冲区索引index，偏移offset，size，时间戳timestampUs，flags传入，把错误信息回传到errorDetailMsg。

- flush, stop, reset, release
实际上，这些操作，都是通过JMediaCodec中对应的方法实现。其中，release实际调用setMediaCodec(env, thiz, NULL)。
```
static sp<JMediaCodec> setMediaCodec(
808        JNIEnv *env, jobject thiz, const sp<JMediaCodec> &codec) {
809    sp<JMediaCodec> old = (JMediaCodec *)env->GetLongField(thiz, gFields.context);
810    if (codec != NULL) {
811        codec->incStrong(thiz);
812    }
813    if (old != NULL) {
814        /* release MediaCodec and stop the looper now before decStrong.
815         * otherwise JMediaCodec::~JMediaCodec() could be called from within
816         * its message handler, doing release() from there will deadlock
817         * (as MediaCodec::release() post synchronous message to the same looper)
818         */
819        old->release();
820        old->decStrong(thiz);
821    }
822    env->SetLongField(thiz, gFields.context, (jlong)codec.get());
823
824    return old;
825}
```
setMediaCodec方法中，如果参数codec不等于NULL，需要增加codec的强引用计数。这里release调用传入的参数是NULL，所以不会发生。如果获取的JMediaCodec的强引用old不为NULL，就调用old的release，并且将old的强引用计数减少一。注意，这里release MediaCodec和stop looper必须在减少强引用计数之前调用，否则将会形成死锁。

- JMediaCodec类
JMediaCodec就定义在android_media_MediaCodec.h中。类定义代码我们这里就不拷贝了。接下来我们要重点分析MediaCodec的各个状态转换方法在JMediaCodec中的实现。实际上，JMediaCodec类中的方法，最终又调用了stagefright框架中MediaCodec类中的方法。重点看一下JMediaCodec中的release方法。
```
void JMediaCodec::release() {
192    if (mCodec != NULL) {
193        mCodec->release();
194        mCodec.clear();
195        mInitStatus = NO_INIT;
196    }
197
198    if (mLooper != NULL) {
199        mLooper->unregisterHandler(id());
200        mLooper->stop();
201        mLooper.clear();
202    }
203}
```
这里进行两部操作，一个是调用stagefraight中MediaCodec的release方法，下一步是停止looper。