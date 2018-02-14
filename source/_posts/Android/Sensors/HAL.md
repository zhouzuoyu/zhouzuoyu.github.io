---
title: Android Sensors(1)_HAL
date: 2018-01-29 11:29:15
categories: 
- Android
- Sensors
---

源码分析：hardware/akm/AK8975_FS

大致有4类文件，有着不同的分工
1：hardware\akm\ak8975_fs\libsensors\Sensors.cpp
	hal中一般支持多类设备，起框架作用，主要关注poll方法
2：AkmSensor.cpp
	具体sensor的驱动代码
3：SensorBase.cpp
	sensor的共有方法，比如open /dev/input/eventx...
4：InputEventReader.cpp
	读取数据等方法
<!-- more -->
## Sensors.cpp
起总领作用
```c++
static int open_sensors(const struct hw_module_t* module, const char* id,
                        struct hw_device_t** device)
{
	sensors_poll_context_t *dev = new sensors_poll_context_t();
		// sensors_poll_context_t::sensors_poll_context_t
		mSensors[akm] = new AkmSensor();
		mPollFds[akm].fd = mSensors[akm]->getFd();

	// 填充回调函数
	dev->device.common.module   = const_cast<hw_module_t*>(module);
	dev->device.common.close    = poll__close;
	dev->device.activate        = poll__activate;
	dev->device.setDelay        = poll__setDelay;
	dev->device.poll            = poll__poll;

	*device = &dev->device.common;
}
static struct hw_module_methods_t sensors_module_methods = {
	.open = open_sensors
};
// HAL入口
struct sensors_module_t HAL_MODULE_INFO_SYM = {
	.common = {
		.id = SENSORS_HARDWARE_MODULE_ID,
		.methods = &sensors_module_methods,
	},
	.get_sensors_list = sensors__get_sensors_list,
};
```
重点分析一下poll方法：对每个sensor使用sensor->readEvents读取数据
```c++
static int poll__poll(struct sensors_poll_device_t *dev, sensors_event_t* data, int count) {
	sensors_poll_context_t *ctx = (sensors_poll_context_t *)dev;
	return ctx->pollEvents(data, count);	// sensors_poll_context_t::pollEvents
		do {
			for (int i=0 ; count && i<numSensorDrivers ; i++) {
				SensorBase* const sensor(mSensors[i]);	// 转化为基类
				if ((mPollFds[i].revents & POLLIN) || (sensor->hasPendingEvents())) {
					int nb = sensor->readEvents(data, count);	// 多态机制，读取数据
					
					count -= nb;
					nbEvents += nb;
					data += nb;
				}
			}
		} while (n && count);
}
```
## AkmSensor.cpp
具体sensor驱动
```c++
// 得到指定设备的fd
AkmSensor::AkmSensor()
: SensorBase(NULL, "compass"),	// 对/dev/input/eventx进行ioctl(EVIOCGNAME)得到name，如果是"compass"，则能使用该HAL code
{
	mPendingEvents[MagneticField].version = sizeof(sensors_event_t);
	mPendingEvents[MagneticField].sensor = ID_M;
	mPendingEvents[MagneticField].type = SENSOR_TYPE_MAGNETIC_FIELD;	// 上报的数据类型
	mPendingEvents[MagneticField].magnetic.status = SENSOR_STATUS_ACCURACY_HIGH;

	mPendingEvents[Orientation  ].version = sizeof(sensors_event_t);
	mPendingEvents[Orientation  ].sensor = ID_O;
	mPendingEvents[Orientation  ].type = SENSOR_TYPE_ORIENTATION;
	mPendingEvents[Orientation  ].orientation.status = SENSOR_STATUS_ACCURACY_HIGH;
}
```
再看数据读取函数，对接上Sensors.cpp
```c++
int AkmSensor::readEvents(sensors_event_t* data, int count)
	// 读取数据
	ssize_t n = mInputReader.fill(data_fd);	// 
	// 处理数据
	while (count && mInputReader.readEvent(&event)) {
		int type = event->type;
		if (type == EV_ABS) {
			processEvent(event->code, event->value);	// 稍加处理
				case EVENT_TYPE_MAGV_X:
					mPendingMask |= 1<<MagneticField;
					mPendingEvents[MagneticField].magnetic.x = value * CONVERT_M;
					break;
			mInputReader.next();
		} else if (type == EV_SYN) {
            for (int j=0 ; count && mPendingMask && j<numSensors ; j++) {
                if (mPendingMask & (1<<j)) {
                    mPendingMask &= ~(1<<j);
                    mPendingEvents[j].timestamp = time;
                    if (mEnabled[j]) {
                        *data++ = mPendingEvents[j];    // 上传数据
                        count--;
                        numEventReceived++;
                    }
                }
            }
		}
	}
```
## SensorBase.cpp
sensor的共有部分抽象
```c++
// 遍历"/dev/input"下的所有节点，使用ioctl(EVIOCGNAME)得到name，看能否匹配上，匹配上则记录fd
SensorBase::SensorBase(
	const char* dev_name,
	const char* data_name)
    : dev_name(dev_name), data_name(data_name),
      dev_fd(-1), data_fd(-1)
	data_fd = openInput(data_name);
		const char *dirname = "/dev/input";
		dir = opendir(dirname);
		strcpy(devname, dirname);
		filename = devname + strlen(devname);
		*filename++ = '/';
		while((de = readdir(dir))) {
			strcpy(filename, de->d_name);
			fd = open(devname, O_RDONLY);
			if (fd>=0) {
				char name[80];
				if (ioctl(fd, EVIOCGNAME(sizeof(name) - 1), &name) < 1) {
					name[0] = '\0';
				}
				if (!strcmp(name, inputName)) {
					strcpy(input_name, filename);
					break;
				}
			}
		}
		closedir(dir);
    	return fd;
```
## InputEventReader.cpp
维护sensor数据
```c++
// 读取"/dev/input/eventx"数据
ssize_t InputEventCircularReader::fill(int fd)
{
	// read name为compass的"/dev/input/eventx"
	const ssize_t nread = read(fd, mHead, mFreeSpace * sizeof(input_event));

	numEventsRead = nread / sizeof(input_event);
	if (numEventsRead) {
		mHead += numEventsRead;
		mFreeSpace -= numEventsRead;
		if (mHead > mBufferEnd) {
			size_t s = mHead - mBufferEnd;
			memcpy(mBuffer, mBufferEnd, s * sizeof(input_event));
			mHead = mBuffer + s;
		}
	}

	return numEventsRead;
}

// 更新数据指针位置
ssize_t InputEventCircularReader::readEvent(input_event const** events)
{
	*events = mCurr;
}
void InputEventCircularReader::next()
{
	mCurr++;
}
```
