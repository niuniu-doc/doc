---
title: postman添加sign签名
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---

#### 安装postman插件
```
https://www.jianshu.com/p/451e0d009304
```


#### 常用功能使用
```
https://www.jianshu.com/p/15f8dfaaecef
```


#### 生成sign签名
```
1. 编写脚本 
   以order项目为例(下边sign.js)


2. 在请求中添加脚本
   managerEnvirenment -> add -> sign (值可以随便给、这个是用来给postman使用的、在发送请求时会替换为计算出来的sign值)
   如下图所示


3. 发送请求、可以发现签名是ok的


4. 如果发现签名不对、可以调试脚本
   在chrome地址栏中输入：chrome://flags/#debug-packed-apps ，开启Debugging for packed app
   输入chrome://inspect/#apps，选择postman的inspect  会弹出调试框
   
   在弹出的调试框里、选择elements在下边可以看到console
   可以查看签名的方式是否正确(配合idea调试、对比签名方式和签名结果)
   idea调试：以debug模式运行、
   在指定合适的位置打断点、代码逻辑运行到断点处即会被中断掉、可以进行单步调试~~
   查找问题出现的原因
   
```




#### sign.js
```js
var appkey = '6615be7b44ca4ab9ac03060088202792'; // 自动化测试的key
 //获取当前时间
 function createTime() {
     return (new Date()).valueOf();
 }
 var time = createTime();
 var method = request.method;
 delete request.data["sign"];
 console.log("request data is : " + request.data);
 var keys = Object.keys(request.data), i, len = keys.length;
 keys.sort();
 console.log("sortedKeys is : " + keys)
 // Build the request body string from the Postman request.data object
 var requestBody = "";
 var firstpass = true;

 // 构造数据为 key=param&key=param....字符串
 for(var index in keys){
 if (keys[index] == "sign") {
 	continue;
 }
if(!firstpass){
    requestBody += "&";
}
        
if(keys[index]=="create_time"){
    request.data[keys[index]]=time;
    console.log(request.data[keys[index]]);
    }
    requestBody += keys[index] + "=" + request.data[keys[index]];
    firstpass = false;
}
requestBody += '&key=' + appkey;   
console.log("request body is : " + requestBody);

var md5=CryptoJS.MD5(requestBody, appkey);
var base64md5 = CryptoJS.enc.Base64.stringify(md5);
console.log(base64md5);
postman.setEnvironmentVariable('sign', base64md5); // 将变量放入postman 变量中;



```
图一、添加postman环境变量
![image.png](https://upload-images.jianshu.io/upload_images/14027542-5337c8380fbe226e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-323bcf356a646c1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-2cf8d68e2ad29999.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


图二、postman脚本调试
![image.png](https://upload-images.jianshu.io/upload_images/14027542-6f2ffb4315989907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-066f5d87bcf85312.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/14027542-5e0888d32e2590c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
