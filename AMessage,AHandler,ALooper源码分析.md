# AMessage，AHandler和ALooper源码分析

AMessage继承自RefBase类,它有AHandler和ALooper的成员mHandler和mLooper，并且ALooper还是它的友元。成员变量mWhat是消息id，成员变量mTarget指明消息的handler（只用于调试）。成员函数setWhat和setTarget分别设置这两个属性。
```
void AMessage::setWhat(uint32_t what) {
68    mWhat = what;
69}
70
71uint32_t AMessage::what() const {
72    return mWhat;
73}
74
75void AMessage::setTarget(const sp<const AHandler> &handler) {
76    if (handler == NULL) {
77        mTarget = 0;
78        mHandler.clear();
79        mLooper.clear();
80    } else {
81        mTarget = handler->id();
82        mHandler = handler->getHandler();
83        mLooper = handler->getLooper();
84    }
85}
```
成员函数what返回mWhat属性。而setTarget方法，如果参数handler等于NULL，就把mTarget生成NULL（0），同时清空mHandler和mLooper队列；否则，mTarget设成handler的id，mHandler和mLooper分别设置成handler中的handler和looper。
AMessage中有一组set函数：
```
	void setInt32(const char *name, int32_t value);
98    void setInt64(const char *name, int64_t value);
99    void setSize(const char *name, size_t value);
100    void setFloat(const char *name, float value);
101    void setDouble(const char *name, double value);
102    void setPointer(const char *name, void *value);
103    void setString(const char *name, const char *s, ssize_t len = -1);
104    void setString(const char *name, const AString &s);
105    void setObject(const char *name, const sp<RefBase> &obj);
106    void setBuffer(const char *name, const sp<ABuffer> &buffer);
107    void setMessage(const char *name, const sp<AMessage> &obj);
108
109    void setRect(
110            const char *name,
111            int32_t left, int32_t top, int32_t right, int32_t bottom);
```
用来设置name对应的值。与之对应的又一组find函数：
```
bool findInt32(const char *name, int32_t *value) const;
116    bool findInt64(const char *name, int64_t *value) const;
117    bool findSize(const char *name, size_t *value) const;
118    bool findFloat(const char *name, float *value) const;
119    bool findDouble(const char *name, double *value) const;
120    bool findPointer(const char *name, void **value) const;
121    bool findString(const char *name, AString *value) const;
122    bool findObject(const char *name, sp<RefBase> *obj) const;
123    bool findBuffer(const char *name, sp<ABuffer> *buffer) const;
124    bool findMessage(const char *name, sp<AMessage> *obj) const;
125
126    // finds any numeric type cast to a float
127    bool findAsFloat(const char *name, float *value) const;
128
129    bool findRect(
130            const char *name,
131            int32_t *left, int32_t *top, int32_t *right, int32_t *bottom) const;
```
用来根据name查找对应的值。
这里我们重点分析一下setMessage和findMessage的实现。首先看setMessage的代码：
```
void AMessage::setMessage(const char *name, const sp<AMessage> &obj) {
    Item *item = allocateItem(name);
    item->mType = kTypeMessage;

    if (obj != NULL) { obj->incStrong(this); }
    item->u.refValue = obj.get();
}
```
首先调用allocateItem，根据name创建一个Item，item的type设置成kTypeMessage。如果参数obj不等于NULL，就增加obj的强引用计数，同时，item中的refValue要设置成参数obj的值。kTypeMessage是一个枚举值，它的定义在AMessage.h中：
```
enum Type {
167        kTypeInt32,
168        kTypeInt64,
169        kTypeSize,
170        kTypeFloat,
171        kTypeDouble,
172        kTypePointer,
173        kTypeString,
174        kTypeObject,
175        kTypeMessage,
176        kTypeRect,
177        kTypeBuffer,
178    };
```
这里用到的allocateItem函数的代码是这样的：
```
AMessage::Item *AMessage::allocateItem(const char *name) {
187    size_t len = strlen(name);
188    size_t i = findItemIndex(name, len);
189    Item *item;
190
191    if (i < mNumItems) {
192        item = &mItems[i];
193        freeItemValue(item);
194    } else {
195        CHECK(mNumItems < kMaxNumItems);
196        i = mNumItems++;
197        item = &mItems[i];
198        item->setName(name, len);
199    }
200
201    return item;
202}
```
获取name的长度，调用findItemIndex获取那么的index i。如果i小于mNumItems，使item指向mItems的第i个item，然后调用freeItemValue，释放对应的item。如果i大于或等于mNumItems，检查mNumItems是否小于kMaxNumItems。mNumItems的值赋值给i，然后使mNumItems的值加1，item指向mItems的第i个item，设置item的name为参数name。findItemIndex的代码如下：
```
inline size_t AMessage::findItemIndex(const char *name, size_t len) const {
151#ifdef DUMP_STATS
152    size_t memchecks = 0;
153#endif
154    size_t i = 0;
155    for (; i < mNumItems; i++) {
156        if (len != mItems[i].mNameLength) {
157            continue;
158        }
159#ifdef DUMP_STATS
160        ++memchecks;
161#endif
162        if (!memcmp(mItems[i].mName, name, len)) {
163            break;
164        }
165    }
166#ifdef DUMP_STATS
167    {
168        Mutex::Autolock _l(gLock);
169        ++gFindItemCalls;
170        gAverageNumItems += mNumItems;
171        gAverageNumMemChecks += memchecks;
172        gAverageNumChecks += i;
173        reportStats();
174    }
175#endif
176    return i;
177}
```
这是一个内联函数。

· freeItemValue
freeItemValue的代码如下：
```
void AMessage::freeItemValue(Item *item) {
98    switch (item->mType) {
99        case kTypeString:
100        {
101            delete item->u.stringValue;
102            break;
103        }
104
105        case kTypeObject:
106        case kTypeMessage:
107        case kTypeBuffer:
108        {
109            if (item->u.refValue != NULL) {
110                item->u.refValue->decStrong(this);
111            }
112            break;
113        }
114
115        default:
116            break;
117    }
118}
119
```
根据item的类型，执行不同的free操作。

· findMessage
下面我们再来分析一下findMessage的代码：
```
bool AMessage::findMessage(const char *name, sp<AMessage> *obj) const {
350    const Item *item = findItem(name, kTypeMessage);
351    if (item) {
352        *obj = static_cast<AMessage *>(item->u.refValue);
353        return true;
354    }
355    return false;
356}
```
比较简单，调用findItem，根据那么，和类型“kTypeMessage”，调用findItem获取item，如果item不为NULL，就把item的refValue转化为AMessage类型的指针，返回true；否则，返回false。
AMessage中，我们最关心的是以下方法：postAndAwaitResponse等。
```
status_t AMessage::postAndAwaitResponse(sp<AMessage> *response) {
396    sp<ALooper> looper = mLooper.promote();
397    if (looper == NULL) {
398        ALOGW("failed to post message as target looper for handler %d is gone.", mTarget);
399        return -ENOENT;
400    }
401
402    sp<AReplyToken> token = looper->createReplyToken();
403    if (token == NULL) {
404        ALOGE("failed to create reply token");
405        return -ENOMEM;
406    }
407    setObject("replyID", token);
408
409    looper->post(this, 0 /* delayUs */);
410    return looper->awaitResponse(token, response);
411}
412
413status_t AMessage::postReply(const sp<AReplyToken> &replyToken) {
414    if (replyToken == NULL) {
415        ALOGW("failed to post reply to a NULL token");
416        return -ENOENT;
417    }
418    sp<ALooper> looper = replyToken->getLooper();
419    if (looper == NULL) {
420        ALOGW("failed to post reply as target looper is gone.");
421        return -ENOENT;
422    }
423    return looper->postReply(replyToken, this);
424}
```
postAndAwaitResponse方法中，首先通过mLooper的promote方法，获取ALooper的一个强引用计数looper,如果looper为NULL，直接返回ENOMEM。通过looper->createReplyToken()创建AReplyToken的强引用计数token。调用`setObject("replyID", token)`设置replyID。调用looper->post(this, 0 /* delayUs */)发送消息，然后，looper->awaitResponse(token, response)等待返回。

## AHandler类
AHandler类的代码比较简单，其中定义了一个ALooper的弱引用计数mLooper，一个ALooper::handler_id类型的mID。其中的deliverMessage方法代码如下：
```
void AHandler::deliverMessage(const sp<AMessage> &msg) {
27    onMessageReceived(msg);
28    mMessageCounter++;
29
30    if (mVerboseStats) {
31        uint32_t what = msg->what();
32        ssize_t idx = mMessages.indexOfKey(what);
33        if (idx < 0) {
34            mMessages.add(what, 1);
35        } else {
36            mMessages.editValueAt(idx)++;
37        }
38    }
39}
```
首先触发本类的onMessageReceived(msg)，这是一个纯虚函数，子类中需要有它的具体实现。然后把Message技术mMessageCounter加1。获取message的类型（what）和idx。如果idx小于0，在mMessages中添加一个what类型的消息；否则，编辑idx出的Message。

## ALooper类
ALooper类也是RefBase的子类。ALooper中有一个内部的struct LooperThread，继承自Thread，并且，这个struct中有一个指向ALooper类型的指针mLooper。ALooper中的成员变量mThread是一个LooperThread的强引用计数。ALooper的start方法开启Looper，其代码如下：
```
status_t ALooper::start(
97        bool runOnCallingThread, bool canCallJava, int32_t priority) {
98    if (runOnCallingThread) {
99        {
100            Mutex::Autolock autoLock(mLock);
101
102            if (mThread != NULL || mRunningLocally) {
103                return INVALID_OPERATION;
104            }
105
106            mRunningLocally = true;
107        }
108
109        do {
110        } while (loop());
111
112        return OK;
113    }
114
115    Mutex::Autolock autoLock(mLock);
116
117    if (mThread != NULL || mRunningLocally) {
118        return INVALID_OPERATION;
119    }
120
121    mThread = new LooperThread(this, canCallJava);
122
123    status_t err = mThread->run(
124            mName.empty() ? "ALooper" : mName.c_str(), priority);
125    if (err != OK) {
126        mThread.clear();
127    }
128
129    return err;
130}
```
如果runOnCallingThread为true（在调用线程运行），首先使用mLock加同步锁，如果mThread为NULL，或者mRunningLocally为true（在本地线程运行），直接返回INVALID_OPERATION。将mRunningLocally设为true。执行一个无限循环，直到loop方法返回false为止。返回OK。如果runOnCallingThread为false，先用mLock加同步锁，如果mThread为NULL，或者mRunningLocally为true（在本地线程运行），直接返回INVALID_OPERATION。生成一个新的LooperThread，赋值给mThread。调用mThread的run方法。如果run方法返回的状态不是OK，需要调用clear方法。
loop方法的代码如下：
```
bool ALooper::loop() {
195    Event event;
196
197    {
198        Mutex::Autolock autoLock(mLock);
199        if (mThread == NULL && !mRunningLocally) {
200            return false;
201        }
202        if (mEventQueue.empty()) {
203            mQueueChangedCondition.wait(mLock);
204            return true;
205        }
206        int64_t whenUs = (*mEventQueue.begin()).mWhenUs;
207        int64_t nowUs = GetNowUs();
208
209        if (whenUs > nowUs) {
210            int64_t delayUs = whenUs - nowUs;
211            mQueueChangedCondition.waitRelative(mLock, delayUs * 1000ll);
212
213            return true;
214        }
215
216        event = *mEventQueue.begin();
217        mEventQueue.erase(mEventQueue.begin());
218    }
219
220    event.mMessage->deliver();
221
222    // NOTE: It's important to note that at this point our "ALooper" object
223    // may no longer exist (its final reference may have gone away while
224    // delivering the message). We have made sure, however, that loop()
225    // won't be called again.
226
227    return true;
228}
```
首先加同步锁。如果mThread为NULL，或者mRunningLocally为false（非本地线程执行），返回false。如果mEventQueue列表为空，等待解锁，并且返回true。获取mEventQueue中的一个元素的mWhenUs属性（当时的微秒数）whenUs，以及当前的微秒数nowUs，如果whenUs大于nowUs，whenUs减去nowUs的差值delayUs。等待一个delayUs * 1000的相对时间后解锁，并返回true。获取mQueue中的第一个event，并且从mQueue中移除这个event。调用event.mMessage->deliver()投递消息到handler，返回true。
mEventQueue中的Event从何而来呢？是通过post方法加入的。post方法的代码如下：
```
void ALooper::post(const sp<AMessage> &msg, int64_t delayUs) {
169    Mutex::Autolock autoLock(mLock);
170
171    int64_t whenUs;
172    if (delayUs > 0) {
173        whenUs = GetNowUs() + delayUs;
174    } else {
175        whenUs = GetNowUs();
176    }
177
178    List<Event>::iterator it = mEventQueue.begin();
179    while (it != mEventQueue.end() && (*it).mWhenUs <= whenUs) {
180        ++it;
181    }
182
183    Event event;
184    event.mWhenUs = whenUs;
185    event.mMessage = msg;
186
187    if (it == mEventQueue.begin()) {
188        mQueueChangedCondition.signal();
189    }
190
191    mEventQueue.insert(it, event);
192}
```
首先要加同步锁，得到whenUs的值，通过迭代器遍历mEventQueue，在mEventQueue中找到mWhenUs大于whenUs的Event的位置。创建新Event，event的mWhenUs就是之前的whenUs，mMessage是参数msg。如果找到的位置位于mEventQueue的开始，直接调用mQueueChangedCondition.signal()激活解锁。把event插入到前面找到的位置。
ALooper中还有个重要的方法stop：
```
status_t ALooper::stop() {
133    sp<LooperThread> thread;
134    bool runningLocally;
135
136    {
137        Mutex::Autolock autoLock(mLock);
138
139        thread = mThread;
140        runningLocally = mRunningLocally;
141        mThread.clear();
142        mRunningLocally = false;
143    }
144
145    if (thread == NULL && !runningLocally) {
146        return INVALID_OPERATION;
147    }
148
149    if (thread != NULL) {
150        thread->requestExit();
151    }
152
153    mQueueChangedCondition.signal();
154    {
155        Mutex::Autolock autoLock(mRepliesLock);
156        mRepliesCondition.broadcast();
157    }
158
159    if (!runningLocally && !thread->isCurrentThread()) {
160        // If not running locally and this thread _is_ the looper thread,
161        // the loop() function will return and never be called again.
162        thread->requestExitAndWait();
163    }
164
165    return OK;
166}
```
首先加同步锁，调用mThread的clear，恢复mRunningLocally为false。如果thread为NULL，或者mRunningLocally为false，返回INVALID_OPERATION。stop主要做一些退出前的清理操作。