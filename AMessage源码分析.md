# AMessage源码分析

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