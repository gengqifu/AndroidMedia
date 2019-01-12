## 概述
MediaCodec勇于访问底层的编解码器，是Android底层多媒体支持框架的一部分。它可以操作三种数据：压缩后的，原始的音频数据，以及原始的视频数据。
MediaCodec以异步的方式处理数据，会用到一组输入和输出缓冲区，后文会详细介绍。MediaCodec有内部类BufferInfo来描述缓冲区。
BufferInfo的代码如下：
```
public final static class BufferInfo {
        /**
         * Update the buffer metadata information.
         *
         * @param newOffset the start-offset of the data in the buffer.
         * @param newSize   the amount of data (in bytes) in the buffer.
         * @param newTimeUs the presentation timestamp in microseconds.
         * @param newFlags  buffer flags associated with the buffer.  This
         * should be a combination of  {@link #BUFFER_FLAG_KEY_FRAME} and
         * {@link #BUFFER_FLAG_END_OF_STREAM}.
         */
        public void set(
                int newOffset, int newSize, long newTimeUs, @BufferFlag int newFlags) {
            offset = newOffset;
            size = newSize;
            presentationTimeUs = newTimeUs;
            flags = newFlags;
        }
        /**
         * The start-offset of the data in the buffer.
         */
        public int offset;
        /**
         * The amount of data (in bytes) in the buffer.  If this is {@code 0},
         * the buffer has no data in it and can be discarded.  The only
         * use of a 0-size buffer is to carry the end-of-stream marker.
         */
        public int size;
        /**
         * The presentation timestamp in microseconds for the buffer.
         * This is derived from the presentation timestamp passed in
         * with the corresponding input buffer.  This should be ignored for
         * a 0-sized buffer.
         */
        public long presentationTimeUs;
        /**
         * Buffer flags associated with the buffer.  A combination of
         * {@link #BUFFER_FLAG_KEY_FRAME} and {@link #BUFFER_FLAG_END_OF_STREAM}.
         *
         * <p>Encoded buffers that are key frames are marked with
         * {@link #BUFFER_FLAG_KEY_FRAME}.
         *
         * <p>The last output buffer corresponding to the input buffer
         * marked with {@link #BUFFER_FLAG_END_OF_STREAM} will also be marked
         * with {@link #BUFFER_FLAG_END_OF_STREAM}. In some cases this could
         * be an empty buffer, whose sole purpose is to carry the end-of-stream
         * marker.
         */
        @BufferFlag
        public int flags;
        /** @hide */
        @NonNull
        public BufferInfo dup() {
            BufferInfo copy = new BufferInfo();
            copy.set(offset, size, presentationTimeUs, flags);
            return copy;
        }
    };
```
属性offset用来描述数据在缓冲区中的起始偏移量；属性size用来描述数据在缓冲区的大小（以字节为单位）。如果size为0的话，说明缓冲区中没有数据，可以被丢弃。唯一使用size为0的buffer的情况是，携带end-of-stream（流结束）的标记。presentationTimeUs用来表示缓冲区中的presentation timestamp（以毫为单位）。这源自于相对应的输入缓冲区中的presentation timestamp。对于size为0的缓冲区，这个值应该被忽略。flag是用来表示一个和缓冲区相关联的标记。一个BUFFER_FLAG_KEY_FRAME和BUFFER_FLAG_END_OF_STREAM的组合。关键帧将会被标记为BUFFER_FLAG_KEY_FRAME。与输入缓冲区被标记为BUFFER_FLAG_END_OF_STREAM相对应的最后一个输出缓冲区也会被标记为BUFFER_FLAG_END_OF_STREAM。在某些情况下，这可能是一个空缓冲区，它唯一的作用就是用来携带一个end-of-stream标记。
set(int newOffset, int newSize, long newTimeUs, @BufferFlag int newFlags)方法用来设置以上几个属性。
dup()方法用来返回当前BufferInfo的一个拷贝。

MediaCodec类中的一些常量定义如下：
```
/**
     * This indicates that the (encoded) buffer marked as such contains
     * the data for a key frame.
     *
     * @deprecated Use {@link #BUFFER_FLAG_KEY_FRAME} instead.
     */
    public static final int BUFFER_FLAG_SYNC_FRAME = 1;
    /**
     * This indicates that the (encoded) buffer marked as such contains
     * the data for a key frame.
     */
    public static final int BUFFER_FLAG_KEY_FRAME = 1;
    /**
     * This indicated that the buffer marked as such contains codec
     * initialization / codec specific data instead of media data.
     */
    public static final int BUFFER_FLAG_CODEC_CONFIG = 2;
    /**
     * This signals the end of stream, i.e. no buffers will be available
     * after this, unless of course, {@link #flush} follows.
     */
    public static final int BUFFER_FLAG_END_OF_STREAM = 4;
    /**
     * This indicates that the buffer only contains part of a frame,
     * and the decoder should batch the data until a buffer without
     * this flag appears before decoding the frame.
     */
    public static final int BUFFER_FLAG_PARTIAL_FRAME = 8;
    /**
     * This indicates that the buffer contains non-media data for the
     * muxer to process.
     *
     * All muxer data should start with a FOURCC header that determines the type of data.
     *
     * For example, when it contains Exif data sent to a MediaMuxer track of
     * {@link MediaFormat#MIMETYPE_IMAGE_ANDROID_HEIC} type, the data must start with
     * Exif header ("Exif\0\0"), followed by the TIFF header (See JEITA CP-3451C Section 4.5.2.)
     *
     * @hide
     */
    public static final int BUFFER_FLAG_MUXER_DATA = 16;
    ```
    BUFFER_FLAG_SYNC_FRAME用来标记当前是一个关键帧。这个常量已经过时了，要用新的BUFFER_FLAG_KEY_FRAME。
    BUFFER_FLAG_CODEC_CONFIG表示这是一个编码初始化或者特定编码信息帧，而不是媒体数据。
    BUFFER_FLAG_END_OF_STREAM用来标记流的结束，在这个标记之后缓冲区将不再可用。
    BUFFER_FLAG_PARTIAL_FRAME用来标记当前缓冲区仅包含部分帧数据，解码器需要将知道不再出现这个flag为止的一组数据打包为一帧。
    BUFFER_FLAG_MUXER_DATA用来标记缓冲区中的是non-media数据，供muxer用来处理。所有的muxer数据都以FOURCC头开始，用来决定数据的类型。

    属性mEventHandler，mOnFrameRenderedHandler，mCallbackHandler的类型都是EventHandler，EventHandler类的定义如下：
    ```
    private class EventHandler extends Handler {
        private MediaCodec mCodec;
        public EventHandler(@NonNull MediaCodec codec, @NonNull Looper looper) {
            super(looper);
            mCodec = codec;
        }
        @Override
        public void handleMessage(@NonNull Message msg) {
            switch (msg.what) {
                case EVENT_CALLBACK:
                {
                    handleCallback(msg);
                    break;
                }
                case EVENT_SET_CALLBACK:
                {
                    mCallback = (MediaCodec.Callback) msg.obj;
                    break;
                }
                case EVENT_FRAME_RENDERED:
                    synchronized (mListenerLock) {
                        Map<String, Object> map = (Map<String, Object>)msg.obj;
                        for (int i = 0; ; ++i) {
                            Object mediaTimeUs = map.get(i + "-media-time-us");
                            Object systemNano = map.get(i + "-system-nano");
                            if (mediaTimeUs == null || systemNano == null
                                    || mOnFrameRenderedListener == null) {
                                break;
                            }
                            mOnFrameRenderedListener.onFrameRendered(
                                    mCodec, (long)mediaTimeUs, (long)systemNano);
                        }
                        break;
                    }
                default:
                {
                    break;
                }
            }
        }
        private void handleCallback(@NonNull Message msg) {
            if (mCallback == null) {
                return;
            }
            switch (msg.arg1) {
                case CB_INPUT_AVAILABLE:
                {
                    int index = msg.arg2;
                    synchronized(mBufferLock) {
                        validateInputByteBuffer(mCachedInputBuffers, index);
                    }
                    mCallback.onInputBufferAvailable(mCodec, index);
                    break;
                }
                case CB_OUTPUT_AVAILABLE:
                {
                    int index = msg.arg2;
                    BufferInfo info = (MediaCodec.BufferInfo) msg.obj;
                    synchronized(mBufferLock) {
                        validateOutputByteBuffer(mCachedOutputBuffers, index, info);
                    }
                    mCallback.onOutputBufferAvailable(
                            mCodec, index, info);
                    break;
                }
                case CB_ERROR:
                {
                    mCallback.onError(mCodec, (MediaCodec.CodecException) msg.obj);
                    break;
                }
                case CB_OUTPUT_FORMAT_CHANGE:
                {
                    mCallback.onOutputFormatChanged(mCodec,
                            new MediaFormat((Map<String, Object>) msg.obj));
                    break;
                }
                default:
                {
                    break;
                }
            }
        }
    }
    ```
    EventHandler扩展自Handler。它有一个私有属性mCodec就是MediaCodec的实例，EventHandler的构造函数将会设置mCodec以及EventHandler的Looper。EventHandler的handleMessage会处理EVENT_CALLBACK，EVENT_SET_CALLBACK，EVENT_FRAME_RENDERED三种消息。但我们在MediaCodec类中却找不到EVENT_CALLBACK和EVENT_FRAME_RENDERED消息发出的地方。不过，我们可以看到有个方法：
    ```
    private void postEventFromNative(
            int what, int arg1, int arg2, @Nullable Object obj) {
        synchronized (mListenerLock) {
            EventHandler handler = mEventHandler;
            if (what == EVENT_CALLBACK) {
                handler = mCallbackHandler;
            } else if (what == EVENT_FRAME_RENDERED) {
                handler = mOnFrameRenderedHandler;
            }
            if (handler != null) {
                Message msg = handler.obtainMessage(what, arg1, arg2, obj);
                handler.sendMessage(msg);
            }
        }
    }
    ```
    从方法名就可以看出，该方法将会投递出来自native层的event，实际上是对native层event的再投递。postEventFromNative方法会在native层注册，并且从native层触发。我们后续分析native层代码时会详细讨论。
    我们先看从本类中发出的消息EVENT_SET_CALLBACK。在handleMessage()方法中看到，在收到EVENT_SET_CALLBACK消息时，会用msg中携带的obj给属性mCallback赋值。mCallback是内部类Callback的一个实例。Callback是一个抽象类，它的定义如下：
    ```
    public static abstract class Callback {
        /**
         * Called when an input buffer becomes available.
         *
         * @param codec The MediaCodec object.
         * @param index The index of the available input buffer.
         */
        public abstract void onInputBufferAvailable(@NonNull MediaCodec codec, int index);
        /**
         * Called when an output buffer becomes available.
         *
         * @param codec The MediaCodec object.
         * @param index The index of the available output buffer.
         * @param info Info regarding the available output buffer {@link MediaCodec.BufferInfo}.
         */
        public abstract void onOutputBufferAvailable(
                @NonNull MediaCodec codec, int index, @NonNull BufferInfo info);
        /**
         * Called when the MediaCodec encountered an error
         *
         * @param codec The MediaCodec object.
         * @param e The {@link MediaCodec.CodecException} object describing the error.
         */
        public abstract void onError(@NonNull MediaCodec codec, @NonNull CodecException e);
        /**
         * Called when the output format has changed
         *
         * @param codec The MediaCodec object.
         * @param format The new output format.
         */
        public abstract void onOutputFormatChanged(
                @NonNull MediaCodec codec, @NonNull MediaFormat format);
    }
```
Callback的作用就是异步地向用户发送各种MediaCodec事件。当input buffer可用时，将会触发onInputBufferAvailable，它的参数是一个MediaCodec的对象，以及可用的缓冲区的索引；当输出缓冲区可用时，将会触发onOutputBufferAvailable，它的参数是一个MediaCodec的对象，一个可用输出缓冲区的索引，以及一个包含可用输出缓冲区信息的BufferInfo对象；当MediaCodec发生错误时，将会触发onError方法，它的参数是一个MediaCodec的对象，以及一个CodecException对象；当输出分缓冲区故事发生变化时，会触发onOutputFormatChanged, 它有两个参数，一个MediaCodec对象，以及一个MediaFormat对象。**MediaFormat**类的源码将在单独的章节剖析。

回到上文提到的EVENT_SET_CALLBACK消息，它是从本类内发出的，来自方法setCallback，它的代码如下：
```
public void setCallback(@Nullable /* MediaCodec. */ Callback cb, @Nullable Handler handler) {
    if (cb != null) {
        synchronized (mListenerLock) {
            EventHandler newHandler = getEventHandlerOn(handler, mCallbackHandler);
            // NOTE: there are no callbacks on the handler at this time, but check anyways
            // even if we were to extend this to be callable dynamically, it must
            // be called when codec is flushed, so no messages are pending.
            if (newHandler != mCallbackHandler) {
                mCallbackHandler.removeMessages(EVENT_SET_CALLBACK);
                mCallbackHandler.removeMessages(EVENT_CALLBACK);
                mCallbackHandler = newHandler;
            }
        }
    } else if (mCallbackHandler != null) {
        mCallbackHandler.removeMessages(EVENT_SET_CALLBACK);
        mCallbackHandler.removeMessages(EVENT_CALLBACK);
    }
    if (mCallbackHandler != null) {
        // set java callback on main handler
        Message msg = mCallbackHandler.obtainMessage(EVENT_SET_CALLBACK, 0, 0, cb);
        mCallbackHandler.sendMessage(msg);
        // set native handler here, don't post to handler because
        // it may cause the callback to be delayed and set in a wrong state.
        // Note that native codec may start sending events to the callback
        // handler after this returns.
        native_setCallback(cb);
    }
}
```
如果参数cb不为null，它先清除消息队列里的EVENT_SET_CALLBACK消息和EVENT_CALLBACK消息，通过getEventHandlerOn获取一个新的EventHandler newHandler,然后将newHandler赋值给mCallbackHandler；如果参数cb为null的话，则只移除EVENT_SET_CALLBACK和EVENT_CALLBACK消息。接下来，如果mCallbackHandler不为null，就生成一个EVENT_SET_CALLBACK的Message，并且发送到mCallbackHandler。最后，调用native_setCallback设置native层的handler。当这个方法return之后，native的编码器将会开始向callback handler发送事件。具体细节我们将在分析native层代码时讨论。

接下来我们看一下，当收到EVENT_CALLBACK事件时，将会发生什么。如前面分析的，EVENT_CALLBACK事件是由native层发送的，在EventHandler源码中看到，当收到EVENT_CALLBACK事件时，会调用EventHandler内部的handleCallback方法。该方法会根据msg.arg1来决定具体的行为。如果msg.arg1等于CB_INPUT_AVAILABLE，将会首先调用方法validateInputByteBuffer，然后触发onInputBufferAvailable回调，而输入缓冲区和输出缓冲区的index，则由msg.arg2指定；如果msg.arg1等于CB_OUTPUT_AVAILABLE,则分别从msg.arg2和msg.obj中获取bufer index和buffer info，然后调用validateOutputByteBuffer，最后，触发onOutputBufferAvailable回调；如果msg.arg1等于CB_ERROR，那么就触发onError回调；如果msg.arg1等于CB_OUTPUT_FORMAT_CHANGE，就触发onOutputFormatChanged回调。
上文提到的validateInputByteBuffer和validateOutputByteBuffer的代码如下：
```
private final void validateInputByteBuffer(
            @Nullable ByteBuffer[] buffers, int index) {
    if (buffers != null && index >= 0 && index < buffers.length) {
        ByteBuffer buffer = buffers[index];
        if (buffer != null) {
            buffer.setAccessible(true);
            buffer.clear();
        }
    }
}

private final void validateOutputByteBuffer(
            @Nullable ByteBuffer[] buffers, int index, @NonNull BufferInfo info) {
    if (buffers != null && index >= 0 && index < buffers.length) {
        ByteBuffer buffer = buffers[index];
        if (buffer != null) {
            buffer.setAccessible(true);
            buffer.limit(info.offset + info.size).position(info.offset);
        }
    }
}
```
validateInputByteBuffer首先判断传入的ByteBuffer数组buffers是否为null，传入的buffer index是否>=0，并且要小于buffers的长度。在满足以上条件的前提下，获取buffers的第index条数据buffer，如果buffer不为null，调用buffer.setAccessible(true)和buffer.clear()使buffer可读。validateOutputByteBuffer首先判断传入的ByteBuffer数组buffers是否为null，传入的buffer index是否>=0，并且要小于buffers的长度。在满足以上条件的前提下，获取buffers第index条数据buffer，如果buffer不为null，调用buffer.setAccessible(true)和buffer.limit(info.offset + info.size).position(info.offset)使buffer可访问，并且重新设置buffer的limit和position。
上文提到的validateInputByteBuffer和validateOutputByteBuffer调用处，分别要传入一个ByteBuffer数组，分别是mCachedInputBuffers和mCachedOutputBuffers。对这两个属性的操作，有一组函数。除了上文提到的两个，还有：
```
private final void invalidateByteBuffer(
            @Nullable ByteBuffer[] buffers, int index) {
    if (buffers != null && index >= 0 && index < buffers.length) {
        ByteBuffer buffer = buffers[index];
        if (buffer != null) {
            buffer.setAccessible(false);
        }
    }
}

private final void revalidateByteBuffer(
            @Nullable ByteBuffer[] buffers, int index) {
    synchronized(mBufferLock) {
        if (buffers != null && index >= 0 && index < buffers.length) {
            ByteBuffer buffer = buffers[index];
            if (buffer != null) {
                buffer.setAccessible(true);
            }
        }
    }
}
```
invalidateByteBuffer会使缓冲区不可访问，而revalidateByteBuffer则使缓冲区可以访问。
```
private final void invalidateByteBuffers(@Nullable ByteBuffer[] buffers) {
    if (buffers != null) {
        for (ByteBuffer buffer: buffers) {
            if (buffer != null) {
                buffer.setAccessible(false);
            }
        }
    }
}
```
invalidateByteBuffers会使一组ByteBuffer不可访问。
```
private final void freeByteBuffer(@Nullable ByteBuffer buffer) {
    if (buffer != null /* && buffer.isDirect() */) {
        // all of our ByteBuffers are direct
        java.nio.NioUtils.freeDirectBuffer(buffer);
    }
}
private final void freeByteBuffers(@Nullable ByteBuffer[] buffers) {
    if (buffers != null) {
        for (ByteBuffer buffer: buffers) {
            freeByteBuffer(buffer);
        }
    }
}
private final void freeAllTrackedBuffers() {
    synchronized(mBufferLock) {
        freeByteBuffers(mCachedInputBuffers);
        freeByteBuffers(mCachedOutputBuffers);
        mCachedInputBuffers = null;
        mCachedOutputBuffers = null;
        mDequeuedInputBuffers.clear();
        mDequeuedOutputBuffers.clear();
    }
}
```
feeeByteBuffer调用java.nio.NioUtils.freeDirectBuffer释放buffer空间；freeByteBuffers通过逐一调用freeByteBuffer释放一组buffer的空间。freeAllTrackedBuffers通过调用freeByteBuffers释放mCachedInputBuffers和mCachedOutputBuffers的空间，并且将两者置空。同时，清空mDequeuedInputBuffers和mDequeuedOutputBuffers。
```
private final void cacheBuffers(boolean input) {
    ByteBuffer[] buffers = null;
    try {
        buffers = getBuffers(input);
        invalidateByteBuffers(buffers);
    } catch (IllegalStateException e) {
        // we don't get buffers in async mode
    }
    if (input) {
        mCachedInputBuffers = buffers;
    } else {
        mCachedOutputBuffers = buffers;
    }
}
```
cacheBuffers调用native方法getBuffers()获取一个缓冲区，输入还是输出缓冲区由参数input指定。然后调用invalidateByteBuffers，然后，根据input的值，决定将buffers赋值给mCachedInputBuffers还是mCachedOutputBuffers。
```
public ByteBuffer[] getInputBuffers() {
    if (mCachedInputBuffers == null) {
        throw new IllegalStateException();
    }
    // FIXME: check codec status
    return mCachedInputBuffers;
}
public ByteBuffer[] getOutputBuffers() {
    if (mCachedOutputBuffers == null) {
        throw new IllegalStateException();
    }
    // FIXME: check codec status
    return mCachedOutputBuffers;
}
public ByteBuffer getInputBuffer(int index) {
    ByteBuffer newBuffer = getBuffer(true /* input */, index);
    synchronized(mBufferLock) {
        invalidateByteBuffer(mCachedInputBuffers, index);
        mDequeuedInputBuffers.put(index, newBuffer);
    }
    return newBuffer;
}
public Image getInputImage(int index) {
    Image newImage = getImage(true /* input */, index);
    synchronized(mBufferLock) {
        invalidateByteBuffer(mCachedInputBuffers, index);
        mDequeuedInputBuffers.put(index, newImage);
    }
    return newImage;
}
public ByteBuffer getOutputBuffer(int index) {
    ByteBuffer newBuffer = getBuffer(false /* input */, index);
    synchronized(mBufferLock) {
        invalidateByteBuffer(mCachedOutputBuffers, index);
        mDequeuedOutputBuffers.put(index, newBuffer);
    }
    return newBuffer;
}
public Image getOutputImage(int index) {
    Image newImage = getImage(false /* input */, index);
    synchronized(mBufferLock) {
        invalidateByteBuffer(mCachedOutputBuffers, index);
        mDequeuedOutputBuffers.put(index, newImage);
    }
    return newImage;
}
```
getInputBuffers返回一组输入缓冲区。该方法要在start返回之后调用。调用这个方法之后，就不能再使用之前调用这个方法返回的缓冲区。这个方法已经过时了，应该使用getInputBuffer，每次获取一个输入缓冲区。从API 21开始，移除输入缓冲区是自动进行的。如果使用了input surface，不要使用这个方法。getOutputBuffers（）返回一组输出缓冲区。这个方法要在start返回之后，或者dequeueOutputBuffer触发了INFO_OUTPUT_BUFFERS_CHANGED之后调用。调用这个方法之后，之前调用这个方法返回的缓冲区都不能再使用。这个方法过时了，用getOutputBuffer方法每次获取一个输出缓冲区。从API 21开始，移除的输出缓冲区的limit和position，将会被设置为有效的数据范围。如果正在使用output surface，就不要使用这个方法。getInputImage与getInputBuffer类似，只是返回一个视频帧的Image，Image由native方法getImage返回。getOutputImage和getOutputBuffer类似，返回一个只读的Image对象，该Image来自一个离队的输出缓冲区的原始视频帧。

接下来我们来分析获取MediaCodec的一组方法。
```
public static MediaCodec createDecoderByType(@NonNull String type)
        throws IOException {
    return new MediaCodec(type, true /* nameIsType */, false /* encoder */);
}
public static MediaCodec createEncoderByType(@NonNull String type)
        throws IOException {
    return new MediaCodec(type, true /* nameIsType */, true /* encoder */);
}
public static MediaCodec createByCodecName(@NonNull String name)
        throws IOException {
    return new MediaCodec(
            name, false /* nameIsType */, false /* unused */);
}
private MediaCodec(
        @NonNull String name, boolean nameIsType, boolean encoder) {
    Looper looper;
    if ((looper = Looper.myLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else if ((looper = Looper.getMainLooper()) != null) {
        mEventHandler = new EventHandler(this, looper);
    } else {
        mEventHandler = null;
    }
    mCallbackHandler = mEventHandler;
    mOnFrameRenderedHandler = mEventHandler;
    mBufferLock = new Object();
    native_setup(name, nameIsType, encoder);
}
```
createDecoderByType根据mime type获取MediaCodec的decoder实例，参数是一个mime type的String。createEncoderByType根据mime type生成一个MediaCodec的encoder实例。createByCodecName根据精确的名字来生成MediaCodec，前提是用户要知道精确的名字。MediaCodec的构造函数，有三个参数，第一个参数name，可能是type，也可能是精确的名字，取决于第二个参数nameIsType，第三个参数用来表明是编码器还是解码器。构造函数首先尝试获取当前线程的Looper，如果成功获取，就以当前MediaCodec和获取的looper创建一个EventHandler；如果没有成功获取当前线程Looper，就尝试获取Main Looper，如果成功获取，就以Main Looper为参数创建一个EventHandler；如果以上两次尝试都失败，将mEventHandler赋值为null。之后，将mEventHandler赋值给mCallbackHandler和mOnFrameRenderedHandler，为mBufferLock分配一个新的对象，最后，调用native_setup。

接下来，我们来分析MediaCodec状态相关函数。
```
public void configure(
        @Nullable MediaFormat format,
        @Nullable Surface surface, @Nullable MediaCrypto crypto,
        @ConfigureFlag int flags) {
    configure(format, surface, crypto, null, flags);
}
public void configure(
        @Nullable MediaFormat format, @Nullable Surface surface,
        @ConfigureFlag int flags, @Nullable MediaDescrambler descrambler) {
    configure(format, surface, null,
            descrambler != null ? descrambler.getBinder() : null, flags);
}
private void configure(
        @Nullable MediaFormat format, @Nullable Surface surface,
        @Nullable MediaCrypto crypto, @Nullable IHwBinder descramblerBinder,
        @ConfigureFlag int flags) {
    if (crypto != null && descramblerBinder != null) {
        throw new IllegalArgumentException("Can't use crypto and descrambler together!");
    }
    String[] keys = null;
    Object[] values = null;
    if (format != null) {
        Map<String, Object> formatMap = format.getMap();
        keys = new String[formatMap.size()];
        values = new Object[formatMap.size()];
        int i = 0;
        for (Map.Entry<String, Object> entry: formatMap.entrySet()) {
            if (entry.getKey().equals(MediaFormat.KEY_AUDIO_SESSION_ID)) {
                int sessionId = 0;
                try {
                    sessionId = (Integer)entry.getValue();
                }
                catch (Exception e) {
                    throw new IllegalArgumentException("Wrong Session ID Parameter!");
                }
                keys[i] = "audio-hw-sync";
                values[i] = AudioSystem.getAudioHwSyncForSession(sessionId);
            } else {
                keys[i] = entry.getKey();
                values[i] = entry.getValue();
            }
            ++i;
        }
    }
    mHasSurface = surface != null;
    native_configure(keys, values, surface, crypto, descramblerBinder, flags);
}
```
首先要通过configure系列函数对MediaCodec进行配置。configure函数的各个参数如下：MediaFormat类型format，用来表示输入数据（解码器）的格式或输出数据（编码器）期望的格式；Surface类型surface，指定一个surface，用来渲染解码器的输出。flags，int类型，指定CONFIGURE_FLAG_ENCODE来配置组件为编码器；MediaDescrambler类型descrambler，指定一个解码器，来进行媒体数据的安全解码，对于非安全编码，设为null。configure方法首先声明一个String类型的keys数组，以及一个Object类型的values数组，他们的size是formatMap的size。对formatMap进行Entry遍历，如果key等于**MediaFormat.KEY_AUDIO_SESSION_ID**，获取entry的value，赋值给一个int型sessionId，keys[i]的值设为“audio-hw-sync“，values[i]的值设为AudioSystem.getAudioHwSyncForSession(sessionId)；否则，keys[i]的值就设成entry的key值，values[i]就设成entry的value值。如果参数surface不为null，就将参数surface的值赋值给mHasSurface，否则，mHasSurface设成null。最后，调用native_configure。
```
public final void start() {
    native_start();
    synchronized(mBufferLock) {
        cacheBuffers(true /* input */);
        cacheBuffers(false /* input */);
    }
}
```
start方法首先调用native_start方法，之后，通过调用cacheBuffers两次，来分别获取输入和输出缓冲区。start之后，MediaCodec进入Executing状态，executing有三个子状态：flushed，running，和End-of-Stream。start之后，codec马上进入Flushed状态，持有所有的缓冲区。当第一个输入缓冲区被移除，codec进入Running状态，将持续整个生命周期。当讲end-of-stream标记插入输入缓冲区，codec进入End-of-Stream状态。在这个状态，codec不再接受新的输入缓冲区，但将继续生成新的输出缓冲区，知道到达end-of-stream状态。通过调用flush方法，可以重新进入Flused状态。
```
public final void flush() {
    synchronized(mBufferLock) {
        invalidateByteBuffers(mCachedInputBuffers);
        invalidateByteBuffers(mCachedOutputBuffers);
        mDequeuedInputBuffers.clear();
        mDequeuedOutputBuffers.clear();
    }
    native_flush();
}
```
flush首先分别对mCachedInputBuffers和mCachedOutputBuffers调用invalidateByteBuffers，然后在对mDequeuedInputBuffers和mDequeuedOutputBuffers分别clear。最后，调用native_flush。调用stop方法将会使codec回到Uninitialized状态，因此它需要重新被configure。当用完codec并确定不再使用，必须调用release方法释放codec。
```
public final void stop() {
    native_stop();
    freeAllTrackedBuffers();
    synchronized (mListenerLock) {
        if (mCallbackHandler != null) {
            mCallbackHandler.removeMessages(EVENT_SET_CALLBACK);
            mCallbackHandler.removeMessages(EVENT_CALLBACK);
        }
        if (mOnFrameRenderedHandler != null) {
            mOnFrameRenderedHandler.removeMessages(EVENT_FRAME_RENDERED);
        }
    }
}
public final void release() {
    freeAllTrackedBuffers(); // free buffers first
    native_release();
}
```
stop方法首先调用native_stop(),之后调用freeAllTrackedBuffers，然后，如果mCallbackHandler非空，就从mCallbackHandler消息队列中移除EVENT_SET_CALLBACK和EVENT_CALLBACK。如果mOnFrameRenderedHandler非空，就从mOnFrameRenderedHandler消息队列中移除EVENT_FRAME_RENDERED。
release方法，首先调用freeAllTrackedBuffers，然后调用native_release。
在极少数情况下，codec可能遇到错误，进入Error状态。通过调用reset放法，可以使codec重新可用。可以在任意状态调用这个方法，将codec恢复到Uninitialized状态。
```
public final void reset() {
    freeAllTrackedBuffers(); // free buffers first
    native_reset();
}
```
reset首先调用freeAllTrackedBuffers，之后调用native_reset。
