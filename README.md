#include <Arduino.h>
#include "WiFi.h"
#include "PubSubClient.h"
#include "Ticker.h"
//#include "uFire_SHT20.h"
#include "ArduinoJson.h"
#include <OneWire.h>
#include <DallasTemperature.h>

 /* 引脚定义 */
const int oneWireBus = 0;

/* 实例初始化 */
OneWire oneWire(oneWireBus);
DallasTemperature sensors(&oneWire);

//uFire_SHT20 sht20;
 
const char* ssid = "mate40";              //wifi名
const char* password = "cl021210";      //wifi密码
const char* mqtt_server = "mqtts.heclouds.com"; //onenet 的 IP地址 mqtts.heclouds.com 183.230.40.96
const int port = 1883;                    //端口号

#define mqtt_pubid "7hRfGKsO8R"    //产品ID
#define mqtt_devid "2021116541"       //设备名称
//鉴权信息
#define mqtt_password "version=2018-10-31&res=products%2F7hRfGKsO8R%2Fdevices%2F2021116541&et=1717062024&method=md5&sign=dsjhJ5f5hQoWmnoARSq2zg%3D%3D"       //鉴权信息

WiFiClient espClient;           //创建一个WIFI连接客户端
PubSubClient client(espClient); // 创建一个PubSub客户端, 传入创建的WIFI客户端
Ticker tim1;                    //定时器,用来循环上传数据
Ticker tim2;                    //定时器,用来循环上传数据
float temp;
float humi;
float dis;

//设备上传数据的post主题
#define ONENET_TOPIC_PROP_POST "$sys/" mqtt_pubid "/" mqtt_devid "/dp/post/json" //"$sys/" mqtt_pubid "/" mqtt_devid "/thing/property/post"
//接收下发属性设置主题
#define ONENET_TOPIC_PROP_SET  "$sys/" mqtt_pubid "/" mqtt_devid "/dp/post/json/+" //"$sys/" mqtt_pubid "/" mqtt_devid "/thing/property/set"
//接收下发属性设置成功的回复主题
#define ONENET_TOPIC_PROP_SET_REPLY "$sys/" mqtt_pubid "/" mqtt_devid "/dp/post/accepted" //"$sys/" mqtt_pubid "/" mqtt_devid "/thing/property/set_reply"
 
//接收设备属性获取命令主题
#define ONENET_TOPIC_PROP_GET "$sys/" mqtt_pubid "/" mqtt_devid "/cmd/request/+" //"$sys/" mqtt_pubid "/" mqtt_devid "/thing/property/get"
//接收设备属性获取命令成功的回复主题
#define ONENET_TOPIC_PROP_GET_REPLY "$sys/" mqtt_pubid "/" mqtt_devid "/cmd/response/+/+" //"$sys/" mqtt_pubid "/" mqtt_devid "/thing/property/get_reply"
 
//这是post上传数据使用的模板
#define ONENET_POST_BODY_FORMAT "{\"id\":%d,\"dp\":%s}"
//#define ONENET_POST_BODY_FORMAT
int postMsgId = 0; //记录已经post了多少条


//连接WIFI相关函数
void setupWifi()
{
  delay(10);
  Serial.println("connect WIFI");
  WiFi.begin(ssid, password);
  while (!WiFi.isConnected())
  {
    Serial.print(".");
    delay(500);
  }
  Serial.println("OK");
  Serial.println("Wifi connected!");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());

}


//向主题发送模拟的温湿度数据
void sendTempAndHumi()
{
  if (client.connected())
  {
    //先拼接出json字符串
    char param[82];
    char jsonBuf[178];
    //sprintf(param, "{ \"temp\":[{\"v\":%.2f}],\"humi\":[{\"v\":%.2f}] }", temp, humi); //我们把要上传的数据写在param里
    sprintf(param, "{ \"temp\":[{\"v\":%.2f}] }", temp); //我们把要上传的数据写在param里
    postMsgId += 1;
    sprintf(jsonBuf, ONENET_POST_BODY_FORMAT, postMsgId, param);
    //再从mqtt客户端中发布post消息
    if (client.publish(ONENET_TOPIC_PROP_POST, jsonBuf))
    {
      Serial.print("Post message to cloud: ");
      Serial.println(jsonBuf);
    }
    else
    {
      Serial.println("Publish message to cloud failed!");
    }
  }

}


//重连函数, 如果客户端断线,可以通过此函数重连
void clientReconnect()
{
  while (!client.connected()) //再重连客户端
  {
    Serial.println("reconnect MQTT...");
    if ( client.connect(mqtt_devid, mqtt_pubid, mqtt_password) )
    {
      Serial.println("connected");
    }
    else
    {
      Serial.println("failed");
      Serial.println(client.state());
      Serial.println("try again in 5 sec");
      delay(5000);
    }
  }
}


void callback(char *topic, byte *payload, unsigned int length)
{
  Serial.println("message rev:");
  Serial.println(topic);
  for (size_t i = 0; i < length; i++)
  {
    Serial.print((char)payload[i]);
  }
  Serial.println();
 
  if (strstr(topic, ONENET_TOPIC_PROP_SET))
  {
    DynamicJsonDocument doc(100);
    DeserializationError error = deserializeJson(doc, payload);
    if (error)
    {
      Serial.println("parse json failed");
      return;
    }
    JsonObject setAlinkMsgObj = doc.as<JsonObject>();
    serializeJsonPretty(setAlinkMsgObj, Serial);
    String str = setAlinkMsgObj["id"];
    Serial.println(str);
    char sendbuf[100];
    sprintf(sendbuf, "{\"id\": \"%s\",\"code\":200,\"msg\":\"success\"}", str.c_str());
    Serial.println(sendbuf);
    client.publish(ONENET_TOPIC_PROP_SET_REPLY, sendbuf);
  }
  if (strstr(topic, ONENET_TOPIC_PROP_GET))
  {
    DynamicJsonDocument doc(100);
    DeserializationError error = deserializeJson(doc, payload);
    if (error)
    {
      Serial.println("parse json failed");
      return;
    }
    JsonObject setAlinkMsgObj = doc.as<JsonObject>();
    serializeJsonPretty(setAlinkMsgObj, Serial);
    String str = setAlinkMsgObj["id"];
    Serial.println(str);
    char sendbuf[100];
    //sprintf(sendbuf, "{\"id\": \"%s\",\"code\":200,\"msg\":\"success\",\"data\":{\"temp\":%.2f,\"humi\":%.2f}}", str.c_str(), temp, humi);
    sprintf(sendbuf, "{\"id\": \"%s\",\"code\":200,\"msg\":\"success\",\"data\":{\"temp\":%.2f }}", str.c_str(), temp);
    Serial.println(sendbuf);
    client.publish(ONENET_TOPIC_PROP_GET_REPLY, sendbuf);
  }
}


void update_date(void)
{
  temp = sensors.getTempCByIndex(0);
  humi++;
  dis++;
  }

 
 

void setup() {
  // put your setup code here, to run once:


  Serial.begin(115200); //初始化串口
  sensors.begin();
  //Wire.begin();
  //sht20.begin();
  delay(3000);
  setupWifi();                                           //调用函数连接WIFI
  client.setServer(mqtt_server, port);                   //设置客户端连接的服务器,连接Onenet服务器, 使用1883端口
  delay(2000);
  Serial.println("setServer Init!"); 
  //client.setCallback(callback);
  client.connect(mqtt_devid, mqtt_pubid, mqtt_password); //客户端连接到指定的产品的指定设备.同时输入鉴权信息
  delay(2000);
  Serial.println("connect Init!");
  
  if (client.connected())
  {
    Serial.println("OneNet is connected!"); //判断以下是不是连好了.
  }
  client.subscribe(ONENET_TOPIC_PROP_SET);
  client.subscribe(ONENET_TOPIC_PROP_GET);
  client.setCallback(callback);
  tim1.attach(8, sendTempAndHumi); //定时每20秒调用一次发送数据函数sendTempAndHumi
  tim2.attach(4, update_date);
  temp =10;
  humi =20;
  dis = 100;
}
int count = 0;

void loop() {
  // put your main code here, to run repeatedly:
  sensors.requestTemperatures(); 
  float temp = sensors.getTempCByIndex(0);
  if (!WiFi.isConnected()) //先看WIFI是否还在连接
  {
    setupWifi();
  }
  if (!client.connected()) //如果客户端没连接ONENET, 重新连接
  {
    clientReconnect();
    delay(100);
  }
  client.loop(); //客户端循环检测
}


