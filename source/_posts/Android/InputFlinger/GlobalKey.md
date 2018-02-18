---
title: InputFlinger(5)_GlobalKey
date: 2018-02-11 12:00:00
categories: 
- Android
- InputFlinger
---

# 源码
GlobalKeyManager.java (base\policy\src\com\android\internal\policy\impl)

# 配置文件
> frameworks/base/core/res/res/xml/global_keys.xml

eg.
key keyCode="KEYCODE_VOLUME_UP" 
component="com.android.example.keys/.VolumeKeyHandler"
<!-- more -->

# 功能
根据global_keys.xml，接收到键值则发送对应广播

# 源码分析
```java
final class GlobalKeyManager {
    // 发送对应广播
    boolean handleGlobalKey(Context context, int keyCode, KeyEvent event) {
        if (mKeyMapping.size() > 0) {
            ComponentName component = mKeyMapping.get(keyCode);
            if (component != null) {
                Intent intent = new Intent(Intent.ACTION_GLOBAL_BUTTON)
                        .setComponent(component)
                        .putExtra(Intent.EXTRA_KEY_EVENT, event);
                context.sendBroadcastAsUser(intent, UserHandle.CURRENT, null);
                return true;
            }
        }
        return false;
    }
	// 解析frameworks/base/core/res/res/xml/global_keys.xml到mKeyMapping中
    private void loadGlobalKeys(Context context) {
    	parser = context.getResources().getXml(com.android.internal.R.xml.global_keys);
        XmlUtils.beginDocument(parser, TAG_GLOBAL_KEYS);
        int version = parser.getAttributeIntValue(null, ATTR_VERSION, 0);
        if (GLOBAL_KEY_FILE_VERSION == version) {
            while (true) {
                XmlUtils.nextElement(parser);
                String element = parser.getName();
                if (element == null) {
                    break;
                }
                if (TAG_KEY.equals(element)) {
                    String keyCodeName = parser.getAttributeValue(null, ATTR_KEY_CODE);
                    String componentName = parser.getAttributeValue(null, ATTR_COMPONENT);
                    int keyCode = KeyEvent.keyCodeFromString(keyCodeName);
                    if (keyCode != KeyEvent.KEYCODE_UNKNOWN) {
                        mKeyMapping.put(keyCode, ComponentName.unflattenFromString(componentName));	// 添加到GlobalKeyManager.mKeyMapping
                    }
                }
            }
        }
    }
}
```
