# I-D-F
a intelligence-drinking-fountain

//Arduino 

#define SSID        "HONOR V20" //改为你的热点名称, 不要有中文
#define PASSWORD    "12345ssdlh"//改为你的WiFi密码Wi-Fi密码
#define DEVICEID    "576081575" //OneNet上的设备ID
String apiKey = "nTaezRxrg1qx8rjN=zLOvzjq0Zw=";//与你的设备绑定的APIKey
/***/
#define HOST_NAME   "api.heclouds.com"
#define HOST_PORT   (80)
#define INTERVAL_SENSOR   5000             //定义传感器采样时间间隔  597000
#define INTERVAL_NET      5000             //定义发送时间
//传感器部分================================   
#include <Wire.h>                                  //调用库  
#include <ESP8266.h>
#include <I2Cdev.h>                                //调用库  
/*******温湿度*******/
#include <Microduino_SHT2x.h>
/*******光照*******/
#define  sensorPin_1  A0
#define IDLE_TIMEOUT_MS  3000      // Amount of time to wait (in milliseconds) with no data 
                                   // received before closing the connection.  If you know the server
                                   // you're accessing is quick to respond, you can reduce this value.
//WEBSITE     
char buf[10];
#define INTERVAL_sensor 2000
unsigned long sensorlastTime = millis();
float tempOLED, humiOLED, lightnessOLED;
#define INTERVAL_OLED 1000
String mCottenData;
String jsonToSend;
//3,传感器值的设置 
//float sensor_tem, sensor_hum, sensor_lux;                    //传感器温度、湿度、光照   
//char  sensor_tem_c[7], sensor_hum_c[7], sensor_lux_c[7] ;    //换成char数组传输
#include <SoftwareSerial.h>
#define EspSerial mySerial
#define UARTSPEED  9600
SoftwareSerial mySerial(2, 3); /* RX:D3, TX:D2 */
ESP8266 wifi(&EspSerial);
//ESP8266 wifi(Serial1);                                      //定义一个ESP8266（wifi）的对象
unsigned long net_time1 = millis();                          //数据上传服务器时间
unsigned long sensor_time = millis();                        //传感器采样时间计时器
//int SensorData;                                   //用于存储传感器数据
String postString;                                //用于存储发送数据的字符串
//String jsonToSend;                                //用于存储发送的json格式参数
Tem_Hum_S2 TempMonitor;
volatile float dist;                 //测得的距离
char c_dist[7];                      //换成char
float checkdistance_A1_A3() {
  digitalWrite(A1, LOW);
  delayMicroseconds(2);
  digitalWrite(A1, HIGH);
  delayMicroseconds(10);
  digitalWrite(A1, LOW);
  float distance = pulseIn(A3, HIGH) / 58.00;
  delay(10);
  return distance;
}
void setup(void)     //初始化函数  
{       
  dist = 0;
  pinMode(A1, OUTPUT);
  pinMode(A3, INPUT);
  Serial.begin(9600);
  pinMode(4, OUTPUT);
  //初始化串口波特率  
    Wire.begin();
    Serial.begin(9600);
    while (!Serial); // wait for Leonardo enumeration, others continue immediately
    Serial.print(F("setup begin\r\n"));
    delay(100);
    pinMode(sensorPin_1, INPUT);
  WifiInit(EspSerial, UARTSPEED);
  Serial.print(F("FW Version:"));
  Serial.println(wifi.getVersion().c_str());
  if (wifi.setOprToStationSoftAP()) {
    Serial.print(F("to station + softap ok\r\n"));
  } else {
    Serial.print(F("to station + softap err\r\n"));
  }
  if (wifi.joinAP(SSID, PASSWORD)) {
    Serial.print(F("Join AP success\r\n"));
    Serial.print(F("IP:"));
    Serial.println( wifi.getLocalIP().c_str());
  } else {
    Serial.print(F("Join AP failure\r\n"));
  }
  if (wifi.disableMUX()) {
    Serial.print(F("single ok\r\n"));
  } else {
    Serial.print(F("single err\r\n"));
  }
  Serial.print(F("setup end\r\n"));
}
void loop(void)     //循环函数  
{   
   dist = checkdistance_A1_A3();
  Serial.println(dist);
  delay(1000);
  if (dist < 5) {    
    tone(4,100,1000);
  }
  if (sensor_time > millis())  sensor_time = millis();  
  if(millis() - sensor_time > INTERVAL_SENSOR)              //传感器采样时间间隔  
  {  
    getSensorData();                                        //读串口中的传感器数据
    sensor_time = millis();
  } 
  if (net_time1 > millis())  net_time1 = millis();
  if (millis() - net_time1 > INTERVAL_NET)                  //发送数据时间间隔
  {                
    updateSensorData();                                     //将数据上传到服务器的函数
    net_time1 = millis();
  }
}
void getSensorData(){  
    //sensor_tem = TempMonitor.getTemperature();  
   // sensor_hum = TempMonitor.getHumidity();   
    //获取光照
    //sensor_lux = analogRead(A0);    
    //delay(1000);
    dtostrf(dist, 2, 2, c_dist);
    //dtostrf(sensor_hum, 2, 1, sensor_hum_c);
    //dtostrf(sensor_lux, 3, 1, sensor_lux_c);
}
void updateSensorData() {
  if (wifi.createTCP(HOST_NAME, HOST_PORT)) { //建立TCP连接，如果失败，不能发送该数据
    Serial.print("create tcp ok\r\n");
jsonToSend="{\"distance\":";
    dtostrf(dist,1,2,buf);
    jsonToSend+="\""+String(buf)+"\"";
    //jsonToSend+=",\"Humidity\":";
    //dtostrf(sensor_hum,1,2,buf);
    //jsonToSend+="\""+String(buf)+"\"";
   // jsonToSend+=",\"Light\":";
   // dtostrf(sensor_lux,1,2,buf);
    //jsonToSend+="\""+String(buf)+"\"";
    jsonToSend+="}";
    postString="POST /devices/";
    postString+=DEVICEID;
    postString+="/datapoints?type=3 HTTP/1.1";
    postString+="\r\n";
    postString+="api-key:";
    postString+=apiKey;
    postString+="\r\n";
    postString+="Host:api.heclouds.com\r\n";
    postString+="Connection:close\r\n";
    postString+="Content-Length:";
    postString+=jsonToSend.length();
    postString+="\r\n";
    postString+="\r\n";
    postString+=jsonToSend;
    postString+="\r\n";
    postString+="\r\n";
    postString+="\r\n";
  const char *postArray = postString.c_str();                 //将str转化为char数组
  Serial.println(postArray);
  wifi.send((const uint8_t*)postArray, strlen(postArray));    //send发送命令，参数必须是这两种格式，尤其是(const uint8_t*)
  Serial.println("send success");   
     if (wifi.releaseTCP()) {                                 //释放TCP连接
        Serial.print("release tcp ok\r\n");
        } 
     else {
        Serial.print("release tcp err\r\n");
        }
      postArray = NULL;                                       //清空数组，等待下次传输数据
  } else {
    Serial.print("create tcp err\r\n");
  }
}

M-cookie

#define SSID "HONOR V20"//改为你的热点名称, 不要有中文
#define PASSWORD "12345ssdlh"//改为你的WiFi密码Wi-Fi密码
#define DEVICEID "576081575"//OneNet上的设备ID
String apiKey = "nTaezRxrg1qx8rjN=zLOvzjq0Zw=";//与你的设备绑定的APIKey
/***/
#define HOST_NAME   "api.heclouds.com"
#define HOST_PORT   (80)
#define INTERVAL_SENSOR   5000             //定义传感器采样时间间隔  597000
#define INTERVAL_NET      5000             //定义发送时间
//传感器部分================================   
#include <Wire.h>                                  //调用库  
#include <ESP8266.h>
#include <I2Cdev.h>                                //调用库  
/*******温湿度*******/
#include <Microduino_SHT2x.h>
/*******光照*******/
#define  sensorPin_1  A0
#define IDLE_TIMEOUT_MS  3000      // Amount of time to wait (in milliseconds) with no data 
                                   // received before closing the connection.  If you know the server
                                   // you're accessing is quick to respond, you can reduce this value.
//WEBSITE     
char buf[10];
#define INTERVAL_sensor 2000
unsigned long sensorlastTime = millis();
float tempOLED, humiOLED, lightnessOLED;
#define INTERVAL_OLED 1000
String mCottenData;
String jsonToSend;
//3,传感器值的设置 
float sensor_tem, sensor_hum, sensor_lux;                    //传感器温度、湿度、光照   
char  sensor_tem_c[7], sensor_hum_c[7], sensor_lux_c[7] ;    //换成char数组传输
#include <SoftwareSerial.h>
#define EspSerial mySerial
#define UARTSPEED  9600
SoftwareSerial mySerial(2, 3); /* RX:D3, TX:D2 */
ESP8266 wifi(&EspSerial);
//ESP8266 wifi(Serial1);                                      //定义一个ESP8266（wifi）的对象
unsigned long net_time1 = millis();                          //数据上传服务器时间
unsigned long sensor_time = millis();                        //传感器采样时间计时器
//int SensorData;                                   //用于存储传感器数据
String postString;                                //用于存储发送数据的字符串
//String jsonToSend;                                //用于存储发送的json格式参数
Tem_Hum_S2 TempMonitor;
void setup(void)     //初始化函数  
{       
  //初始化串口波特率  
    Wire.begin();
    Serial.begin(115200);
    while (!Serial); // wait for Leonardo enumeration, others continue immediately
    Serial.print(F("setup begin\r\n"));
    delay(100);
    pinMode(sensorPin_1, INPUT);
  WifiInit(EspSerial, UARTSPEED);
  Serial.print(F("FW Version:"));
  Serial.println(wifi.getVersion().c_str());
  if (wifi.setOprToStationSoftAP()) {
    Serial.print(F("to station + softap ok\r\n"));
  } else {
    Serial.print(F("to station + softap err\r\n"));
  }
  if (wifi.joinAP(SSID, PASSWORD)) {
    Serial.print(F("Join AP success\r\n"));
    Serial.print(F("IP:"));
    Serial.println( wifi.getLocalIP().c_str());
  } else {
    Serial.print(F("Join AP failure\r\n"));
  }
  if (wifi.disableMUX()) {
    Serial.print(F("single ok\r\n"));
  } else {
    Serial.print(F("single err\r\n"));
  }
  Serial.print(F("setup end\r\n"));
}
void loop(void)     //循环函数  
{   
  if (sensor_time > millis())  sensor_time = millis();  
  if(millis() - sensor_time > INTERVAL_SENSOR)              //传感器采样时间间隔  
  {  
    getSensorData();                                        //读串口中的传感器数据
    sensor_time = millis();
  }  
  if (net_time1 > millis())  net_time1 = millis();
  if (millis() - net_time1 > INTERVAL_NET)                  //发送数据时间间隔
  {                
    updateSensorData();                                     //将数据上传到服务器的函数
    net_time1 = millis();
  }
}
void getSensorData(){  
    sensor_tem = TempMonitor.getTemperature();  
    sensor_hum = TempMonitor.getHumidity();   
    //获取光照
    sensor_lux = analogRead(A0);    
    delay(1000);
    dtostrf(sensor_tem, 2, 1, sensor_tem_c);
    dtostrf(sensor_hum, 2, 1, sensor_hum_c);
    dtostrf(sensor_lux, 3, 1, sensor_lux_c);
}
void updateSensorData() {
  if (wifi.createTCP(HOST_NAME, HOST_PORT)) { //建立TCP连接，如果失败，不能发送该数据
    Serial.print("create tcp ok\r\n");
jsonToSend="{\"Temperature\":";
    dtostrf(sensor_tem,1,2,buf);
    jsonToSend+="\""+String(buf)+"\"";
    jsonToSend+="}";
    postString="POST /devices/";
    postString+=DEVICEID;
    postString+="/datapoints?type=3 HTTP/1.1";
    postString+="\r\n";
    postString+="api-key:";
    postString+=apiKey;
    postString+="\r\n";
    postString+="Host:api.heclouds.com\r\n";
    postString+="Connection:close\r\n";
    postString+="Content-Length:";
    postString+=jsonToSend.length();
    postString+="\r\n";
    postString+="\r\n";
    postString+=jsonToSend;
    postString+="\r\n";
    postString+="\r\n";
    postString+="\r\n";
  const char *postArray = postString.c_str();                 //将str转化为char数组
  Serial.println(postArray);
  wifi.send((const uint8_t*)postArray, strlen(postArray));    //send发送命令，参数必须是这两种格式，尤其是(const uint8_t*)
  Serial.println("send success");   
     if (wifi.releaseTCP()) {                                 //释放TCP连接
        Serial.print("release tcp ok\r\n");
        } 
     else {
        Serial.print("release tcp err\r\n");
        }
      postArray = NULL;                                       //清空数组，等待下次传输数据
  } else {
    Serial.print("create tcp err\r\n");
  }
}

//WE-chat

//Start.js
Page({
    data: {
    },
    navigate: function() {
        wx.navigateTo({
            url: '../wifi_station/shuiwen/shuiwen',
        }) 
    },
     navigate1: function () {
    wx.navigateTo({
      url: '../wifi_station/juli/juli',
    })
  }
})

//Start.json
{
  "enablePullDownRefresh": true
}

//Start.wxml
<view class="container" > <image src="http://5b0988e595225.cdn.sohucs.com/images/20170918/bbc9b371d83e443b87506a26889c5e85.jpeg" class='in-image' mode='widthFix'></image> <other-widget class="other-wigdget"></other-widget> </view>
<button class="button"  bindtap="navigate">current temperature</button>
<button class="button1"  bindtap="navigate1">current water level</button>
<div class="snow-container">
<div class="snow foreground"></div>
<div class="snow foreground layered"></div>
<div class="snow middleground"></div>
<div class="snow middleground layered"></div>
<div class="snow background"></div>
<div class="snow background layered"></div>
</div>
<canvas>distance</canvas>

//Start.wxss
.container{
position: fixed;
height:1000px;
left:0;
right:0;
}
.other-widget{
position: absolute;
}
.button{
margin-bottom: 20rpx ;
font-weight:bold;
font-style:oblique;
border:2px solid #f877c2;
padding:10px 40px;
background:#e280a138;
width:300px;
border-radius:25px;
color:#ac050538;
justify-content:center;
}
.button1{
margin-bottom: 100rpx ;
font-weight:bold;
font-style:oblique;
border:2px solid #f877c2;
padding:10px 40px;
background:#e280a138;
width:300px;
border-radius:25px;
color:#ac050538;
}
page{
height: 100%;
background-color:#ff000038;
}
.snow {
display: block;
position: absolute;
z-index: 2;
top: 0;
right: 0;
bottom: 0;
left: 0;
pointer-events: none;
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
-webkit-animation: snow linear infinite;
animation: snow linear infinite;
}
.snow.foreground {
background-image: url("https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-large-075d267ecbc42e3564c8ed43516dd557.png");
-webkit-animation-duration: 15s;
animation-duration: 15s;
}
.snow.foreground.layered {
-webkit-animation-delay: 7.5s;
animation-delay: 7.5s;
}
.snow.middleground {
background-image: url(https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-medium-0b8a5e0732315b68e1f54185be7a1ad9.png);
-webkit-animation-duration: 20s;
animation-duration: 20s;
}
.snow.middleground.layered {
-webkit-animation-delay: 10s;
animation-delay: 10s;
}
.snow.background {
background-image: url(https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-small-1ecd03b1fce08c24e064ff8c0a72c519.png);
-webkit-animation-duration: 30s;
animation-duration: 30s;
}
.snow.background.layered {
-webkit-animation-delay: 15s;
animation-delay: 15s;
}
@-webkit-keyframes snow {
0% {
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
}
100% {
-webkit-transform: translate3d(15%, 100%, 0);
transform: translate3d(15%, 100%, 0);
}
}
@keyframes snow {
0% {
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
}
100% {
-webkit-transform: translate3d(15%, 100%, 0);
transform: translate3d(15%, 100%, 0);
}
}

//Shuiwen.js
var myCharts = require("../../../utils/wxcharts.js")//引入一个绘图的插件
const devicesId = "576081575" // 填写在OneNet上获得的devicesId 形式就是一串数字 例子:9939133
const api_key = "nTaezRxrg1qx8rjN=zLOvzjq0Zw=" // 填写在OneNet上的 api-key 例子: VeFI0HZ44Qn5dZO14AuLbWSlSlI=
Page({
  data: {},
  /**
   * @description 页面下拉刷新事件
   */
  onPullDownRefresh: function () {
    wx.showLoading({
      title: "正在获取"
    })
    this.getDatapoints().then(datapoints => {
      this.update(datapoints)
      wx.hideLoading()
    }).catch((error) => {
      wx.hideLoading()
      console.error(error)
    })
  },
  /**
   * @description 页面加载生命周期
   */
  onLoad: function () {
    console.log(`your deviceId: ${devicesId}, apiKey: ${api_key}`)
    //每隔6s自动获取一次数据进行更新
    const timer = setInterval(() => {
      this.getDatapoints().then(datapoints => {
        this.update(datapoints)
      })
    }, 6000)
    wx.showLoading({
      title: '加载中'
    })
    this.getDatapoints().then((datapoints) => {
      wx.hideLoading()
      this.firstDraw(datapoints)
    }).catch((err) => {
      wx.hideLoading()
      console.error(err)
      clearInterval(timer) //首次渲染发生错误时禁止自动刷新
    })
  },
  /**
   * 向OneNet请求当前设备的数据点
   * @returns Promise
   */
  getDatapoints: function () {
    return new Promise((resolve, reject) => {
      wx.request({
        url: `https://api.heclouds.com/devices/${devicesId}/datapoints?datastream_id=Light,Temperature,Humidity&limit=20`,
        /**
         * 添加HTTP报文的请求头, 
         * 其中api-key为OneNet的api文档要求我们添加的鉴权秘钥
         * Content-Type的作用是标识请求体的格式, 从api文档中我们读到请求体是json格式的
         * 故content-type属性应设置为application/json
         */
        header: {
          'content-type': 'application/json',
          'api-key': api_key
        },
        success: (res) => {
          const status = res.statusCode
          const response = res.data
          if (status !== 200) { // 返回状态码不为200时将Promise置为reject状态
            reject(res.data)
            return;
          }
          if (response.errno !== 0) { //errno不为零说明可能参数有误, 将Promise置为reject
            reject(response.error)
            return;
          }
          if (response.data.datastreams.length === 0) {
            reject("当前设备无数据, 请先运行硬件实验")
          }

          //程序可以运行到这里说明请求成功, 将Promise置为resolve状态
          resolve({
            temperature: response.data.datastreams[0].datapoints.reverse(),
            light: response.data.datastreams[1].datapoints.reverse(),
            humidity: response.data.datastreams[2].datapoints.reverse()
          })
        },
        fail: (err) => {
          reject(err)
        }
      })
    })
  },
  /**
   * @param {Object[]} datapoints 从OneNet云平台上获取到的数据点
   * 传入获取到的数据点, 函数自动更新图标
   */
  update: function (datapoints) {
    const wheatherData = this.convert(datapoints);
    this.lineChart_hum.updateData({
      categories: wheatherData.categories,
      series: [{
        name: 'humidity',
        data: wheatherData.humidity,
        format: (val, name) => val.toFixed(2)
      }],
    })
    this.lineChart_light.updateData({
      categories: wheatherData.categories,
      series: [{
        name: 'light',
        data: wheatherData.light,
        format: (val, name) => val.toFixed(2)
      }],
    })
    this.lineChart_tempe.updateData({
      categories: wheatherData.categories,
      series: [{
        name: 'tempe',
        data: wheatherData.tempe,
        format: (val, name) => val.toFixed(2)
      }],
    })
  },
  /**
   * 
   * @param {Object[]} datapoints 从OneNet云平台上获取到的数据点
   * 传入数据点, 返回使用于图表的数据格式
   */
  convert: function (datapoints) {
    var categories = [];
    var humidity = [];
    var light = [];
    var tempe = [];
    var length = datapoints.humidity.length
    for (var i = 0; i < length; i++) {
      categories.push(datapoints.humidity[i].at.slice(5, 19));
      humidity.push(datapoints.humidity[i].value);
      light.push(datapoints.light[i].value);
      tempe.push(datapoints.temperature[i].value);
    }
    return {
      categories: categories,
      humidity: humidity,
      light: light,
      tempe: tempe
    }
  },
  /**
   * 
   * @param {Object[]} datapoints 从OneNet云平台上获取到的数据点
   * 传入数据点, 函数将进行图表的初始化渲染
   */
  firstDraw: function (datapoints) {
    //得到屏幕宽度
    var windowWidth = 320;
    try {
      var res = wx.getSystemInfoSync();
      windowWidth = res.windowWidth;
    } catch (e) {
      console.error('getSystemInfoSync failed!');
    }
    var wheatherData = this.convert(datapoints);
    //新建湿度图表
    /*
    this.lineChart_hum = new myCharts({
      canvasId: 'humidity',
      type: 'line',
      categories: wheatherData.categories,
      animation: false,
      background: '#f5f5f5',
      series: [{
        name: 'humidity',
        data: wheatherData.humidity,
        format: function (val, name) {
          return val.toFixed(2);
        }
      }],
      xAxis: {
        disableGrid: true
      },
      yAxis: {
        title: 'humidity (%)',
        format: function (val) {
          return val.toFixed(2);
        }
      },
      width: windowWidth,
      height: 200,
      dataLabel: false,
      dataPointShape: true,
      extra: {
        lineStyle: 'curve'
      }
    });*/
    // 新建光照强度图表
    /*this.lineChart_light = new myCharts({
      canvasId: 'light',
      type: 'line',
      categories: wheatherData.categories,
      animation: false,
      background: '#f5f5f5',
      series: [{
        name: 'light',
        data: wheatherData.light,
        format: function (val, name) {
          return val.toFixed(2);
        }
      }],
      xAxis: {
        disableGrid: true
      },
      yAxis: {
        title: 'light (lux)',
        format: function (val) {
          return val.toFixed(2);
        }
      },
      width: windowWidth,
      height: 200,
      dataLabel: false,
      dataPointShape: true,
      extra: {
        lineStyle: 'curve'
      }
    });*/
    this.lineChart_tempe = new myCharts({
      canvasId: 'tempe',
      type: 'line',
      categories: wheatherData.categories,
      animation: false,
      background: '#f5f5f5',
      series: [{
        name: 'temperature',
        data: wheatherData.tempe,
        format: function (val, name) {
          return val.toFixed(2);
        }
      }],
      xAxis: {
        disableGrid: true
      },
      yAxis: {
        title: 'temperature (摄氏度)',
        format: function (val) {
          return val.toFixed(2);
        }
      },
      width: windowWidth,
      height: 200,
      dataLabel: false,
      dataPointShape: true,
      extra: {
        lineStyle: 'curve'
      }
    });
  },
})

//Shuiwen.json
{
    "enablePullDownRefresh": true
}

//Shuiwen.wxml
<text class="tshuiwen">水温记录：</text>
<canvas class = "canvas" id="chart2"
canvas-id="distance"></canvas>
<canvas class = "canvas" id="chart3"
canvas-id="tempe"></canvas>
<view class="container" > <image class='in-image' mode='widthFix'></image> <other-widget class="other-wigdget"></other-widget> </view>
<div class="snow-container">
<div class="snow foreground"></div>
<div class="snow foreground layered"></div>
<div class="snow middleground"></div>
<div class="snow middleground layered"></div>
<div class="snow background"></div>
<div class="snow background layered"></div>
</div>

//Shuiwen.wxss
.tshuiwen{
  color: cadetblue;
}
.canvas{
  width: 100%;
  height: 400rpx
}
#chart3{
  top: 10rpx
}
#chart2{
  top: 30rpx
}
.container{
position: fixed;
height:1000px;
left:0;
right:0;
}
.other-widget{
position: absolute;
}
.button{
margin-bottom: 1000rpx ;
font-weight:bold;
font-style:oblique;
border:2px solid #f877c2;
padding:10px 40px;
background:#e280a138;
width:300px;
border-radius:25px;
color:#ac050538;
}
page{
height: 100%;
background-color:#ff000038;
}
.snow {
display: block;
position: absolute;
z-index: 2;
top: 0;
right: 0;
bottom: 0;
left: 0;
pointer-events: none;
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
-webkit-animation: snow linear infinite;
animation: snow linear infinite;
}
.snow.foreground {
background-image: url("https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-large-075d267ecbc42e3564c8ed43516dd557.png");
-webkit-animation-duration: 15s;
animation-duration: 15s;
}
.snow.foreground.layered {
-webkit-animation-delay: 7.5s;
animation-delay: 7.5s;
}
.snow.middleground {
background-image: url(https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-medium-0b8a5e0732315b68e1f54185be7a1ad9.png);
-webkit-animation-duration: 20s;
animation-duration: 20s;
}
.snow.middleground.layered {
-webkit-animation-delay: 10s;
animation-delay: 10s;
}
.snow.background {
background-image: url(https://dl6rt3mwcjzxg.cloudfront.net/assets/snow/snow-small-1ecd03b1fce08c24e064ff8c0a72c519.png);
-webkit-animation-duration: 30s;
animation-duration: 30s;
}
.snow.background.layered {
-webkit-animation-delay: 15s;
animation-delay: 15s;
}
@-webkit-keyframes snow {
0% {
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
}
100% {
-webkit-transform: translate3d(15%, 100%, 0);
transform: translate3d(15%, 100%, 0);
}
}
@keyframes snow {
0% {
-webkit-transform: translate3d(0, -100%, 0);
transform: translate3d(0, -100%, 0);
}
100% {
-webkit-transform: translate3d(15%, 100%, 0);
transform: translate3d(15%, 100%, 0);
}
}

//App.js
App({
  onLaunch() {
    
  },
  onShow: function () {
  },
  onHide: function () {
  }
})

//App.json
{
  "pages": [
    "pages/start/start",
    "pages/wifi_station/shuiwen/shuiwen",
    "pages/wifi_station/juli/juli"
  ],
  "window": {
    "backgroundTextStyle": "light",
    "navigationBarBackgroundColor": "#ffffff",
    "navigationBarTitleText": "智能饮水机",
    "navigationBarTextStyle": "black",
    "enablePullDownRefresh": true
  },
  "debug": false,
  "sitemapLocation": "sitemap.json"
}

//App.wxss
/**app.wxss**/
page{
  height: 100%;
}
