---
title: AudioTrack流程
date: 2019-12-09 13:33:00
categories: 
- Android
- audio
---

应用每new AudioTrack都会在PlaybackThread创建出一个track，即应用程序的AudioTrack和PlaybackThread的track是一一对应的。

根据new AudioTrack的streamType参数选择最合适的output
每一个output都对应一个PlaybackThread
PlaybackThread都有mTracks，在对应的PlaybackThread中new Track <!--more-->
```c++
sp<AudioTrack> track = new AudioTrack(AUDIO_STREAM_MUSIC,// stream type
									   rate,
									   AUDIO_FORMAT_PCM_16_BIT,// word length, PCM
									   AUDIO_CHANNEL_OUT_MONO,
									   iMem);
	mStatus = set(streamType, sampleRate, format, channelMask,
					0 /*frameCount*/, flags, cbf, user, notificationFrames,
					sharedBuffer, false /*threadCanCallJava*/, sessionId, transferType, offloadInfo,
					uid, pid, pAttributes);
		status = createTrack_l();
			const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
			// 根据new AudioTrack的streamType参数选择最合适的output
			audio_io_handle_t output = AudioSystem::getOutputForAttr(&mAttributes, mSampleRate, mFormat, mChannelMask, mFlags, mOffloadInfo);
			// output对应的PlaybackThread new Track
			sp<IAudioTrack> track = audioFlinger->createTrack(mStreamType,	// AudioFlinger::createTrack
					                                          mSampleRate,
					                                          // AudioFlinger only sees 16-bit PCM
					                                          mFormat == AUDIO_FORMAT_PCM_8_BIT &&
					                                              !(mFlags & AUDIO_OUTPUT_FLAG_DIRECT) ?
					                                                  AUDIO_FORMAT_PCM_16_BIT : mFormat,
					                                          mChannelMask,
					                                          &temp,
					                                          &trackFlags,
					                                          mSharedBuffer,
					                                          output,
					                                          tid,
					                                          &mSessionId,
					                                          mClientUid,
					                                          &status);
```

PlaybackThread会维护mTracks
```c++
sp<IAudioTrack> AudioFlinger::createTrack(...)
{
	// 获取output对应的PlaybackThread
	PlaybackThread *thread = checkPlaybackThread_l(output);
		return mPlaybackThreads.valueFor(output).get();
	// 在PlaybackThread下new Track
    track = thread->createTrack_l(client, streamType, sampleRate, format,
            channelMask, frameCount, sharedBuffer, lSessionId, flags, tid, clientUid, &lStatus);
		track = new Track(this, client, streamType, sampleRate, format,
                          channelMask, frameCount, NULL, sharedBuffer,
                          sessionId, uid, *flags, TrackBase::TYPE_DEFAULT);
		mTracks.add(track);
}
```
