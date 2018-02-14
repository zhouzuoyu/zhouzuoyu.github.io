---
title: InputFlinger(2)_EventHub
date: {{ date }}
categories: 
- Android
- InputFlinger
---

# 源码
EventHub.cpp (native\services\inputflinger)

# 功能
使用epoll和inotify机制共同监测
1: /dev/input节点的增减
2: /dev/input/eventx事件上报

# 源码分析
## inotify
使用inotify机制来监测/dev/input/下节点的增减
<!-- more -->
### 创建inotify
```c++
EventHub::EventHub
	// inotify
	mINotifyFd = inotify_init();
    inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);	// 监测"/dev/input"下有无新节点生成
    // epoll /dev/input/下的节点变化
	mEpollFd = epoll_create(EPOLL_SIZE_HINT);
	eventItem.events = EPOLLIN;
    eventItem.data.u32 = EPOLL_ID_INOTIFY;
    epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);
```

### 监测新增设备数据
例如新增设备：则把/dev/input/eventx添加到mEpollFd进行监测
```c++
EventHub::readNotifyLocked()
	// /dev/input/
	strcpy(devname, DEVICE_PATH);
	filename = devname + strlen(devname);
    *filename++ = '/';
    // 从inotify中读取拔插的设备名
    read(mINotifyFd, event_buf, sizeof(event_buf));
    event = (struct inotify_event *)(event_buf + event_pos);
    strcpy(filename, event->name);
    if (event->mask & IN_CREATE)	// create
        openDeviceLocked(devname);
    else							// remove
        closeDeviceByPathLocked(devname);
```

```c++
EventHub::openDeviceLocked(const char *devicePath)	// /dev/input/eventx
	fd = open(devicePath, O_RDWR | O_CLOEXEC);
	
	// Get device identifier.
	ioctl(fd, EVIOCGID, &inputId);
    identifier.bus = inputId.bustype;
    identifier.product = inputId.product;
    identifier.vendor = inputId.vendor;
    identifier.version = inputId.version;
	
	// 创建Device(EventHub.cpp)
	deviceId = mNextDeviceId++;
	Device* device = new Device(fd, deviceId, String8(devicePath), identifier);
	
	// load idc文件
	loadConfigurationLocked(device);
	
	// 如果是keyboard则load kl文件和kcm文件
	ioctl(fd, EVIOCGBIT(EV_KEY, sizeof(device->keyBitmask)), device->keyBitmask);	// 底层设置了EV_KEY，才能认为是一个keyboard设备
	bool haveKeyboardKeys = containsNonZeroByte(device->keyBitmask, 0, sizeof_bit_array(BTN_MISC))
            || containsNonZeroByte(device->keyBitmask, sizeof_bit_array(KEY_OK),
                    sizeof_bit_array(KEY_MAX + 1));
	if (haveKeyboardKeys || haveGamepadButtons)
        device->classes |= INPUT_DEVICE_CLASS_KEYBOARD;
	if (device->classes & (INPUT_DEVICE_CLASS_KEYBOARD | INPUT_DEVICE_CLASS_JOYSTICK))
        keyMapStatus = loadKeyMapLocked(device);
	
	// 监测/dev/input/eventx设备
	eventItem.events = mUsingEpollWakeup ? EPOLLIN : EPOLLIN | EPOLLWAKEUP;
    eventItem.data.u32 = deviceId;
	epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem);
	
	addDeviceLocked(device);
		mDevices.add(device->id, device);
```

## getEvents
epoll_wait监测/dev/input/目录下所有的文件，有数据则read，最终将event数据传递给buffer
```c++
EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize)
	RawEvent* event = buffer;
	
	for (;;) {
		// 对/dev/input/下的所有节点都使用epoll进行监测
		if (mNeedToScanDevices) {	// 构造函数中为true
            mNeedToScanDevices = false;
            scanDevicesLocked();
            	scanDirLocked(DEVICE_PATH);	// "/dev/input"
            		dir = opendir(dirname);
            		strcpy(devname, dirname);
            		filename = devname + strlen(devname);
				    *filename++ = '/';
            		while((de = readdir(dir))) {
						strcpy(filename, de->d_name);
						openDeviceLocked(devname);	// devname == '/dev/input/eventx'
					}
            mNeedToSendFinishedDeviceScan = true;
        }
        
		// 读取数据
		while (mPendingEventIndex < mPendingEventCount) {
			const struct epoll_event& eventItem = mPendingEventItems[mPendingEventIndex++];			// 解析事件
			// 1: "/dev/input"下增加了节点/删除了节点，标志位置位但是不读取数据
			if (eventItem.data.u32 == EPOLL_ID_INOTIFY) {
				if (eventItem.events & EPOLLIN)
                    mPendingINotify = true;	// 节点变动
                continue;
			}
			if (mPendingINotify && mPendingEventIndex >= mPendingEventCount) {
		        mPendingINotify = false;
		        readNotifyLocked();	// 将新增eventx添加到epoll进行监测
		        deviceChanged = true;
		    }
			// 2: 读取event数据，将input_event转化为RawEvent传递出去
			deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
			mDevices.valueAt(deviceIndex);
			if (eventItem.events & EPOLLIN) {
				readSize = read(device->fd, readBuffer, sizeof(struct input_event) * capacity);	// 有readSize个event
				count = size_t(readSize) / sizeof(struct input_event);
				for (size_t i = 0; i < count; i++) {
					struct input_event& iev = readBuffer[i];
					event->deviceId = deviceId;
                    event->type = iev.type;
                    event->code = iev.code;
                    event->value = iev.value;
                    event += 1;
				}
			}
		}
		// 有数据则结束此次getEvents
		if (event != buffer || awoken) {
            break;
        }
        // epoll_wait，等待事件
		pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);	// mPendingEventItems：产生的所有事件
		mPendingEventCount = size_t(pollResult);	// mPendingEventCount: 有多少events需要处理
	}
```

## idc/kl/kcm文件
### idc
> InputDeviceConfiguration，用于描述这个input设备特性
idc优先级如下：
* /system/usr/idc/Vendor_XXXX_Product_XXXX_Version_XXXX.idc
* /data/system/devices/idc/Vendor_XXXX_Product_XXXX_Version_XXXX.idc
* /system/usr/idc/Vendor_XXXX_Product_XXXX.idc
* /data/system/devices/idc/Vendor_XXXX_Product_XXXX.idc
* /system/usr/idc/DEVICE_NAME.idc
* /data/system/devices/idc/DEVICE_NAME.idc

```c++
EventHub::loadConfigurationLocked(Device* device)
	// 1: Search idc文件
	device->configurationFile = getInputDeviceConfigurationFilePathByDeviceIdentifier(device->identifier, INPUT_DEVICE_CONFIGURATION_FILE_TYPE_CONFIGURATION);
		// 1: 在/system/usr/idc/和/data/system/devices/idc/下search Vendor_XXXX_Product_XXXX_Version_XXXX.idc
		String8 versionPath(getInputDeviceConfigurationFilePathByName(
		        String8::format("Vendor_%04x_Product_%04x_Version_%04x",
		                deviceIdentifier.vendor, deviceIdentifier.product,
		                deviceIdentifier.version),
		        type));
		    // 1.1: 先检查/system目录下
			path.setTo(getenv("ANDROID_ROOT"));					// /system
			path.append("/usr/");								// /system/usr/
			appendInputDeviceConfigurationFileRelativePath(path, name, type);
				path.append(CONFIGURATION_FILE_DIR[type]);		// /system/usr/idc/
				for (size_t i = 0; i < name.length(); i++) {
					char ch = name[i];
					if (!isValidNameChar(ch)) {
						ch = '_';
					}
					path.append(&ch, 1);						// /system/usr/idc/Vendor_%04x_Product_%04x_Version_%04x
				}
				path.append(CONFIGURATION_FILE_EXTENSION[type]);// /system/usr/idc/Vendor_%04x_Product_%04x_Version_%04x.idc
			if (!access(path.string(), R_OK))
			    return path;
			// 1.2: 再检查/data
			path.setTo(getenv("ANDROID_DATA"));									// /data
			path.append("/system/devices/");									// /data/system/devices/idc/
			appendInputDeviceConfigurationFileRelativePath(path, name, type);	// /data/system/devices/idc/Vendor_%04x_Product_%04x_Version_%04x.idc
			if (!access(path.string(), R_OK))
				return path;
		// 2: 在/system/usr/idc/和/data/system/devices/idc/下search Vendor_XXXX_Product_XXXX.idc
		String8 productPath(getInputDeviceConfigurationFilePathByName(
		    String8::format("Vendor_%04x_Product_%04x",
		            deviceIdentifier.vendor, deviceIdentifier.product),
		    type));
		// 3: 在/system/usr/idc/和/data/system/devices/idc/下search deviceIdentifier.name.idc
		getInputDeviceConfigurationFilePathByName(deviceIdentifier.name, type);

	// 2: load
	PropertyMap::load(device->configurationFile, &device->configuration);
```

### kl
kl功能：
linux中的Scancode -> android的AKEYCODE_XXX(native\include\android\Keycodes.h)
key 139 MENU

```c++
KeyLayoutMap::Parser::parse()
	// 解析kl文件
	while (!mTokenizer->isEof()) {
		// 解析kl文件一行信息
		if (!mTokenizer->isEol() && mTokenizer->peekChar() != '#') {
			String8 keywordToken = mTokenizer->nextToken(WHITESPACE);
			if (keywordToken == "key")											// search关键字'key'：key 139    MENU
		    	parseKey();
		    		// scancode
		    		String8 codeToken = mTokenizer->nextToken(WHITESPACE);		// "139"
		    		int32_t code = int32_t(strtol(codeToken.string(), &end, 0));// 139
		    		// AKEYCODE_XXX
		    		String8 keyCodeToken = mTokenizer->nextToken(WHITESPACE);	// "MENU"
		    		keyCode = getKeyCodeByLabel(keyCodeToken.string());			// AKEYCODE_MENU
		    			return int32_t(lookupValueByLabel(label, KEYCODES));
		    				while (list->literal) {
								if (strcmp(literal, list->literal) == 0)
									return list->value;
								list++;
							}
					key.keyCode = keyCode;	// AKEYCODE_XXX
					key.flags = flags;
					// 把linux的scancode和android的AKEYCODE_XXX的对应关系填充到 mKeysByScanCode
					KeyedVector<int32_t, Key>& map = mapUsage ? mMap->mKeysByUsageCode : mMap->mKeysByScanCode;	// mKeysByScanCode
					map.add(code, key);
		}
		mTokenizer->nextLine();
	}
```

### kcm
kcm功能：
1：AKEYCODE_XXX->unicode
2：定义Scancode -> AKEYCODE_XXX的对应关系
```c++
KeyCharacterMap::Parser::parse()
	while (!mTokenizer->isEof()) {
		if (!mTokenizer->isEol() && mTokenizer->peekChar() != '#') {
		switch (mState) {
            case STATE_TOP: {
                String8 keywordToken = mTokenizer->nextToken(WHITESPACE);
                /*
                 * Scancode -> AKEYCODE_XXX
                 * map key 139 MENU
                 */
                if (keywordToken == "map") {												// map 
                	// scancode->AKEYCODE_XXX
                    mTokenizer->skipDelimiters(WHITESPACE);
                    status_t status = parseMap();
                    	if (keywordToken == "key") {										// map key
							mTokenizer->skipDelimiters(WHITESPACE);
							return parseMapKey();
								String8 codeToken = mTokenizer->nextToken(WHITESPACE);		// map key 139
								int32_t code = int32_t(strtol(codeToken.string(), &end, 0));// "139" -> 139
								KeyedVector<int32_t, int32_t>& map = mapUsage ? mMap->mKeysByUsageCode : mMap->mKeysByScanCode;
								String8 keyCodeToken = mTokenizer->nextToken(WHITESPACE);	// MENU
								int32_t keyCode = getKeyCodeByLabel(keyCodeToken.string());	// AKEYCODE_MENU: 82
								map.add(code, keyCode);	// 存在mKeysByScanCode，so Scancode 139 -> AKEYCODE_MENU 82
						}
                } else if (keywordToken == "key") {
                	/*
                	 * AKEYCODE_XXX->unicode，分为两部，STATE_TOP->STATE_KEY
                	 * Part 1
                	 * key MENU {
                	 */
                    mTokenizer->skipDelimiters(WHITESPACE);
                    status_t status = parseKey();
                    	int32_t keyCode = getKeyCodeByLabel(keyCodeToken.string());	// AKEYCODE_MENU
                    	String8 openBraceToken = mTokenizer->nextToken(WHITESPACE);	// {
                    	mKeyCode = keyCode;	// AKEYCODE_MENU
						mMap->mKeys.add(keyCode, new Key());
						mState = STATE_KEY;
                }
                break;
            }
			
			// Part 2
            case STATE_KEY: {
                status_t status = parseKeyProperty();
                break;
            }
            }
		}
		mTokenizer->nextLine();
	}
```

### kl/kcm文件优先级
kl/kcm文件优先级如下：
/system/usr/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl/.kcm
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl/.kcm
/system/usr/keylayout/Vendor_XXXX_Product_XXXX.kl/.kcm
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX.kl/.kcm
/system/usr/keylayout/DEVICE_NAME.kl/.kcm
/data/system/devices/keylayout/DEVICE_NAME.kl/.kcm
 
/system/usr/keylayout/Generic.kl
/data/system/devices/keylayout/Generic.kl
```c++
EventHub::loadKeyMapLocked(Device* device)
	device->keyMap.load(device->identifier, device->configuration);
		// 1: 看idc文件中是否有设置kl和kcm
		if (deviceConfiguration->tryGetProperty(String8("keyboard.layout"),
                keyLayoutName)) {
            status_t status = loadKeyLayout(deviceIdenfifier, keyLayoutName);
            if (status == NAME_NOT_FOUND) {
                ALOGE("Configuration for keyboard device '%s' requested keyboard layout '%s' but "
                        "it was not found.",
                        deviceIdenfifier.name.string(), keyLayoutName.string());
            }
        }
        if (deviceConfiguration->tryGetProperty(String8("keyboard.characterMap"),
                keyCharacterMapName)) {
            status_t status = loadKeyCharacterMap(deviceIdenfifier, keyCharacterMapName);
            if (status == NAME_NOT_FOUND) {
                ALOGE("Configuration for keyboard device '%s' requested keyboard character "
                        "map '%s' but it was not found.",
                        deviceIdenfifier.name.string(), keyLayoutName.string());
            }
        }
        if (isComplete())
            return OK;

        // 2: Vendor_XXXX_Product_XXXX_Version_XXXX/ Vendor_XXXX_Product_XXXX/ DEVICE_NAME kl和kcm文件
        if (probeKeyMap(deviceIdenfifier, String8::empty()))
        	// 2.1 kl
        	loadKeyLayout(deviceIdentifier, keyMapName);
        		// 2.1.1 search kl/kcm file
        		String8 path(getPath(deviceIdentifier, name, INPUT_DEVICE_CONFIGURATION_FILE_TYPE_KEY_LAYOUT));
        			return getInputDeviceConfigurationFilePathByDeviceIdentifier(deviceIdentifier, type);
        		// 2.1.2 解析path中的kl信息到keyLayoutMap
        		KeyLayoutMap::load(path, &keyLayoutMap);
        			status_t status = Tokenizer::open(filename, &tokenizer);
        			sp<KeyLayoutMap> map = new KeyLayoutMap();
        			Parser parser(map.get(), tokenizer);
        			parser.parse();
        			*outMap = map;
        		// 2.1.3 path保存到keyLayoutFile
        		keyLayoutFile.setTo(path);
        	// 2.2 kcm
        	loadKeyCharacterMap(deviceIdentifier, keyMapName);
        		String8 path(getPath(deviceIdentifier, name, INPUT_DEVICE_CONFIGURATION_FILE_TYPE_KEY_CHARACTER_MAP));
            	KeyCharacterMap::load(path, KeyCharacterMap::FORMAT_BASE, &keyCharacterMap);
            	keyCharacterMapFile.setTo(path);
	        return OK;
		
		// 3: Generic.kl和kcm文件
		if (probeKeyMap(deviceIdenfifier, String8("Generic")))
	        return OK;
```
