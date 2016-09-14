## MTCall信息数据库存储 ##
### 1. 相关类 ###
- CallsManager.java
- CallLogManager.java
- DbModifierWithNotification.java
- CallLogProvider.java

### 2. 过程简述 ###
   数据库的存储工作是被CallsManager的回调，将Call信息传给个CallLogManger，电话记录的控制中心，将Call中的数据解析，判断。传入异步任务进行耗时操作，操作为对数据的查询比对，增加，打包，以及最后的数据库插入。
### 3. 调用流程 ###
#### CallsManager.java ####
```java 
private void setCallState(Call call, int newState, String tag)
```
这个方法在`onSuccessfulIncomingCall`中调用，简略代码：
```java
private void setCallState(Call call, int newState, String tag) {
        if (call == null) {
            return;
        }
        int oldState = call.getState();
        Log.i(this, "setCallState %s -> %s, call: %s", CallState.toString(oldState),
                CallState.toString(newState), call);
        if (newState != oldState) {
            
            call.setState(newState, tag);//改变传入call的状态

            Trace.beginSection("onCallStateChanged");

            if (mCalls.contains(call)) {
               
                CallsManagerListener somcAmCallsManagerListener = null;
                
                for (CallsManagerListener listener : mListeners) {
                    
                    if (listener instanceof SomcAmCallsManager) {
                        somcAmCallsManagerListener = listener;
                    } else {
                        listener.onCallStateChanged(call, oldState, newState);//调用回调方法
                    }
                  
                }
               
                if (somcAmCallsManagerListener != null) {
                    somcAmCallsManagerListener.onCallStateChanged(call, oldState, newState);
                }
                
                updateCallsManagerState();
            }
            
        }
        
    }
```
  

**我们调用了回调方法，回调接口在`CallLogManager.java`中存在实现关系**

#### CallLogManager.java ####

```java 
    public void onCallStateChanged(Call call, int oldState, int newState) {
        int disconnectCause = call.getDisconnectCause().getCode();
        boolean isNewlyDisconnected =
                newState == CallState.DISCONNECTED || newState == CallState.ABORTED;
        boolean isCallCanceled = isNewlyDisconnected && disconnectCause == DisconnectCause.CANCELED;

        if (isNewlyDisconnected &&
                (oldState != CallState.SELECT_PHONE_ACCOUNT &&
                 !call.isConference() &&
                 !isCallCanceled)) {
            int type;
            if (!call.isIncoming()) {
                type = Calls.OUTGOING_TYPE;
            } else if (disconnectCause == DisconnectCause.MISSED) {
                type = Calls.MISSED_TYPE;
            } else {
                type = getCallLogTypeForAm(call);//这里获得的type是INCOMING_TYPE
            }
            logCall(call, type);//执行此方法
        }
    }
```

CallLogManager中的logCall方法主要完成将Call中的信息取出并判断

```java
    void logCall(Call call, int callLogType) {
        final long creationTime = call.getCreationTimeMillis();
        final long age = call.getAgeMillis();

        final String logNumber = getLogNumber(call);
        final PhoneAccountHandle emergencyAccountHandle =
                TelephonyUtil.getDefaultEmergencyPhoneAccount().getAccountHandle();

        PhoneAccountHandle accountHandle = call.getTargetPhoneAccount();
        if (emergencyAccountHandle.equals(accountHandle)) {
            accountHandle = null;
        }
        SomcAmCallsManager amCallsManager = mCallsManager.getSomcAmCallsManager();
		//Answering machine, streams and other properties.
		//管理应答机，流管理和其他配置

        if (amCallsManager.isThisAnsweringMachineCall(call) && isOkToLogThisCall) {
            setAmContentValues(logNumber, call.getHandlePresentation(), creationTime);
        }

        CallerInfo callerInfo = call.getCallerInfo();
        if (callerInfo != null) {
            callerInfo.name = SomcTelecomUtils.getNameFromCall(mContext, call);
            callerInfo.cnapName = callerInfo.name;
        }
        logCall(callerInfo, logNumber, call.getHandlePresentation(),
                callLogType, callFeatures, accountHandle, creationTime, age, null,
                call.isEmergencyCall());//将取出的数据进行下一操作
    } 
```

之后执行logCall方法，完成异步任务的准备和开启工作

```java
    private void logCall(
            CallerInfo callerInfo,
            String number,
            int presentation,
            int callType,
            int features,
            PhoneAccountHandle accountHandle,
            long start,
            long duration,
            Long dataUsage,
            boolean isEmergency) {

        final boolean okToLogEmergencyNumber =
                mContext.getResources().getBoolean(R.bool.allow_emergency_numbers_in_call_log);

        final boolean isOkToLogThisCall = !isEmergency || okToLogEmergencyNumber;

        sendAddCallBroadcast(callType, duration);//TODO:

        if (isOkToLogThisCall) {//数据为true
           
            AddCallArgs args = new AddCallArgs(mContext, callerInfo, number, presentation,
                    callType, features, accountHandle, start, duration, dataUsage);
            logCallAsync(args);//继续执行的方法

            if ((callerInfo != null) && (callType != Calls.OUTGOING_TYPE)
                    && (!TextUtils.isEmpty(callerInfo.cnapName))
                    && (!TextUtils.isEmpty(number))
                    && (!callerInfo.contactExists)) {
                new AddCnapTask().execute(args);//true，true，true，true，第五个参数是是否存在于数据库中，不存在则执行
            }
        } 
    }
```
继续执行 `logCallAsync(args)`,新建并开启一个异步任务
```java
public AsyncTask<AddCallArgs, Void, Uri[]> logCallAsync(AddCallArgs args) {
        return new LogCallAsyncTask().execute(args);
    }
```

#### LogCallAsyncTask ####
内部类，根据重写的方法，执行`doInBackground`，执行结果是我们需要的`Uri`数组
```java
protected Uri[] doInBackground(AddCallArgs... callList) {
            int count = callList.length;
            Uri[] result = new Uri[count];//单人或多人通话
            for (int i = 0; i < count; i++) {
                AddCallArgs c = callList[i];
                try {
                    result[i] = Calls.addCall(c.callerInfo, c.context, c.number, c.presentation,
                            c.callType, c.features, c.accountHandle, c.timestamp, c.durationInSec,
                            c.dataUsage, true /* addForAllUsers */);
                } catch (Exception e) {
                }
            }
            return result;
        }
```
继续执行`Calls.addCall()`方法，存在
```java
import android.provider.CallLog.Calls;
```
因此调用到了`CallLog.Calls`此方法要完成数据库操作之前的准备工作。包括获取已经存在联系人的Uri，更新DataUsage等操作。
```java
 public static Uri addCall(CallerInfo ci, Context context, String number,
                int presentation, int callType, int features, PhoneAccountHandle accountHandle,
                long start, int duration, Long dataUsage, boolean addForAllUsers, boolean is_read) {
            int numberPresentation = PRESENTATION_ALLOWED;

            TelecomManager tm = null;
            try {
                tm = TelecomManager.from(context);
            } catch (UnsupportedOperationException e) {}

            String accountAddress = null;

            if (tm != null && accountHandle != null) {
                PhoneAccount account = tm.getPhoneAccount(accountHandle);
                if (account != null) {
                    Uri address = account.getSubscriptionAddress();
                    if (address != null) {
                        accountAddress = address.getSchemeSpecificPart();//此处简要说明，通过TelecomManager
                                                                        //获取到本地已经存在的联系人的Uri
                    }
                }
            }

            /*正常来电的情况下， 删除的代码块不会被执行*/
            // accountHandle information
            String accountComponentString = null;
            String accountId = null;
            if (accountHandle != null) {
                accountComponentString = accountHandle.getComponentName().flattenToString();
                accountId = accountHandle.getId();
            }

            ContentValues values = new ContentValues(6);
            //存放信息
            values.put(NUMBER, number);
            values.put(NUMBER_PRESENTATION, Integer.valueOf(numberPresentation));
            values.put(TYPE, Integer.valueOf(callType));
            values.put(FEATURES, features);
            values.put(DATE, Long.valueOf(start));
            values.put(DURATION, Long.valueOf(duration));
            if (dataUsage != null) {
                values.put(DATA_USAGE, dataUsage);
            }
            values.put(PHONE_ACCOUNT_COMPONENT_NAME, accountComponentString);
            values.put(PHONE_ACCOUNT_ID, accountId);
            values.put(PHONE_ACCOUNT_ADDRESS, accountAddress);
            values.put(NEW, Integer.valueOf(1));

            if (callType == MISSED_TYPE) {
                values.put(IS_READ, Integer.valueOf(is_read ? 1 : 0));
            }

            if ((ci != null) && (ci.contactIdOrZero > 0)) {               

                final Cursor cursor;

                if (ci.normalizedNumber != null) {
                    final String normalizedPhoneNumber = ci.normalizedNumber;
                    cursor = resolver.query(Phone.CONTENT_URI,
                            new String[] { Phone._ID },
                            Phone.CONTACT_ID + " =? AND " + Phone.NORMALIZED_NUMBER + " =?",
                            new String[] { String.valueOf(ci.contactIdOrZero),
                                    normalizedPhoneNumber},
                            null);
                } else {
                  
                }

                if (cursor != null) {
                    try {
                        if (cursor.getCount() > 0 && cursor.moveToFirst()) {
                            final String dataId = cursor.getString(0);
                            updateDataUsageStatForData(resolver, dataId);//将Calls表中的DataUsage字段更新
                            if (duration >= MIN_DURATION_FOR_NORMALIZED_NUMBER_UPDATE_MS
                                    && callType == Calls.OUTGOING_TYPE//条件不成立
                                    && TextUtils.isEmpty(ci.normalizedNumber)) {
                                updateNormalizedNumber(context, resolver, dataId, number);//不执行
                            }
                        }
                    } finally {
                        cursor.close();
                    }
                }
            }

            Uri result = null;

            if (addForAllUsers) {
                // Insert the entry for all currently running users, in order to trigger any
                // ContentObservers currently set on the call log.
                final UserManager userManager = (UserManager) context.getSystemService(
                        Context.USER_SERVICE);
                List<UserInfo> users = userManager.getUsers(true);
                final int currentUserId = userManager.getUserHandle();
                final int count = users.size();
                for (int i = 0; i < count; i++) {
                    final UserInfo user = users.get(i);
                    final UserHandle userHandle = user.getUserHandle();
                    if (userManager.isUserRunning(userHandle)
                            && !userManager.hasUserRestriction(UserManager.DISALLOW_OUTGOING_CALLS,
                                    userHandle)
                            && !user.isManagedProfile()) {
                        Uri uri = addEntryAndRemoveExpiredEntries(context,
                                ContentProvider.maybeAddUserId(CONTENT_URI, user.id), values);//这里是针对多用户存在的情况下
                        if (user.id == currentUserId) {
                            result = uri;
                        }
                    }
                }
            } else {
                result = addEntryAndRemoveExpiredEntries(context, CONTENT_URI, values);//默认的操作，继续执行，得到返回的Uri数组
            }

            return result;
        }
```
继续执行, 获取`ContentProvider`去处理插入和一项删除操作，传入的Uri为：
```java
public static final Uri CONTENT_URI = Uri.parse("content://call_log/calls");
```
```java
private static Uri addEntryAndRemoveExpiredEntries(Context context, Uri uri,
                ContentValues values) {
            final ContentResolver resolver = context.getContentResolver();
            Uri result = resolver.insert(uri, values);
            resolver.delete(uri, "_id IN " +
                    "(SELECT _id FROM calls ORDER BY " + DEFAULT_SORT_ORDER
                    + " LIMIT -1 OFFSET 500)", null);
            return result;
        }
```
### 数据库操作 ###
根据Uri匹配规则
#### CallLogProvider.java ####
执行insert方法,在此方法中，进行了线程安全，数据比对，添加其他信息继续执行。
```java
public Uri insert(Uri uri, ContentValues values) {
        waitForAccess(mReadAccessLatch);//特殊情况下线程中断等待interrupt方法
        checkForSupportedColumns(sCallsProjectionMap, values);//传入的数据是否对应和正确，抛出异常
        // Inserting a voicemail record through call_log requires the voicemail
        // permission and also requires the additional voicemail param set.
        if (hasVoicemailValue(values)) {
            checkIsAllowVoicemailRequest(uri);
            mVoicemailPermissions.checkCallerHasWriteAccess(getCallingPackage());
        }
        if (mCallsInserter == null) {//获取帮助类
            SQLiteDatabase db = mDbHelper.getWritableDatabase();
            mCallsInserter = new DatabaseUtils.InsertHelper(db, Tables.CALLS);
        }

        ContentValues copiedValues = new ContentValues(values);

        // Add the computed fields to the copied values.
        mCallLogInsertionHelper.addComputedValues(copiedValues);//添加了诸如Google位置信息，群组信息等

        long rowId = getDatabaseModifier(mCallsInserter).insert(copiedValues);//继续执行
        if (rowId > 0) {
            return ContentUris.withAppendedId(uri, rowId);
        }
        return null;
    }
```
调用getDatabaseModifier()将会新建一个DbModifierWithNotification对象。
#### DatabaseModifier.java ####
```java
private DatabaseModifier getDatabaseModifier(SQLiteDatabase db) {
        return new DbModifierWithNotification(Tables.CALLS, db, context());
    }
```
构造方法
```java
public DbModifierWithNotification(String tableName, SQLiteDatabase db, Context context) {
        this(tableName, db, null, context);
    }
```
继续调用
```java
private DbModifierWithNotification(String tableName, SQLiteDatabase db,
            InsertHelper insertHelper, Context context) {
        mTableName = tableName;
        mDb = db;
        mInsertHelper = insertHelper;
        mContext = context;
        mBaseUri = mTableName.equals(Tables.VOICEMAIL_STATUS) ?
                Status.CONTENT_URI : Voicemails.CONTENT_URI;
        mIsCallsTable = mTableName.equals(Tables.CALLS);//进行判断
        mVoicemailPermissions = new VoicemailPermissions(mContext);
    }
```
对象新建完成之后要继续进行insert（）方法
```java
public long insert(ContentValues values) {
        Set<String> packagesModified = getModifiedPackages(values);
        long rowId = mInsertHelper.insert(values);//具体的插入操作细节我们不需要去关心，数据已经完整的赋予了InsertHelper
        if (rowId > 0 && packagesModified.size() != 0) {
            notifyVoicemailChangeOnInsert(
                    ContentUris.withAppendedId(mBaseUri, rowId), packagesModified);//通知有语音消息
        }
        if (rowId > 0 && mIsCallsTable) {
            notifyCallLogChange();//执行此方法
        }
        return rowId;//返回的操作，回头再去看最终返回，对返回的Uri没有操作
    }
```
### 小结 ###
##### 至此，数据库的操作就已经完成 #####
#### 之后便是广播之后的操作，广播之后，会勾起BackupApp的服务，暂时不考虑。####