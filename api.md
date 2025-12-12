###水泵流速、开关、初始化：

机载电脑TCP连接类：

class MyTcpClient : public QObject
{
public:

// Q_INVOKABLE 方法供 QML 调用

    Q_INVOKABLE void connectToHost(const QString &ip, quint16 port);//连接机载电脑
    Q_INVOKABLE void disconnectFromHost();//断开机载电脑
    Q_INVOKABLE void sendMessage(const QString &message);//发送流速
    Q_INVOKABLE void send_isopen_pump(const char &message);//控制水泵开关
    Q_INVOKABLE void send_init_pump(const char &message);//水泵初始化

  }
  
//流速、流量获取类

  class PumpModel : public QObject {
public:
    explicit PumpModel(QObject *parent = nullptr);

    double flowRate() const;
    void setFlowRate(double rate);

    double totalVolume() const;
    void setTotalVolume(double volume);

           // 添加Q_INVOKABLE方法
    Q_INVOKABLE double getFlowRateValue() const { return m_flowRate; }      //获取流速
    Q_INVOKABLE double getTotalVolumeValue() const { return m_totalVolume; }  //获取流量
}'''

//AppState.qml         全局变量

property int appMode: 0       //清洗模式和巡查模式切换 

property var clearModes: [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]//所有航点水泵自动状态，1为关闭

property int isopen_pump: 1   //水泵开启状态，1默认关闭

使用示例：

qml文件中引入->        import "qrc:/qml/FlightDisplay" as Shared

使用变量时->           Shared.AppState.isopen_pump = 0
    
