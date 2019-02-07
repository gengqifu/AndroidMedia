# MediaMuxer.jara源码分析

音视频通过Codec（编码器）编码之后，还需要经过MediaMuxer（混合器）“混合”。混合器在framework的实现就是MediaMuxer类。MediaMuxer中又个内部类OutputFormat：
```
public static final class OutputFormat {
80        /* Do not change these values without updating their counterparts
81         * in include/media/stagefright/MediaMuxer.h!
82         */
83        private OutputFormat() {}
84        /** MPEG4 media file format*/
85        public static final int MUXER_OUTPUT_MPEG_4 = 0;
86        public static final int MUXER_OUTPUT_WEBM   = 1;
87    };
```
从这里看到，目前MediaMuxer支持的输出格式只有mp4和webm两种。Muxer有一些状态，一共有以下几种：
```
private static final int MUXER_STATE_UNINITIALIZED  = -1;
113    private static final int MUXER_STATE_INITIALIZED    = 0;
114    private static final int MUXER_STATE_STARTED        = 1;
115    private static final int MUXER_STATE_STOPPED        = 2;
116
117    private int mState = MUXER_STATE_UNINITIALIZED;
```
MUXER_STATE_UNINITIALIZED是默认状态。MediaMuxer的构造函数中，也会将mState设置为MUXER_STATE_UNINITIALIZED。
类中的方法setOrientationHint(int degrees)会设置输出视频的角度。这个角度并不会改变输入视频帧的角度，只是告诉播放器在播放的时候选择正确的角度。默认的角度是0。类中还有个setLocation方法，会设置和保存拍摄视频时的经纬度。
与生命周期相关的方法有start，stop，release等。
```
public void start() {
227        if (mNativeObject == 0) {
228            throw new IllegalStateException("Muxer has been released!");
229        }
230        if (mState == MUXER_STATE_INITIALIZED) {
231            nativeStart(mNativeObject);
232            mState = MUXER_STATE_STARTED;
233        } else {
234            throw new IllegalStateException("Can't start due to wrong state.");
235        }
236    }
237
238    /**
239     * Stops the muxer.
240     * <p>Once the muxer stops, it can not be restarted.</p>
241     * @throws IllegalStateException if muxer is in the wrong state.
242     */
243    public void stop() {
244        if (mState == MUXER_STATE_STARTED) {
245            nativeStop(mNativeObject);
246            mState = MUXER_STATE_STOPPED;
247        } else {
248            throw new IllegalStateException("Can't stop due to wrong state.");
249        }
250    }
public void release() {
484        if (mState == MUXER_STATE_STARTED) {
485            stop();
486        }
487        if (mNativeObject != 0) {
488            nativeRelease(mNativeObject);
489            mNativeObject = 0;
490            mCloseGuard.close();
491        }
492        mState = MUXER_STATE_UNINITIALIZED;
493    }
```
这些方法最终都需要调用对应的native方法，并且设置分mState为对应的状态。start方法必须在MUXER_STATE_INITIALIZED状态下调用，stop和release方法必须在MUXER_STATE_STARTED状态下调用，否则，都会抛出IllegalStateException异常。release方法，在调用stop方法之后，还需要关闭CloseGuard。
addTrack方法以指定的格式，添加一个track。
```
public int addTrack(@NonNull MediaFormat format) {
392        if (format == null) {
393            throw new IllegalArgumentException("format must not be null.");
394        }
395        if (mState != MUXER_STATE_INITIALIZED) {
396            throw new IllegalStateException("Muxer is not initialized.");
397        }
398        if (mNativeObject == 0) {
399            throw new IllegalStateException("Muxer has been released!");
400        }
401        int trackIndex = -1;
402        // Convert the MediaFormat into key-value pairs and send to the native.
403        Map<String, Object> formatMap = format.getMap();
404
405        String[] keys = null;
406        Object[] values = null;
407        int mapSize = formatMap.size();
408        if (mapSize > 0) {
409            keys = new String[mapSize];
410            values = new Object[mapSize];
411            int i = 0;
412            for (Map.Entry<String, Object> entry : formatMap.entrySet()) {
413                keys[i] = entry.getKey();
414                values[i] = entry.getValue();
415                ++i;
416            }
417            trackIndex = nativeAddTrack(mNativeObject, keys, values);
418        } else {
419            throw new IllegalArgumentException("format must not be empty.");
420        }
421
422        // Track index number is expected to incremented as addTrack succeed.
423        // However, if format is invalid, it will get a negative trackIndex.
424        if (mLastTrackIndex >= trackIndex) {
425            throw new IllegalArgumentException("Invalid format.");
426        }
427        mLastTrackIndex = trackIndex;
428        return trackIndex;
429    }
```
参数format不能为null，否则会抛出IllegalArgumentException异常。而且addTrack方法必须在MUXER_STATE_INITIALIZED状态下调用，也就是在start之前，否则，也会抛出IllegalStateException异常。如果mNativeObject等于0，同样会抛出IllegalStateException异常，说明Muxer已经被释放了。从参数format分解出format map，最后调用nativeAddTrack(mNativeObject, keys, values)，发送到native，并且返回track index。如果format没有成功分解出format map，抛出IllegalArgumentException异常。更新mLastTrackIndex为之前返回的track index。
通过方法writeSampleData把已经编码的音视频采样数据写入muxer。
```
public void writeSampleData(int trackIndex, @NonNull ByteBuffer byteBuf,
446            @NonNull BufferInfo bufferInfo) {
447        if (trackIndex < 0 || trackIndex > mLastTrackIndex) {
448            throw new IllegalArgumentException("trackIndex is invalid");
449        }
450
451        if (byteBuf == null) {
452            throw new IllegalArgumentException("byteBuffer must not be null");
453        }
454
455        if (bufferInfo == null) {
456            throw new IllegalArgumentException("bufferInfo must not be null");
457        }
458        if (bufferInfo.size < 0 || bufferInfo.offset < 0
459                || (bufferInfo.offset + bufferInfo.size) > byteBuf.capacity()
460                || bufferInfo.presentationTimeUs < 0) {
461            throw new IllegalArgumentException("bufferInfo must specify a" +
462                    " valid buffer offset, size and presentation time");
463        }
464
465        if (mNativeObject == 0) {
466            throw new IllegalStateException("Muxer has been released!");
467        }
468
469        if (mState != MUXER_STATE_STARTED) {
470            throw new IllegalStateException("Can't write, muxer is not started");
471        }
472
473        nativeWriteSampleData(mNativeObject, trackIndex, byteBuf,
474                bufferInfo.offset, bufferInfo.size,
475                bufferInfo.presentationTimeUs, bufferInfo.flags);
476    }
```
参数trackIndex不能小于0，也不能大于mLastTrackIndex，byteBuffer和bufferInfo不能为null，bufferInfo的size不能小于0，bufferInfo的offset不能小于0，offset加上size不能大于byteBuffer的capacity，presentationTimeUs不能小于0，mNativeObject不能等于0，状态不能是MUXER_STATE_STARTED，否则，都会抛出IllegalArgumentException。最后，调用native方法nativeWriteSampleData输出sample data。