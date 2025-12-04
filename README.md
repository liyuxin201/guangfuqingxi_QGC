1.AppState.qml文件是定义的全局变量存储文件；

2.Flowview.qml
*无人机作业任务中的水泵监控、流量显示、远程控制与自动返航逻辑管理**

| 功能 | 说明 |
|------|------|
| 💧 **实时监控** | 实时显示水泵流速（L/min）、累计流量（L）、剩余水量等数据 |
| ⚙️ **远程控制** | 支持设置目标流速、启动/暂停泵工作、初始化控制 |
| ⚡ **TCP通信** | 与机载端（PumpModel、MyTcpClient）进行数据通讯 |
| 🚨 **低液位检测** | 自动检测水箱液位，当剩余水量 ≤ 0.5L 时触发“自动返航加水”流程 |
| ✈️ **自动返航** | 在低液位时触发返航逻辑，记录当前任务点后执行返航 |
| 🔁 **任务恢复** | 加水完成后可点击“返回工作点”继续执行剩余任务 |
| 🧪 **调试模式** | 可开启调试模式，用于开发阶段模拟低液位事件 |

## 🧩 组件结构与逻辑分区

1. **顶部控制栏**
   - “返回”按钮（触发 `closeRequested()`）
   - “初始化”按钮（发送初始化水泵命令）
   - （可选）测试模式开关 / 模拟低液位按钮（仅调试模式下可见）

2. **警报显示区域**
   - 当触发低液位或返航时显示闪烁警示按钮  
   - 点击显示返航状态对话框

3. **数据监控区**
   - 流速、累计流量、水箱容积、剩余容积  
   - 实时刷新频率为 100ms

4. **命令输入区**
   - 可设置目标流速（0~100）并发送至机载端

5. **连接配置区**
   - 输入 IP 地址与端口号，连接/断开机载电脑

6. **状态通知栏**
   - 条形状态条 + 文本，显示当前连接状态

7. **自动返航逻辑**
   - 与 `PumpModel` 联动检测剩余水量，当 ≤ 0.5L 时自动触发返航
   - 自动调用无人机对象的 `returnMission()`  
   - 用户加水完成后点击“返回工作点”重新执行任务

---

## 🧠 内部自动逻辑流程

```mermaid  
flowchart TD  
A[开始监控] --> B[读取流量与水箱容积]  
B --> |剩余水量 > 0.5L| C[持续更新显示]  
B --> |剩余水量 ≤ 0.5L| D[检测到低液位]  
D --> E[触发 lowWaterLevelAlert 信号]  
E --> F[发送停止泵指令 + 记录航点]  
F --> G[调用 activeVehicle.returnMission()]  
G --> H[返航加水中...]  
H --> I[用户确认加水完成 → 返回工作点]  
I --> J[activeVehicle.startMission(savedIndex)]  
J --> K[恢复正常工作]

3.FlyStatusView.qml
//新增进入清洗界面按钮
TextButton {
                  id:clean_btn
                  visible: Shared.AppState.appMode === 0
                      buttonText: qsTr("进入清洗界面")
                      height: button_height
                      width: 200 * ScreenTools.scaleWidth
                      onClicked: {
                          flowViewDialog.open()
                      }
                }


//在上传航点按钮上新增定位航点按钮实现飞机打点
TextButton {
                  buttonText: qsTr("定位航点")
                  height: button_height
                  width: 200 * ScreenTools.scaleWidth
                  onClicked: {
                        // 判断是否成功定位
                        if (!VkSdkInstance.vehicleManager.activeVehicle) return
                        let activeVehicle = VkSdkInstance.vehicleManager.activeVehicle
                        let gpsFixType = activeVehicle.GNSS1.gpsInputFixType
                        if (gpsFixType > 1) {
                            let gpsLat = activeVehicle.GNSS1.gpsInputLatitude
                            let gpsLon = activeVehicle.GNSS1.gpsInputLongitude
//调用qml中存在的添加航点函数，通过获取经纬度添加航点，mapControl是FlyViewMap的一个对象
                            mapControl.addWaypointByCoordinate(gpsLon, gpsLat)
                        } else {
                            console.warn(qsTr("GPS未定位,无法添加航点"))
                        }
                  }
            }

            TextButton {
                  buttonText: qsTr("上传航点")
                  height: button_height
                  width: 200 * ScreenTools.scaleWidth
                  onClicked: {
                        VkSdkInstance.vehicleManager.activeVehicle.uploadMissionModel(
                                          missionModel)
                        mapControl.add_type = 0
                        right_button_status = 3 //显示执行页面
                  }
            }

4.MessageAllShow.qml是基础信息显示界面，里面新增了雷达的障碍物距离的显示；

5.WaypointSettingsDialog.qml
新增航点设置自动清洗开启或关闭按钮，通过定时器判断当前航点变化，当航点变化时会自动开启或关闭水阀

function checkArrivalStatus() {
        if (activeVehicle && activeVehicle.missionCurrent) {
                    savedMissionWaypointIndex = activeVehicle.missionCurrent.missionCurrentSeq
                     console.log("🔖 记录当前航点序号:", savedMissionWaypointIndex)
                } else {
                    savedMissionWaypointIndex = -1
                    // console.warn("⚠️ 无法获取当前任务信息，航点序号记录失败")
                }
        if((savedMissionWaypointIndex !== lastMissionWaypointIndex) && (savedMissionWaypointIndex > 0))
        {
        if(Shared.AppState.clearModes[savedMissionWaypointIndex-1]===1)
        {
            Shared.AppState.isopen_pump = 1;
            MyTcpClient.send_isopen_pump(1);//关闭水泵
        }
        if(Shared.AppState.clearModes[savedMissionWaypointIndex-1]===0)
        {
             Shared.AppState.isopen_pump = 0;
            MyTcpClient.send_isopen_pump(0);//开启水泵
        }
        }

        if(lastHeartbeatCustomMode !== activeVehicle.heartbeat.heartbeatCustomMode)
        {
            if(activeVehicle.heartbeat.heartbeatCustomMode === 12 || activeVehicle.heartbeat.heartbeatCustomMode === 11)
            {

                Shared.AppState.isopen_pump = 1;
                MyTcpClient.send_isopen_pump(1);//关闭水泵
            }
        }
         lastMissionWaypointIndex = savedMissionWaypointIndex;
        lastHeartbeatCustomMode = activeVehicle.heartbeat.heartbeatCustomMode;//保存上次航点或者飞行模式
        }
