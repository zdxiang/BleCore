# BleCore Android蓝牙低功耗(BLE)快速开发框架

### 用法

        allprojects {
            repositories {
                ...
                maven { url 'https://jitpack.io' }
            }
        }

        dependencies {
            implementation 'com.github.buhuiming:BleCore:1.0.0-beta01'
        }

#### 1、添加权限

    //动态申请
    val LOCATION_PERMISSION = if (Build.VERSION.SDK_INT < Build.VERSION_CODES.S) {
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION,
            )
        } else {
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION,
                Manifest.permission.BLUETOOTH_SCAN,
                Manifest.permission.BLUETOOTH_ADVERTISE,
                Manifest.permission.BLUETOOTH_CONNECT,
            )
        }

*    注意：
*    有些设备GPS是关闭状态的话，申请定位权限之后，GPS是依然关闭状态，这里要根据GPS是否打开来跳转页面
*    BleUtil.isGpsOpen(context) 判断GPS是否打开
*    跳转到系统GPS设置页面，GPS设置是全局的独立的，是否打开跟权限申请无关
     startActivity(Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS))
*    跳转到系统蓝牙设置页面
     startActivity(Intent(Settings.ACTION_BLUETOOTH_SETTINGS))

#### 1、初始化
    val options =
            BleOptions.builder()
                .setScanServiceUuid("0000ff80-0000-1000-8000-00805f9b34fb", "0000ff90-0000-1000-8000-00805f9b34fb")
                .setScanDeviceName("midea", "BYD BLE3")
                .setScanDeviceAddress("70:86:CE:88:7A:AF", "5B:AE:65:88:59:5E", "B8:8C:29:8B:BE:07")
                .isContainScanDeviceName(true)
                .setAutoConnect(false)
                .setEnableLog(true)
                .setScanMillisTimeOut(12000)
                //这个机制是：不会因为扫描的次数导致上一次扫描到的数据被清空，也就是onScanStart和onScanComplete
                //都只会回调一次，而且扫描到的数据是所有扫描次数的总和
                .setScanRetryCountAndInterval(2, 1000)
                .setConnectMillisTimeOut(10000)
                .setConnectRetryCountAndInterval(2, 5000)
                .setOperateMillisTimeOut(6000)
                .setWriteInterval(80)
                .setMaxConnectNum(5)
                .setMtu(500)
                .build()
    BleManager.get().init(application, options)

    //或者使用默认配置
    BleManager.get().init(application)

#### 2、扫描
    注意：扫描之前先检查权限、检查GPS开关、检查蓝牙开关
    扫描及过滤过程是在工作线程中进行，所以不会影响主线程的UI操作，最终每一个回调结果都会回到主线程。
    开启扫描：
    BleManager.get().startScan {
        onScanStart {

        }
        onLeScan { bleDevice, currentScanCount ->
            //可以根据currentScanCount是否已有清空列表数据
        }
        onLeScanDuplicateRemoval { bleDevice, currentScanCount ->
            //与onLeScan区别之处在于：同一个设备只会出现一次
        }
        onScanComplete { bleDeviceList, bleDeviceDuplicateRemovalList ->
            //扫描到的数据是所有扫描次数的总和
        }
        onScanFail {
            val msg: String = when (it) {
                is BleScanFailType.UnTypeSupportBle -> "BleScanFailType.UnTypeSupportBle: 设备不支持蓝牙"
                is BleScanFailType.NoBlePermissionType -> "BleScanFailType.NoBlePermissionType: 权限不足，请检查"
                is BleScanFailType.GPSDisable -> "BleScanFailType.BleDisable: 设备未打开GPS定位"
                is BleScanFailType.BleDisable -> "BleScanFailType.BleDisable: 蓝牙未打开"
                is BleScanFailType.AlReadyScanning -> "BleScanFailType.AlReadyScanning: 正在扫描"
                is BleScanFailType.ScanError -> {
                    "BleScanFailType.ScanError: ${it.throwable?.message}"
                }
            }
            BleLogger.e(msg)
            Toast.makeText(application, msg, Toast.LENGTH_SHORT).show()
        }
    }

#### 3、停止扫描
    BleManager.get().stopScan()

#### 4、是否扫描中
    BleManager.get().isScanning()

#### 5、连接
    BleManager.get().connect(device)
    BleManager.get().connect(deviceAddress)

*    在某些型号手机上，connectGatt必须在主线程才能有效，所以把连接过程放在主线程，回调也在主线程。
*    为保证重连成功率，建议断开后间隔一段时间之后进行重连。

#### 6、断开连接
    BleManager.get().disConnect(device)
    BleManager.get().disConnect(deviceAddress)

*    断开后，并不会马上更新状态，所以马上连接会直接返回已连接，而且扫描不出来，要等待一定时间才可以

#### 7、是否已连接
    BleManager.get().isConnected(device)

#### 8、扫描并连接，如果扫描到多个设备，则会连接第一个
    BleManager.get().startScanAndConnect(bleScanCallback: BleScanCallback,
                                         bleConnectCallback: BleConnectCallback)

*    扫描到首个符合扫描规则的设备后，便停止扫描，然后连接该设备。

#### 9、移除该设备的连接回调
    BleManager.get().removeBleConnectCallback(device)

#### 10、设置Notify
    BleManager.get().notify(bleDevice: BleDevice,
                                  serviceUUID: String,
                                  notifyUUID: String,
                                  useCharacteristicDescriptor: Boolean = false,
                                  bleIndicateCallback: BleIndicateCallback)

#### 11、取消Notify
    BleManager.get().stopNotify(bleDevice: BleDevice,
                                  serviceUUID: String,
                                  notifyUUID: String,
                                  useCharacteristicDescriptor: Boolean = false)

#### 12、设置Indicate
    BleManager.get().indicate(bleDevice: BleDevice,
                                  serviceUUID: String,
                                  indicateUUID: String,
                                  useCharacteristicDescriptor: Boolean = false,
                                  bleIndicateCallback: BleIndicateCallback)

#### 13、取消Indicate
    BleManager.get().stopIndicate(bleDevice: BleDevice,
                                  serviceUUID: String,
                                  indicateUUID: String,
                                  useCharacteristicDescriptor: Boolean = false)

#### 14、读取信号值
    BleManager.get().readRssi(bleDevice: BleDevice, bleRssiCallback: BleRssiCallback)
    
*    获取设备的信号强度，需要在设备连接之后进行。
*    某些设备可能无法读取Rssi，不会回调onRssiSuccess(),而会因为超时而回调onRssiFail()。

#### 15、设置Mtu值
    BleManager.get().setMtu(bleDevice: BleDevice, bleMtuChangedCallback: BleMtuChangedCallback) 

*    设置MTU，需要在设备连接之后进行操作。
*    默认每一个BLE设备都必须支持的MTU为23。
*    MTU为23，表示最多可以发送20个字节的数据。
*    该方法的参数mtu，最小设置为23，最大设置为512。
*    并不是每台设备都支持拓展MTU，需要通讯双方都支持才行，也就是说，需要设备硬件也支持拓展MTU该方法才会起效果。
     调用该方法后，可以通过onMtuChanged(int mtu)查看最终设置完后，设备的最大传输单元被拓展到多少。如果设备不支持，
     可能无论设置多少，最终的mtu还是23。 

#### 16、断开某个设备的连接 释放资源
    BleManager.get().setConnectionPriority(connectionPriority: Int)

*    设置连接的优先级，一般用于高速传输大量数据的时候可以进行设置。

#### 17、读特征值数据
    BleManager.get().readData(bleDevice: BleDevice,
                              serviceUUID: String,
                              readUUID: String,
                              bleIndicateCallback: BleReadCallback)

#### 18、断开某个设备的连接 释放资源
    BleManager.get().release(bleDevice: BleDevice)

#### 19、断开所有连接 释放资源
    BleManager.get().releaseAll()

#### 20、一些移除监听的函数
    BleManager.get().removeBleScanCallback()
    BleManager.get().removeBleConnectCallback(bleDevice: BleDevice)
    BleManager.get().removeBleIndicateCallback(bleDevice: BleDevice, indicateUUID: String)
    BleManager.get().removeBleNotifyCallback(bleDevice: BleDevice, notifyUUID: String)
    BleManager.get().removeBleRssiCallback(bleDevice: BleDevice)
    BleManager.get().removeBleMtuChangedCallback(bleDevice: BleDevice)
    BleManager.get().removeBleReadCallback(bleDevice: BleDevice, readUUID: String)
    BleManager.get().removeBleWriteCallback(bleDevice: BleDevice, writeUUID: String)
    
## License

```
Copyright (c) 2023 Bekie

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```