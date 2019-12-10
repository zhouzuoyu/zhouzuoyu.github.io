---
title: AudioTrack使用示例
date: 2019-12-09 10:35:00
categories: 
- Android
- audio
---

> frameworks\base\media\tests\audiotests\shared_mem_test.cpp

```c++
int main(int argc, char *argv[]) {

    return android::main();
		test = new AudioTrackTest();
			InitSine();	// 构建数据
		test->Execute();
			Test01();
}
```
<!--more-->
上层new AudioTrack将会导致对应output的playbackThread中也创建track与之对应
```c++
int AudioTrackTest::Test01() {
	for (int i = 0; i < 1024; i++) {
        sp<AudioTrack> track = new AudioTrack(AUDIO_STREAM_MUSIC,// stream type
               rate,
               AUDIO_FORMAT_PCM_16_BIT,// word length, PCM
               AUDIO_CHANNEL_OUT_MONO,
               iMem);
		    mStatus = set(streamType, sampleRate, format, channelMask,
					0 /*frameCount*/, flags, cbf, user, notificationFrames,
					sharedBuffer, false /*threadCanCallJava*/, sessionId, transferType, offloadInfo,
					uid, pid, pAttributes);
				setAttributesFromStreamType(streamType);
					mAttributes.content_type = AUDIO_CONTENT_TYPE_MUSIC;
        			mAttributes.usage = AUDIO_USAGE_MEDIA;
				status = createTrack_l();

		track->start();
	}
}
```
