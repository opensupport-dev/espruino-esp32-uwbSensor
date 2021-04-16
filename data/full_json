if (ESP32.getState().Wifi==false){ESP32.enableWifi(true);}
/////// Variables SECTION ////////////////////////
var flash=require("fs");
var wifi=require('Wifi');
var ScanAP=[];
var Gain=0,Test1=0,Test2=0;
var T={value1:0,value2:0};
var z=null;
var troubles=0;
var mqtt_troubles=0;
var data_post={};
var ajax={device:"",date:"",time:"",move:"",breath:""};
//var pem="";
var f={
    protocol: 'mqtt',
    web_adr: 'http://api.thingspeak.com/update',
    web_apikey:'7RDU43OROG715QAA',
    ap_ssid:'',
    ap_password: '',
    ap_mode: 'open',
    ap_timeout: 15,
    ap_port: 80,
    st_ssid: 'KT_GiGA_2G_Wave2_397D',
    st_password: '6db80dd238',
    mqtt_server: "test.mosquitto.org",
    mqtt_active: false,
    mqtt_options: {keep_alive: 60,port: 1883,username: "",password: "" },
    mqtt_topic: "umain/sensor/data",
    rs485_speed: 9600,
    rs485_echo: -1
    };
//flash.writeFileSync("setup.ini", JSON.stringify(f));
///// IRQ SECTION /////////////////////////////
wifi.on('sta_left', function() {console.log('AP was disconnected...', wifi.getIP().ip);});
function LaunchAP() {
  var options={};
  if (f.ap_password!='') options ={password: f.ap_password,authMode: f.ap_mode};
    wifi.startAP(f.ap_ssid,{}, function(err) {if (err) throw err;
      console.log("My embeded AP start!");
  });
}
function getScanAP(ListAP){
  var s='';
  for (var prop in ListAP){
    if (ListAP[prop].ssid!=''){s+=ListAP[prop].ssid.toString()+',';}
  }
  s=s.substring(0,s.lastIndexOf(','));
  ScanAP=s.split(',');
  console.log("Got AP list...");
}
function Scan(){
  try{wifi.scan( getScanAP );} catch (e) { console.log('Not avalable to scan Wi-fi'); }
}
function ToRS485(data){
  D12.write(1);
  var delay =data.lastIndexOf('')* (12 / f.rs485_speed) * 1000 + 2;
  delay=Math.ceil(delay);
  setTimeout(function() {D12.write(0);}, delay);
  Serial2.print(data);
}
function ToJSON(postURL, payload, callback){
  var s=JSON.stringify(payload);
  var options = url.parse(postURL);
  options.method = 'POST';
  options.headers = { "Content-Type":"application/json", "Content-Length":s.length};
  if (wifi.getStatus().station=='connected'){
    var req = require("http").request(options, function(res)  {
      var d = "";
      res.on('data', function(data) { d+= data; });
      res.on('close', function(data) { callback(d); });
    });
    req.on('error', function(e) {
      callback();
    });
    req.end(s);
    }
  }
function ToThinkspeak(ch){
  var data='';//"&field1="+value1+"&field2="+value2;
  if (ch.value1 != undefined) {data+="&field1="+ch.value1.toString();}
  if (ch.value2 != undefined) {data+="&field2="+ch.value2.toString();}
  if (ch.value3 != undefined) {data+="&field3="+ch.value3.toString();}
  if (ch.value4 != undefined) {data+="&field4="+ch.value4.toString();}
  if (ch.value5 != undefined) {data+="&field5="+ch.value5.toString();}
  if (ch.value6 != undefined) {data+="&field6="+ch.value6.toString();}
  if (ch.value7 != undefined) {data+="&field7="+ch.value7.toString();}
  if (ch.value8 != undefined) {data+="&field8="+ch.value8.toString();}
   //  headers: {"Content-Type":"application/json","Content-Length":data.length } 
  if ((data !='')||(wifi.getStatus().station!='connected')){
   data=f.web_adr+((f.web_apikey=='') ? "?":("?api_key="+f.web_apikey))+data;
    var options=url.parse(data, true);
        if (options.protocol=="https"){
          options.cert="certificate.cer";}
   // console.log(options);
    require("http").get(options, function(res) {console.log(data);});
  }
}

function ToMQTT(ch){
 if (f.mqtt_active==true){ mqtt.publish(f.mqtt_topic, JSON.stringify(ch));
  console.log('mqtt: '+f.mqtt_topic+' '+JSON.stringify(ch));}
  else{ mqtt_troubles++;
 // mqtt.connect(mqtt.publish(f.mqtt_topic, JSON.stringify(ch)));
  }
}
function FromPOST(req,res) {
  var data = "";
  req.on('data', function(d) { data += d; });
  req.on('end', function() {
    var start=data.indexOf('-----BEGIN CERTIFICATE-----');
    var stop=data.indexOf('-----END CERTIFICATE-----');
    if ((start>0)&(stop>start))
    {
      var pem=data.substring(start+27,stop);
      const reg= /\r\n/gi;
      pem=pem.replace(reg, '');
      var sr=atob(pem);
//      var st="";
//      for (var prop in sr){st+=sr.charCodeAt(prop).toString(16)+',';}
//      console.log(st);
      if(flash.writeFileSync("certificate.cer", sr)) {
        console.log('New sertificate data saved...');
        WebMsg(res,'SAVE FILE','New sertificate data saved!');
      }else{ WebMsg(res,'ERROR','Cant save file to flash!'); }
      console.log(pem);
    }else{
      data_post = {};
      data.split("&").forEach(function(el) {
        var els = el.split("=");
        data_post[els[0]] = decodeURIComponent(els[1]);
      });
           console.log(data_post);
      if (data_post.ap!= undefined) f.st_ssid=data_post.ap.toString();
      if (data_post.ps!= undefined) f.st_password=data_post.ps.toString();
      if (data_post.ak!= undefined) f.web_apikey=data_post.ak.toString();
      if (data_post.sr!= undefined) f.web_adr=data_post.sr.toString();
      if (data_post.sd!= undefined) f.rs485_speed=parseInt(data_post.sd.toString(),10);
      if (data_post.eh!= undefined) f.rs485_echo=parseInt(data_post.eh.toString(),10);
      if (data_post.mp!= undefined) f.mqtt_options.port=parseInt(data_post.mp.toString(),10);
      if (data_post.ms!= undefined) f.mqtt_server=data_post.ms.toString();
      if (data_post.mu!= undefined) f.mqtt_options.username=data_post.mu.toString();
      if (data_post.qp!= undefined) f.mqtt_options.password=data_post.qp.toString();
      if (data_post.tk!= undefined) f.mqtt_topic=data_post.tk.toString();
      if (data_post.wp!= undefined) f.ap_port=parseInt(data_post.wp.toString(),10);
      if (data_post.as!= undefined) f.ap_ssid=data_post.as.toString();
      if (data_post.sl!= undefined) f.protocol=data_post.sl.toString();
      if (data_post.to!= undefined) f.ap_timeout=parseInt(data_post.to.toString(),10);
      res.writeHead(200, {'Content-Type': 'text/html'});
      if(flash.writeFileSync("setup.ini", JSON.stringify(f))) {
        console.log('New data saved...',f);
        WebSetup(res);
      }else{ WebMsg(res,'ERROR','Cant save new data');}
    }
  });
}
///////////////////////////////////////////////////
try {
  flash.readdirSync();
 } catch (e) { //NO_FILESYSTEM
  console.log('Formatting FS - only need to do once');
  E.flashFatFS({ format: true });
  //flash.writeFileSync("setup.ini", JSON.stringify(f));
}
try {
  f=JSON.parse(flash.readFileSync("setup.ini"));
 } catch (e) { 
  console.log('Not found setup.ini');
 }
try {
  flash.readFileSync("certificate.cer");
 } catch (e) { 
  console.log('Not found certificate.cer');
 }
  wifi.restore();
  wifi.stopAP();
f.mqtt_active=false;
if (f.ap_ssid=='') f.ap_ssid='UWB_Sensor_'+wifi.getIP().mac.toString().substring(12,17).replace(':','').toUpperCase();
//  Scan();
  setTimeout(function () {LaunchAP();Scan();}, 15000);
    if (f.ap_timeout>0)  setTimeout(function () { wifi.stopAP();}, f.ap_timeout*60000);
    setInterval(function(){console.log(require('Wifi').getStatus().station + ':'+ require('Wifi').getStatus().ssid, require('Wifi').getAPDetails().ssid +':'+require('Wifi').getStatus().mode);
    if (require('Wifi').getStatus().station!='connected') {troubles++;}else{
     // if ((f.mqtt_active==false)&(f.protocol=='mqtt'))mqtt.connect();
    }
    if (troubles>10) {console.log('DetEct trouble, need reboot!'); ESP32.reboot();}
    if ((mqtt_troubles>100)&(f.protocol=='mqtt')) {
      console.log('DetEct MQTT trouble, need reconnect!'); mqtt.disconnect();mqtt_troubles=0;troubles++;}
  },10000);
//Math.floor((1 + Math.random()) * 0x10000).toString(16).substring(1);
var o=JSON.parse(JSON.stringify(f.mqtt_options));
if (o.username=='') delete o.username;
if (o.password=='') delete o.password;
var mqtt = require("MQTT").create(f.mqtt_server, o);
if (f.protocol=='mqtt'){
 if (wifi.getStatus().station=='connected'){ mqtt.connect();} else {setTimeout(function() {mqtt.connect();}, 15000);}
}
/////////////////////////////////////////////////////////////////////////////////
mqtt.on('connected', function() {
   console.log("MQTT is connected!");
   f.mqtt_active=true;
  mqtt_troubles=0;
  //mqtt.subscribe("#",0);
 // mqtt.subscribe(test_topic);
  mqtt.subscribe(f.mqtt_topic);
  });
mqtt.on('disconnected', function() {
  console.log("Disconnected MQTT...");
   f.mqtt_active=false;
  mqtt_troubles++;
  setTimeout(function() {mqtt.connect();}, 5000);
});
  mqtt.on('publish', function (pub) {
    console.log("topic: "+pub.topic);
    console.log("message: "+pub.message);
   /* if (pub.topic==test_topic) {
      setInterval(function(){var message = "hello, world";console.log('>>');mqtt.publish(my_topic, message);},5000);
     console.log("my topic...");
      mqtt.subscribe(f.mqtt_topic);} */
  });
function onPageRequest(req, res) {
  var a = url.parse(req.url, true);
  if (req.method=="POST"){ FromPOST(req,res);}
  if (a.pathname=="/") {
    if (req.method=="GET"){
      res.writeHead(200, {'Content-Type': 'text/html'});
      WebSetup(res);
    }
  } else if (a.pathname=="/connect") {
    wifi.disconnect(function() {
    wifi.connect( f.st_ssid, {password: f.st_password}, function(err) {
       res.writeHead(200, {'Content-Type': 'text/html'});
      if (err) {
        console.log("Not connected to Wifi.  Error: "+err.toString());
        WebMsg(res,'ERROR','Not connected to Wifi point ('+err.toString()+')');}
      else{
        console.log('Connected to Wifi.  IP address is:', wifi.getIP().ip);
       WebMsg(res,'Wi-Fi connect','Device connected to AP point <b>'+wifi.getDetails().ssid+'</b>,<br>IP address is: '+wifi.getIP().ip);
      wifi.save(); // Next reboot will auto-connect
      }
    });});
  } else if (a.pathname=="/reboot") {
    res.writeHead(200, {'Content-Type': 'text/html'});
     WebMsg(res,'REBOOT','System [SDK: '+ESP32.getState().sdkVersion+'] will restart in few seconds... ');
    setTimeout(function(){ESP32.reboot();},2000);
   } else if (a.pathname=="/certificate") {
     console.log(a);
      } else if (a.pathname=="/post.cgi") {
     console.log('/post.cgi',req,a);
  } else {
    res.writeHead(404, {'Content-Type': 'text/plain'});
    res.end("404: Page "+a.pathname+" not found");
  }
      Scan();
}
function WebSetup(res) {
  var S1='',S2='',S3='';
  res.write('<!DOCTYPE html><html><head><meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />');
  res.write('<style type="text/css"> body { font-size: 75%; font-family: "Open Sans", sans-serif;  color: #444;  margin:0;  background-color: #F3F3F3;}');
  res.write('.dex, .xed { display: none;}');
  res.write('.dx1:checked ~ .dex {display: block;}');
  res.write('.dx2:checked ~ .xed {display: block;}');
  res.write('label { margin-left:10px; margin-right:15px; color: black;}');
  res.write('span { padding: 7px 10px; text-align: center; color: #336699; font-weight: bold;}');
  res.write('input, select, button{  background-color: white;  border-radius: 5px;  padding: 5px 10px;}');
  res.write('div{background-color: #EFEFEF;border: 2px solid #336699;width: 90%;height: 70%;padding: 10px;margin: 3% auto;border-radius: 9px;');
  res.write('  -moz-border-radius: 9px; -o-border-radius: 9px; -webkit-border-radius: 9px;-moz-box-shadow:0 0 50px #555;-webkit-box-shadow:0 0 50px #555;box-shadow:0 0 50px #555;}');
  res.write('</style></head><body><div><svg id="L1"  viewBox="0 0 400 400"  viewBox="0 0 100 100" x="10px" y="10px" width="90px" height="50px" ><g id="IcoLog" fill="SkyBlue">');
  res.write('<path " d="M331,174c-30-19-66-30-103-30 c-37,0-73,10-103,30c-29,19-52,46-63,77c-2,6,1,13,7,15c6,2,13-1,15-7c20-54,77-91,143-91c66,0,124,36,143,91 c2,5,6,8,11,8c1-0,3-0,4-1c6-2,9-9,7-15 C383,220,361,193,331,174z"/>');
  res.write('<path " d="M291,236c-17-14-39-21-63-21 c-23,0-46,8-63,21c-17,14-28,33-30,54c-1,7,4,12,11,13c0,0,1,0,1,0c6,0,11-5,12-11c3-30,33-54,69-54 c36,0,66,24,69,54c1,7,6,11,13,11c7-1,11-6,11-13 C319,270,308,250,291,236z"/>');
  res.write('<path " d="M228,280c-25,0-46,21-46,46c0,25,21,46,46,46 c25,0,46-21,46-46C274,300,253,280,228,280z M228,348c-12,0-22-10-22-22c0-12,10-22,22-22c12,0,22,10,22,22 C250,338,240,348,228,348z"/></g></svg>');
  res.write('<span><a>Seting Wi-Fi connect</a></span><form id="CN" action="/connect" method="get"></form><form id="REBOOT" action="/reboot" method="get"></form>');
  res.write('<form enctype="multipart/form-data" id="FILE" method="post" action="/certificate"></form><form id="SAVE" action="/" method="post"></form><p><label>Name  AP</label><input id="ap" list="ssid" form="SAVE" name="ap" value="'+f.st_ssid+'" /><datalist id="ssid">');
   for (var prop in ScanAP){S2=ScanAP[prop].toString(); if (S2==f.st_ssid) {S1='--> '+S2;} else{S1='';}
     if(S2!=''){ res.write('<option value="'+S2+'">'+S1+'</option>');}}
//
  S1='';S2='';
  res.write('</datalist></p><p><label>Password</label><input type="password" id="pw"  form="SAVE" name="ps" size="10" value="'+f.st_password+'"/><input type="submit"  form="CN" value="Connect"/></p>');
 res.write('<br><label>MAC: '+require('Wifi').getIP().mac+'</label><br><label>IP:&ensp;&emsp;'+require('Wifi').getIP().ip +'</label><br><label>Status: '+require('Wifi').getStatus().station+' </label>');
  //
  if (f.protocol=='mqtt') {S1='checked';}else{S2='checked';}
  res.write('<hr><label>Main server protocol </label><input type="radio" class="dx1" id="sx" form="SAVE" name="sl" '+S1+' value="mqtt"><label>MQTT  </label><input type="radio" class="dx2" id="sy" name="sl" form="SAVE" '+S2+' value="http"><label>HTTP</label><hr>');
  S1='';S2='';
  res.write('<section class="dex"><p><label>Server MQTT</label><input type="text" id="ms"  form="SAVE" name="ms" value="'+f.mqtt_server+'"/></p><p><label>Topic MQTT&ensp;</label><input type="text" id="tk"  form="SAVE" name="tk" value="'+f.mqtt_topic+'"/></p>');
   res.write('<p><label>Port MQTT&emsp;</label><input type="text" id="mp"  form="SAVE" name="mp" value="'+f.mqtt_options.port.toString()+'"/></p>');
  res.write('<p><label>User Name &ensp;</label><input type="text" id="mu"  form="SAVE" name="mu" value="'+f.mqtt_options.username.toString()+'"/></p><p><label>Password &emsp;</label><input type="text" id="qp"  form="SAVE" name="qp" value="'+f.mqtt_options.password.toString()+'"/></p></section>');
  //
  res.write('<section class="xed"><p><label>Link&ensp;</label><input type="text" id="sr"  form="SAVE" name="sr" value="'+f.web_adr+'" size="25"/></p><p><label>API#</label><input type="text" id="ak"  form="SAVE" name="ak" value="'+f.web_apikey.toString()+'"/></p>');
  //
  //
   res.write('<p><label>TLS Certificate</label><input type="file" form="FILE" name="files" /></p><p><input type="submit" form="FILE" value="Send" /></p></section>');
  //
  //
   S1='';S2='';S3='';
  if(f.ap_timeout==15) {S1='selected';}else if(f.ap_timeout==0){S2='selected';}else{S3='selected';}
  res.write('<hr><p><label>Shutdown &emsp;&ensp;</label><select id="sd" form="SAVE" name="to" value="'+f.ap_timeout.toString()+'"><option '+S3+' value="5">5 min</option><option '+S1+' value="15">15 min</option><option   '+S2+' value="0">Never</option></select>');
   S1='';S2='';S3='';
  //
  if(f.rs485_speed==115200) {S1='selected';}else if(f.rs485_speed==19200){S2='selected';}else{S3='selected';}
  res.write('<hr><p><label>RS485 Set/Echo</label><select id="sd" form="SAVE" name="sd" value="'+f.rs485_speed.toString()+'"><option '+S3+' value="9600">9600</option><option  '+S2+' value="19200">19200</option><option '+S1+' value="115200">115200</option></select>');
   S1='';S2='';S3='';
  if(f.rs485_echo==1) {S1='selected';}else if(f.rs485_echo==0){S2='selected';}else{S3='selected';}
  res.write('<select id="eh" form="SAVE" name="eh" value="-1"><option '+S3+' value="-1">No</option><option '+S2+' value="0">@</option><option '+S1+' value="1">All</option></select><hr><input type="submit"  form="REBOOT" value="Reboot"/><input type="submit"  form="SAVE" value="Save"/>');
  res.end('</div></body></html>');
}
function WebMsg(res,name,msg) {
   res.write('<!DOCTYPE html><html><head><meta http-equiv="refresh" content ="10; url=http://'+wifi.getAPIP().ip+'/">');  res.write('<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" /><style type="text/css">');
  res.write('body { font-size: 100%; font-family: "Open Sans", sans-serif;  color: #444;  margin:0;  background-color: #F3F3F3;}');
  res.write('label { margin-left:10px; margin-right:15px; color: black;}');
  res.write('span { padding: 7px 10px; text-align: center; color: #336699; font-weight: bold;}');
  res.write('input, select, button{  background-color: white;  border-radius: 5px;  padding: 5px 10px;}');
  res.write('div{background-color: #EFEFEF;border: 2px solid #336699;width: 90%;height: 40%;padding: 10px;margin: 3% auto;border-radius: 9px;');
  res.write('  -moz-border-radius: 9px; -o-border-radius: 9px; -webkit-border-radius: 9px;-moz-box-shadow:0 0 50px #555;-webkit-box-shadow:0 0 50px #555;box-shadow:0 0 50px #555;}');
  res.end('</style></head><body><div><span><a>'+name+'</a></span><hr><form id="B" action="/" method="get"><p><label>'+msg+'</label></p><p><input type="submit" value="Ok" /></p></form></div></body></html>');
}

D12.mode('output');  //RS-485 control
D12.write(1);
Serial2.setup(f.rs485_speed, { tx: D17, rx: D16 });
Serial2.on('data', function(data) {
  console.log('Serial2: ', data);
  var s=data.split(',');
 if (isNaN(parseInt(s[0],10))){T.value1=undefined;} else {T.value1=parseInt(s[0],10);}
 if (isNaN(parseInt(s[1],10))) {T.value2=undefined;} else {T.value2=parseInt(s[1],10);}
 if (f.protocol=='mqtt'){ ToMQTT(T);} else{
 var xs=true;
   if (f.web_adr!= 'http://api.thingspeak.com/update1')
   {
     var t= new Date();
     ajax.device=f.ap_ssid;
     ajax.date=t.getFullYear().toString()+"-"+t.getMonth().toString()+"-"+t.getDate().toString();
     ajax.time=t.getHours().toString()+":"+t.getMinutes().toString()+":"+t.getSeconds().toString();
     ajax.move=T.value1;
     ajax.breath=T.value2;
     console.log('ajax-->JSON: ', JSON.stringify(ajax));
     ToJSON("http://umain.codexbridge.com/api/index.php/app/sendData",ajax,function(d) {
            console.log("ResponseJSON: "+d);
            });
   } else { try{ToThinkspeak(T);}catch(e){xs=false;console.log(e.toString());}
  }
  if ((f.rs485_echo!=-1)&(xs==true)){ if(f.rs485_echo==0){ToRS485('@');}else{ToRS485(data);}}
 }
});

//setInterval( function() {ToRS485('@',1);},200);
const server=require("http").createServer(onPageRequest);
server.listen(f.ap_port);
////////////////////////////////////////////////////////////////////////////
console.log("End of sketch...");