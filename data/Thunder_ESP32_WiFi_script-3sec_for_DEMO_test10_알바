var wifi_state = ESP32.getState(); // WiFi상태
if (wifi_state.Wifi==false){ ESP32.enableWifi(true); }
wifi_state = ESP32.getState();
console.log(wifi_state);

/// GPIO CONFIG //////////////////////////////////
D32.mode('output');// Cloud
D33.mode('output');// Link 
D25.mode('input'); // Setup knob
D12.mode('output');// RS-485 control
D32.write(0);
D33.write(0);
/////// Variables SECTION ////////////////////////
var fs = require('fs');
var wifi=require('Wifi');
var ScanAP=[];
var T={value1:0,value2:0};
var TT={n:0, h:0, m:[],b:[]};
//var z=null;
var troubles=0;
var ajax={device:"",date:"",time:"",move:"",breath:""}; // 그냥 json형식 데이터 발송을 위한 구조
var mqtt=null;
var setup=false;

var b_flag = 1;// 0 is normal, 1 is warning
var m_flag = 1;// 0 is normal, 1 is warning

var b_cnt = 0;
var pre_b = 0; 

var set_warning_time;
var warning_time;

var ab_cnt = 0;
var ab_flag = 0;

var f={
    protocol: 'http',//'mqtt',
    web_adr: 'http://umain.codexbridge.com/api/index.php/app/sendData',//'http://api.thingspeak.com/update',
    web_apikey:'7RDU43OROG715QAA',
    ap_ssid:'',
    ap_password: 'q1234567',
    ap_mode: 'wpa',
    ap_timeout: 10,
    ap_port: 80,
    st_ssid: 'KT_GiGA_2G_Wave2_397D',
    st_password: '6db80dd238',
    mqtt_server: "mqtt서버",
    mqtt_active: false,
    mqtt_options: {client_id : "aaaaaaaa",keep_alive: 60,port: 1883,username: "",password: "" }, // mqtt에 사용되는 기본 설정, client_id를 새로생성해야합니다. 예)80z_맥주소 이렇게 하면 좋겠네요.
    mqtt_topic: "umain/sensor/data",
    mqtt_subscribe: "umain/sensor/cmd",
    mqtt_timeout: 3,
    distance:  35,
    update_interval: 0
    };

const rs485_speed = 9600;


var block_communications = 0;

//var request_ping = "\u006a\u0000\u00c2\u0000\u000e";
//이부분은 저희 레이더 센서 동작에 대한 기본 설정 값입니다.
const request_update_type =         "\u006a\u0000\u00ca\u0000\u00a6";
const request_update_by_event =     "\u006a\u0000\u00ca\u0001\u00a1";
const request_update_by_interval =  "\u006a\u0000\u00ca\u0002\u00a8";
const request_interval_val =        "\u006a\u0000\u00cc\u0000\u00d8";
const request_update_int_preamb = "\u006a\u0000\u00cc";
const request_dist_preamb = "\u006a\u0000\u00c4";
const request_dist_val = "\u0000\u0070";


var distance_setup = 0;

//var fr=null;
var req_distance = 0;
//var tmp_val = 0;
var no_reset_cnt = 19;
var turn_on_time = "";
var send_type_is_known = 0;
var busy = 0;

var m,l,c,s,ss,ap;
var state;
var errors;

/////////////////////////////////////////////////////////////////////////////////
wifi.getIP(function(err,data)
           {
              console.log("-----------------------------------");
              console.log("My MAC address: ",data.mac);
              console.log("-----------------------------------");
            });
			
//wifi.stopAP();

// 아래부분은 아시겠지만, 설정을 저장하거나 기존 저장되어있는 설정을 읽어옵니다.
try {
  fs.readdirSync();
 } catch (e) { //'Uncaught Error: Unable to mount media : NO_FILESYSTEM'
  console.log('Formatting FS'); // need to do once
  E.flashFatFS({ format: true });
  fs.writeFileSync("setup.ini", JSON.stringify(f));
}

 
try { f = JSON.parse(fs.readFileSync("setup.ini")); } 
catch (e) 
{ 
	console.log('Not found setup.ini');
	setup=true;
}

//try { fs.readFileSync("certificate.cer"); } 
//catch (e) { console.log('Not found certificate.cer'); }

if (D25.read()==0) { setup = true; console.log('Detect knob -SETUP'); }

var o = JSON.parse(JSON.stringify(f.mqtt_options));
if (o.username=='') delete o.username;
if (o.password=='') delete o.password;
mqtt = require("MQTT").create(f.mqtt_server, o);
f.mqtt_active=false;

// mqtt 연결확인하는 부분
if (require('Wifi').getStatus().station=='connected')
{ 
  if (f.protocol==='mqtt')
  {
    mqtt.connect();
  }
  else
  {
    mqtt.disconnect();
  }
}
else{
      wifi.connect(f.st_ssid, {password: f.st_password}, 
                   function(ap)
                   { 
						console.log("Wi-Fi connected to", ap);
                        if (f.protocol==='mqtt')
                        {
                          console.log('Start MQTT...',mqtt);
                          mqtt.connect(); 
                        }
                        else 
                        {
                          console.log('Protocol is ',f.protocol);
                          mqtt.disconnect();
                        }
                    });
}


if (f.ap_ssid=='') f.ap_ssid = 'UWB_Sensor_'+wifi.getIP().mac.toString().substring(12,17).replace(':','').toUpperCase();

//Launch web interface 
if (setup==true) {
	setTimeout(function () {LaunchAP();Scan();}, 15000); // AP리스트에서 자꾸 사리지는 부분이 여기인지.... 모르겠네요. 
	if (f.ap_timeout>0) setTimeout(function () { wifi.stopAP();}, f.ap_timeout*60000); 
}


D12.write(1);
Serial2.setup(rs485_speed, { tx: D17, rx: D16 });

// WIFI IRQ SECTION ------------------------------ 
wifi.on('sta_left', 	function() 			{block_communications = 0; console.log('AP was disconnected...', wifi.getIP().ip);});
wifi.on('associated', 	function(details) 	{console.log('STA associaated as',details );});
wifi.on('auth_change', 	function(details) 	{console.log('STA authorization is change',details ); });
wifi.on('dhcp_timeout', function() 			{troubles++;console.log('DHCP timeout' ); });
wifi.on('sta_joined', 	function(details) 	{
                                              console.log('Wireless was joined ',details ); 
                                              block_communications = 1;
                                              if (f.protocol === 'mqtt') { mqtt.disconnect(); }
                                              setTimeout( function() { block_communications = 0; }, 300000); // Back to normal after 5 minutes
                                            });
wifi.on('disconnected', function(details) 	{
												troubles++;
												console.log('STA was disconnected ',details );
												if ( (mqtt!=null) & (require('Wifi').getStatus().mode=="STA") )	
												{
													setTimeout(function()
													{ 
														wifi.connect(f.st_ssid, {password: f.st_password}, function(ap){ 
																												console.log("Wi-Fi reconnected"); 
																												if (mqtt!=null)  mqtt.connect(); 
																												}
																	);
													},2000);
												}
						}
		);


// MQTT IRQ SECTION ------------------------------ 123라인에의해 생성된 mqtt 함수들
mqtt.on('connected', function() {
						console.log("MQTT is connected!");
						//mqtt.subscribe("#",0);
						//mqtt.subscribe(test_topic);
						if (f.protocol === 'mqtt' && ( block_communications === 0) && (busy === 0)){
                          mqtt.subscribe(f.mqtt_subscribe);
                        }
					}
		);

mqtt.on('disconnected', function() {
							console.log("Disconnected MQTT...");
                            if (f.protocol === 'mqtt' && ( block_communications === 0) && (busy === 0)){
							setTimeout(function() { mqtt.connect(); }, 5000);
                            }
						}
		);
		
mqtt.on('publish', function (pub) {
						console.log("topic: "+pub.topic);
						console.log("message: "+pub.message);
						//if (pub.topic==f.mqtt_subscribe) ToRS485(pub.message);
                        if (f.protocol === 'mqtt' && ( block_communications === 0) && (busy === 0)){
						  if (pub.topic==f.mqtt_subscribe)  console.log("Send by RS485: "+pub.message);
                        }
					}
		);
		
mqtt.on('message', function (msg) {
						console.log("subscribed topic: ",msg);
						//ToRS485(msg.message.toString());
					}
		);
		
mqtt.on('subscribed', function () {
						console.log("Topic is subscribed in success...");
						troubles=0;
					}
		);

// Serial IRQ SECTION ----------------------------- esp32와 저희 센서와 uart로 통신하는 부분입니다.	
Serial2.on('data', function(data) {
  busy = 1;
  process.memory();
  var hex_data;
  hex_data = ConvertStringToHex(data);
  //console.log('Serial_data: ', data.slice(0,2), data.slice(0,2)==' j');
  //console.log('Serial_data_hex: ', hex_data.slice(0,6), hex_data.slice(0,6) === "\u006a", hex_data.slice(4,6)==="6a");
  if (hex_data.slice(4,6)==="6a")
  {
    console.log('Serial2_hex: ', hex_data, hex_data.slice(0, 6));
    //\u006a\u000\u00c4\u006\u0062
    ParseSensorData(hex_data.slice(15,17),hex_data.slice(21,22),data[3]);
  }
  else
  {
    if (data.slice(0,6)==='Motion')
    {
      console.log('Header: ', data);
    }
    else
    {
      console.log('Serial2: ', data);
      if(mqtt.connected==true){D32.write(0); setTimeout(function (){D32.write(1);},250);}
      var s=data.split(',');
      if (isNaN(parseInt(s[0],10))){T.value1=undefined;} else {
                                                                if((T.value1=parseInt(s[0],10)) > 0){
                                                                  console.log('Movement: [A]');
                                                                  TT.m.push(T.value1);
                                                                }
                                                                else if((T.value1=parseInt(s[0],10)) == 0){
                                                                  if(T.value2=parseInt(s[1],10) > 0){
                                                                    T.value2=parseInt(s[1],10);
                                                                    console.log('BPM: ', T.value2);
                                                                    TT.b.push(T.value2);
                                                                    b_flag = 1;
                                                                  }
                                                                }
      }
//      if (isNaN(parseInt(s[1],10))) {T.value2=undefined;} else {T.value2=parseInt(s[1],10);TT.b.push(T.value2);}
      if (f.protocol=='mqtt'  && ( block_communications === 0) && (busy === 0)){ 
        ToMQTT(T);
      } 
      else if (f.protocol=='http'  && ( block_communications === 0) && (busy === 0))
      {
          //var xs=true;
          //if (f.web_adr!= 'http://api.thingspeak.com/update1')
          //{
              var t = new Date();
              ajax.device=f.ap_ssid;
              ajax.date=t.getFullYear().toString()+"-"+(t.getMonth()+1).toString()+"-"+t.getDate().toString();
              ajax.time=t.getHours().toString()+":"+t.getMinutes().toString()+":"+t.getSeconds().toString();
              if (turn_on_time === "") turn_on_time = ajax.date + " " + ajax.time;
              ajax.move=T.value1;
              ajax.breath=T.value2;
              console.log('ajax: ', ajax.device, ajax.date, ajax.time);
            try{
              ToJSON(f.web_adr,ajax,function(d) {
                      console.log("ResponseJSON: "+d);
                  });
            }
            catch(e) { troubles++; }

          //}
          //else
          //{
          //    try{ToThinkspeak(T);}catch(e){xs=false;console.log(e.toString());}
          //}
          //if ((f.rs485_echo!=-1)&(xs==true)){ if(f.rs485_echo==0){ToRS485('@');}else{ToRS485(data);}}
      }
      else
      {
        console.log("Temporaly no sending ", data);
      }
    }
 }
  busy = 0;
});	

Serial2.on('parity', function() {
    console.log("S2 parity error!");
  });

Serial2.on('framing', function() {
    console.log("S2 framing error!");
  });

// FUNCTIONS ------------------------------


function str2crc8(in_str, len){
  let crc8 = 0;
  for (let i = 0; i < len; i++) {
      let new_val = parseInt(in_str.charCodeAt(i));
      crc8 = crc8 ^ new_val;
    for (let j = 0; j < 8; j++) {
        if ((crc8 & 128) !== 0) 
        {
          crc8 = (crc8*2)^7;
        }
        else crc8 = crc8*2;
        crc8 = crc8 & 255;
      }
  }
  let input = crc8.toString(16);
  let decimalValue = parseInt(input, 16); // Base 16 or hexadecimal
  let character = String.fromCharCode(decimalValue);
  return character;
}


function ConvertStringToHex(str) {
              var arr = [];
              for (var i = 0; i < str.length; i++) {
                     arr[i] = ("00" + str.charCodeAt(i).toString(16)).slice(-4);
              }
              return "\\u" + arr.join("\\u");
       }

function DistValToInt(str_val)
{
  var tmp = parseInt(str_val);
  if (tmp>0 && tmp<9)
  {
  tmp = 10+(tmp*5);
  }
  else { tmp=35; }
  return tmp;
}

// esp32의 webpage에서 설정된 내용들
function ParseSensorData(cmd, val, char_val){
  console.log("Request to parse cmd and val:", cmd, val);
  if (cmd === 'c4')
  {
    distance_setup = 1;
    f.distance = DistValToInt(val);
    console.log("New distance val", f.distance);
  }
  if (cmd === 'c7')
  {
    console.log("Right crc code", char_val);
  }
  if (cmd === 'ca')
  {
    console.log("Sending type", val);
    if (parseInt(val) === 1) f.update_interval = 0;
	if (send_type_is_known === 1) send_type_is_known = 2;
  }
  if (cmd === 'cc')
  {
    console.log("Interval set to ", val);
    f.update_interval = parseInt(val);
	if (send_type_is_known === 0) send_type_is_known = 1;
  }
}

function SetDistance()//센서 감지거리 조정
{
  console.log('New distance requested',f.distance);
          req_distance = request_dist_preamb;
          let req_val = (f.distance-10)/5;
          req_val = String.fromCharCode(parseInt(req_val.toString(16), 16));
          req_distance = req_distance + req_val;
          let crc_str = str2crc8(req_distance,4);
          req_distance = req_distance + crc_str;
          ToRS485(req_distance);
          console.log('Sent command to sensor',req_distance);
}

function SetInterval()//센서 데이터 주기 조정
{
  if (f.update_interval === 0)
  {
     ToRS485(request_update_by_event);
     console.log('Set update by interval');
  }
  else
  {
    ToRS485(request_update_by_interval);
    let req_str = request_update_int_preamb;
    let req_val = String.fromCharCode(parseInt((f.update_interval).toString(16), 16));
    req_str = req_str + req_val;
    let crc_str = str2crc8(req_str,4);
    req_str = req_str + crc_str;
    setTimeout( function() {  ToRS485(req_str); }, 2000);
    console.log('Sent update by interval ', f.update_interval);
  }
}

function LaunchAP() {
  console.log('1');
 // var ap_options = {};
 // if (f.ap_password !== '') ap_options = {password: f.ap_password, authMode: f.ap_mode}; // phone can't connect when security mode is set
  wifi.startAP(f.ap_ssid,{}, function(err) {if (err) throw err; console.log("My embeded AP start!");});
}

function getScanAP(ListAP){
  console.log('2');
  var s = '';
  for (let prop in ListAP){
    if (ListAP[prop].ssid!=''){s+=ListAP[prop].ssid.toString()+',';}
  }
  s = s.substring(0,s.lastIndexOf(','));
  ScanAP = s.split(',');
  console.log("Got AP list...");
}

function Scan(){
    console.log('3');
  try{
	wifi.scan( getScanAP );
	} 
  catch (e) { console.log('Not avalable to scan Wi-fi'); }
}

function ToRS485(data){
  process.memory();
  D12.write(1);
  var delay = 10* (12 / rs485_speed) * 1000 + 2;
  delay = Math.ceil(delay);
  setTimeout(function() {D12.write(0);}, delay);
  Serial2.print(data);
  console.log('Print to sensor: ', data);
}

function ToJSON(postURL, payload, callback){
  console.log('4');
  process.memory();
  var s = JSON.stringify(payload);
  var options = url.parse(postURL);
  options.method = 'POST';
  options.headers = { "Content-Type":"application/json", "Content-Length":s.length};
  if (wifi.getStatus().station=='connected'){
    var req = require("http").request(options, function(res)  {
												  var d = "";
												  res.on('data', function(data) { d+= data; });
												  res.on('close', function(data) { callback(d); });
												}
								);
    req.on('error', function(e) {callback();});
    req.end(s);
  }
}


function ToMQTT(ch){
 console.log('6');
 if (f.protocol=='mqtt'  && ( block_communications === 0) && (busy === 0))
 {
 var s = JSON.stringify(ch);
 try {
	mqtt.publish(f.mqtt_topic, s);
 } 
 catch (e) { console.log("MQTT error: ",e); trobles++; }
 console.log('mqtt: '+f.mqtt_topic+' '+s);
 }
else console.log('Tmp no send'); 
}

function SaveSetup(res) //설정 저장
{
  busy = 1;
  if (mqtt.connected)
  {
    setTimeout( function() { SaveSetup(res); }, 1000);
  }
  else
  {
        if(fs.writeFileSync("setup.ini", JSON.stringify(f))) {
			console.log('New data saved...');
			WebSetup(res);
		}
		else
		{ 
			console.log("Can't save new data..."); 
			WebMsg(res,'ERROR','Cant save new data, rebooting in few second..');
            setTimeout(function(){ ESP32.reboot(); },3000);
		}
  
  setTimeout( function() { 
    block_communications = 0; 
    if (f.protocol === 'mqtt')
        {
          console.log('Connect MQTT after save data');
          mqtt.connect();
        }
  }, 180000);
  }
  busy = 0;
}

//web설정내용에 대한 post

function FromPOST(req,res) {
    console.log('7');
  busy = 1;
  if (f.protocol === 'mqtt')
  {
    console.log('Disconnect MQTT before save data');
    mqtt.disconnect();
  }
  block_communications = 1;
  troubles = 0;
  process.memory();
  let data = "";
  req.on('data', function(d) { data += d; });
  req.on('end', function() 
  {
    let start=data.indexOf('-----BEGIN CERTIFICATE-----');
    let stop=data.indexOf('-----END CERTIFICATE-----');
    if ((start>0)&(stop>start))
    {
		//let pem=data.substring(start+27,stop);
		//const reg= /\r\n/gi;
		//pem=pem.replace(reg, '');
		//let sr=atob(pem);
		//var st="";
		//for (var prop in sr){st+=sr.charCodeAt(prop).toString(16)+',';}
		//console.log(st);
		//if(fs.writeFileSync("certificate.cer", sr)) {
		//	console.log('New sertificate data saved...');
		//	WebMsg(res,'SAVE FILE','New sertificate data saved!');
		//}
		//else { WebMsg(res,'ERROR','Cant save file to flash!'); }
		//console.log(pem);
        console.log('Part related to https certificate');
    }
	else
	{
		let els;
        let a, b;
		data.split("&").forEach(function(el) {
          els = el.split("=");
          a = els[0]; b = decodeURIComponent(els[1]);

          if (a === 'ap') f.st_ssid     =  b.toString();
          if (a === 'ps') f.st_password =  b.toString();
          if (a === 'ak') f.web_apikey  =  b.toString();
          if (a === 'sr') f.web_adr     =  b.toString();
          //if (a === 'sd') f.rs485_speed=parseInt(b.toString(),10);
          //if (a === 'eh') f.rs485_echo=parseInt(b.toString(),10);
          if (a === 'mp') f.mqtt_options.port      = parseInt(b.toString(),10);
          if (a === 'ms') f.mqtt_server            = b.toString();
          if (a === 'mu') f.mqtt_options.username  = b.toString();
          if (a === 'qp') f.mqtt_options.password  = b.toString();
          if (a === 'tk') f.mqtt_topic             = b.toString();
          if (a === 'ts') f.mqtt_subscribe         = b.toString();
          if (a === 'wp') f.ap_port                = parseInt(b.toString(),10);
          if (a === 'as') f.ap_ssid                = b.toString();
          if (a === 'sl') f.protocol               = b.toString();
          if (a === 'to') f.ap_timeout             = parseInt(b.toString(),10);
          if (a === 'ti') f.mqtt_timeout           = parseInt(b.toString(),10);
          if (a === 'ds') req_distance             = parseInt(b.toString(),10);
          if (a === 'du') 
          {
            let tmp_val = parseInt(b.toString(),10);
            if ((tmp_val >= 0) && (tmp_val<180)) 
            {
              if (tmp_val !== f.update_interval)
              {
                f.update_interval = tmp_val;
                setTimeout( function() { SetInterval(); }, 3000);
              }
            }
          }

		}
		);

		res.writeHead(200, {'Content-Type': 'text/html'});
        E.defrag();
        if (req_distance !== f.distance)
        {
          f.distance = req_distance;
          distance_setup = 0;
          setTimeout( function() { SetDistance(); }, 4500);
        }
        setTimeout( function() { SaveSetup(res); }, 1000);
    }
  });
}

// 시간함수인데, 바꾸셔서 상관없습니다.
function get_server_time()
{
  /*
  var time_addr = "http://worldtimeapi.org/api/timezone/Asia/Seoul";
  if (f.protocol === 'mqtt') { time_addr = f.mqtt_server;} // Can be asked local server but if it returns too big page then memory fault
  if (f.protocol === 'http') { time_addr = f.web_adr; }
  */
  let fail = 0;
  let time_addr = "http://worldtimeapi.org/api/timezone/Asia/Seoul";
  let options = url.parse(time_addr);
  options.method = 'GET';
  try
  {
    //var requ = require("http").get(time_addr, function(res) { console.log("Request server time");
    let requ = require("http").request(options, function(res) { console.log("Request server time");
      if (ESP32.getState().freeHeap > 15000 && (Date().getFullYear()<2021))
      {
          if ("Date" in res.headers) {
            console.log("Got Date ",res.headers.Date); 
            let date = new Date(res.headers.Date);
            let d = date.getTime();
            d += 9000*3600; // GMT to Seoul time
            if (d>0) setTime(d/1000);
            else console.log("Unable to parse date!");
          }
      }
      });
    requ.end();
  }
  catch (e) { fail = 1; console.log('Server is not available for time reading'); }
  if (fail == 1)
  {
    try
    {
              let t = new Date();
              ajax.device=f.ap_ssid;
              ajax.date=t.getFullYear().toString()+"-"+(t.getMonth()+1).toString()+"-"+t.getDate().toString();
              ajax.time=t.getHours().toString()+":"+t.getMinutes().toString()+":"+t.getSeconds().toString();
              ajax.move=T.value1;
              ajax.breath=T.value2;
              console.log('ajax: ', ajax.device, ajax.date, ajax.time);
            try{
              ToJSON(f.web_adr,ajax,function(d) {
                      console.log("Got time");
                  });
            }
            catch(e) { troubles++; }
    }
    catch (e) { console.log('Time problems'); }
  }
}

function onPageRequest(req, res) {
    console.log('8');
  process.memory();
  let a = url.parse(req.url, true);
  if (req.method=="POST"){ FromPOST(req,res);}
  if (a.pathname=="/") {
    if (req.method=="GET"){
		res.writeHead(200, {'Content-Type': 'text/html'});
		WebSetup(res);
    }
  } 
  else if (a.pathname=="/connect") {
    wifi.disconnect(function() {
		wifi.connect( f.st_ssid, {password: f.st_password}, function(err) {
				res.writeHead(200, {'Content-Type': 'text/html'});
				if (err) {
					console.log("Not connected to Wifi.  Error: "+err.toString());
					WebMsg(res,'ERROR','Not connected to Wifi point ('+err.toString()+')');
				}
				else{
					console.log('Connected to Wifi.  IP address is:', wifi.getIP().ip);
					WebMsg(res,'Wi-Fi connect','Device connected to AP point <b>'+wifi.getDetails().ssid+'</b>,<br>IP address is: '+wifi.getIP().ip);
					wifi.save(); // Next reboot will auto-connect
				}
			});
	});
  }
  else if (a.pathname=="/reboot") {
	res.writeHead(200, {'Content-Type': 'text/html'});
    WebMsg(res,'REBOOT','Sensor will restart in few seconds... ');//(res,'REBOOT','System [SDK: '+ESP32.getState().sdkVersion+'] will restart in few seconds... ');
    setTimeout(function(){ ESP32.reboot(); },2000);
  }
  else if (a.pathname=="/certificate") { console.log(a); } 
  else if (a.pathname=="/post.cgi") { console.log('/post.cgi',req,a); } 
  else {
    res.writeHead(404, {'Content-Type': 'text/plain'});
    res.end("404: Page "+a.pathname+" not found");
  }
  if (ScanAP.length < 1) Scan();
}

// AP모드시 접속했을때 나타나는 웹페이지
function WebSetup(res) {
  console.log('9');
  let S1='',S2='',S3='';
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
  res.write('<span><a>Setting Wi-Fi connect</a></span><hr><form id="CN" action="/connect" method="get"></form><form id="REBOOT" action="/reboot" method="get"></form>');
  res.write('<form enctype="multipart/form-data" id="FILE" method="post" action="/certificate"></form><form id="SAVE" action="/" method="post"></form><p><label>Name  AP</label><input id="ap" list="ssid" form="SAVE" name="ap" value="'+f.st_ssid+'" /><datalist id="ssid">');
  for (let prop in ScanAP){ 
	S2 = ScanAP[prop].toString(); 
	if ( S2 == f.st_ssid) {
		S1='--> '+S2;
	} 
	else{ S1 = ''; }
    if(S2 != ''){ res.write('<option value="'+S2+'">'+S1+'</option>'); }
  }
  //
  S1 = ''; S2 = '';
  res.write('</datalist></p><p><label>Password</label><input type="password" id="pw"  form="SAVE" name="ps" size="10" value="'+f.st_password+'"/></p><p><input type="submit"  form="SAVE" value="Save"/><input type="submit"  form="CN" value="Connect"/></p>');
  res.write('<br><label>MAC: '+require('Wifi').getIP().mac+'</label><br><label>IP:&ensp;&emsp;'+require('Wifi').getIP().ip +'</label><br><label>Status: '+require('Wifi').getStatus().station+' </label>');
  //
  if (f.protocol=='mqtt') {S1='checked';}else{S2='checked';}
  res.write('<hr><label>Main server protocol </label><input type="radio" class="dx1" id="sx" form="SAVE" name="sl" '+S1+' value="mqtt"><label>MQTT  </label><input type="radio" class="dx2" id="sy" name="sl" form="SAVE" '+S2+' value="http"><label>HTTP</label>');
  S1 = ''; S2 = '';
  res.write('<section class="dex"><p><label>Server MQTT</label><input type="text" id="ms"  form="SAVE" name="ms" value="'+f.mqtt_server+'"/></p><p><label>Topic MQTT&ensp;</label><input type="text" id="tk"  form="SAVE" name="tk" value="'+f.mqtt_topic+'"/></p>');
  res.write('<p><label>Port MQTT&emsp;</label><input type="text" id="mp"  form="SAVE" name="mp" value="'+f.mqtt_options.port.toString()+'"/></p>');
  res.write('<p><label>User Name &ensp;</label><input type="text" id="mu"  form="SAVE" name="mu" value="'+f.mqtt_options.username.toString()+'"/></p><p><label>Password &emsp;</label><input type="text" id="qp"  form="SAVE" name="qp" value="'+f.mqtt_options.password.toString()+'"/></p>');
  //
  S1 = ''; S2 = ''; S3 = '';
  if (f.mqtt_timeout==10) { S1 = 'selected'; }
  else if (f.mqtt_timeout == 3){ S3 = 'selected'; }
  else { S2 = 'selected'; }
  
  res.write('<p><label>Time Interval&ensp;</label><select id="si" form="SAVE" name="ti" value="'+f.mqtt_timeout.toString()+'"><option '+S3+' value="3">3 sec</option><option '+S1+' value="10">10 sec</option><option   '+S2+' value="0">Never</option></select></p></section>');
	
  res.write('<section class="xed"><p><label>Link&ensp;</label><input type="text" id="sr"  form="SAVE" name="sr" value="'+f.web_adr+'" size="25"/></p><p><label>API#</label><input type="text" id="ak"  form="SAVE" name="ak" value="'+f.web_apikey.toString()+'"/></p></section><input type="submit"  form="SAVE" value="Save"/>');
  //
  //
//   res.write('<p><label>TLS Certificate</label><input type="file" form="FILE" name="files" /></p><p><input type="submit" form="FILE" value="Send" /></p></section><input type="submit"  form="SAVE" value="Save"/>');
  //
  //
  S1 = ''; S2 = ''; S3 = '';
  let S4 = '', S5 = '', S6 = '',S7 = '', S8 = ''; 
    switch (f.distance) {
    case 15:
      S1 = 'selected';    break;
    case 20:
      S2 = 'selected';    break;
    case 25:
      S3 = 'selected';    break;
    case 30:
      S4 = 'selected';    break;
    case 35:
      S5 = 'selected';    break;
    case 40:
      S6 = 'selected';    break;
    case 45:
      S7 = 'selected';    break;
    case 50:
      S8 = 'selected';    break;
    default: break;
    }
  res.write('<hr><p><label>Distance Set</label><select id="ds" form="SAVE" name="ds" value="'+f.distance.toString()+'m'+'"><option '+S1+' value="15">1.5m</option><option  '+S2+' value="20">2.0m</option><option  '+S3+' value="25">2.5m</option><option  '+S4+' value="30">3.0m</option><option  '+S5+' value="35">3.5m</option><option  '+S6+' value="40">4.0m</option><option  '+S7+' value="45">4.5m</option><option  '+S8+' value="50">5.0m</option></select><input type="submit"  form="SAVE" value="Save"/></p>');
//
  
  res.write('<hr><p><label>Data update interval</label><input type="text" id="du"  form="SAVE" name="du" value="'+f.update_interval.toString()+'"/><input type="submit"  form="SAVE" value="Save"/></p>');
  
  
  ///S1 = ''; S2 = ''; S3 = '';
  //if(f.ap_timeout==15) {S1='selected';}else if(f.ap_timeout==0){S2='selected';}else{S3='selected';}
  //res.write('<hr><p><label>Shutdown &emsp;&ensp;</label><select id="sd" form="SAVE" name="to" value="'+f.ap_timeout.toString()+'"><option '+S3+' value="5">5 min</option><option '+S1+' value="15">15 min</option><option   '+S2+' value="0">Never</option></select>');
  //S1='';S2='';S3='';
  //
  //if(f.rs485_speed==115200) {S1='selected';}else if(f.rs485_speed==19200){S2='selected';}else{S3='selected';}
  //res.write('<hr><p><label>RS485 Set/Echo</label><select id="sd" form="SAVE" name="sd" value="'+f.rs485_speed.toString()+'"><option '+S3+' value="9600">9600</option><option  '+S2+' value="19200">19200</option><option '+S1+' value="115200">115200</option></select>');  

  //S1='';S2='';S3='';
  //if(f.rs485_echo==1) {S1='selected';}else if(f.rs485_echo==0){S2='selected';}else{S3='selected';}
  //res.write('<select id="eh" form="SAVE" name="eh" value="-1"><option '+S3+' value="-1">No</option><option '+S2+' value="0">@</option><option '+S1+' value="1">All</option></select><hr>');
  res.write('<hr><input type="submit"  form="REBOOT" value="Reboot sensor"/>');
  res.end('</div></body></html>');
}

function WebMsg(res,name,msg) {
    console.log('10');
  res.write('<!DOCTYPE html><html><head><meta http-equiv="refresh" content ="10; url=http://'+wifi.getAPIP().ip+'/">');  res.write('<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" /><style type="text/css">');
  res.write('body { font-size: 100%; font-family: "Open Sans", sans-serif;  color: #444;  margin:0;  background-color: #F3F3F3;}');
  res.write('label { margin-left:10px; margin-right:15px; color: black;}');
  res.write('span { padding: 7px 10px; text-align: center; color: #336699; font-weight: bold;}');
  res.write('input, select, button{  background-color: white;  border-radius: 5px;  padding: 5px 10px;}');
  res.write('div{background-color: #EFEFEF;border: 2px solid #336699;width: 90%;height: 40%;padding: 10px;margin: 3% auto;border-radius: 9px;');
  res.write('  -moz-border-radius: 9px; -o-border-radius: 9px; -webkit-border-radius: 9px;-moz-box-shadow:0 0 50px #555;-webkit-box-shadow:0 0 50px #555;box-shadow:0 0 50px #555;}');
  res.end('</style></head><body><div><span><a>'+name+'</a></span><hr><form id="B" action="/" method="get"><p><label>'+msg+'</label></p><p><input type="submit" value="Ok" /></p></form></div></body></html>');
}

/////////////////// loop ////////////////////////////////
req_distance = request_dist_preamb + request_dist_val;
ToRS485(req_distance);
var prev_heap = ESP32.getState().freeHeap;

setTimeout( function() { ToRS485(request_interval_val); }, 1000); 
setTimeout( function() { ToRS485(request_update_type); }, 2000); 


E.enableWatchdog( (50), false); // not implemented for ESP32 yet

// 결국 이부분이 계속 돌아가는 거예요. 돌아가다가 뭔가 오류가 나서 다시 공유기에 접속하고
// 다시 시작할때 mqtt의 client_id가 지정된 값이 아닌 디폴트값으로 바뀌는 부분을 수정해야하고.
// 아마 f 값을 다시 셋팅해야할거 같은데. 모르겠네요.
setInterval( function() {
  if (busy === 0){
                E.kickWatchdog();
                no_reset_cnt += 1;
                state = ESP32.getState();
                errors = E.getErrorFlags();
				m = state.freeHeap;
                l = state.minHeap;
                if (m<prev_heap)
                {
                  console.log("Heap decreasing", m);
                  if ((m<20000) && ((block_communications === 0))) {
                     get_server_time(); // This function helps to release some memory occupied by wifi communications
                  }
                }
                prev_heap = m;
                if (Date().getFullYear()<2021)
                {
                  get_server_time();
                }
                if (f.protocol === 'mqtt')
                {
                  if (mqtt.connected) c = 1;
                }
                else { c= 0; }
				s = require('Wifi').getStatus().station;
                ss = "";
                ap = require('Wifi').getAPDetails().ssid;
				if ( s == 'connected' ) ss = require('Wifi').getStatus().ssid;
				if ( require('Wifi').getStatus().mode=='STA' ) ap = "";
				console.log( m+"/"+l+"-"+troubles+':'+errors.length+" ["+c+'] ' + ss + ':' + ap);
                if (no_reset_cnt > 20 ) {
                  process.memory();
                  E.defrag();
                  no_reset_cnt = 0; 
                  console.log('No reboot since', turn_on_time);
                }
				if ( s != 'connected' ) {
					troubles++;
					D33.write(0);
				}
				else { D33.write(1); }
				if ( c == true )
                {
					D32.write(1);
					if(m<6400)
                    {
						mqtt.disconnect();
                        console.log("Low memory for MQTT, will reconnected...");
                        troubles++;
					}
					else
                    {
						if (f.mqtt_timeout>0) 
                        {
						  TT.n++;
						  TT.h = m;
                          var tmp1 = TT.m;
                          var tmp2 = TT.b;
                          
                          if(!isNaN(parseInt(tmp1))){
                            if(tmp2 => 0 && m_flag == 0) {
                              ToMQTT('ACTIVE');
                              m_flag = 1;
                              b_cnt = 0;
                              b_flag = 0;
                              pre_b = 1;
			  ab_cnt = 0;
                            }
                          }
                          else if(isNaN(parseInt(tmp1))){
                            if(isNaN(parseInt(tmp2))){
                              
                              
                              if(m_flag == 0 && pre_b == 0){
                                if(b_flag == 1 && b_cnt > 2){
                                    ToMQTT('Respiratory ARREST!!!');
                                    b_cnt++;
                                    m_flag = 0;
		                            ab_cnt = 0;
                                }
                                
                                else if(b_flag == 1 && b_cnt < 3){
                                  if(pre_b == 0){
                                    ToMQTT('Searching Vital Signal...');
                                    b_cnt++;
                                    m_flag = 0;
			                        ab_cnt++;
                                  }
                                }
                              }
                              if(m_flag == 0 && pre_b == 1){
                                if(ab_cnt == 20 || ab_cnt > 20){
                                  ToMQTT(' ABSENCE1 ');
                                }
                                else if(ab_cnt < 20){
                                  ToMQTT('Searching Vital Signal...1');
                                }
                                ab_cnt++;
                              }
                              if(m_flag == 1 || pre_b == 1){
                                if(ab_cnt == 20 && ab_cnt > 20){
                                  ToMQTT(' ABSENCE2 ');
                                }
                                else if(ab_cnt < 20){
                                  ToMQTT('Searching Vital Signal...2');
                                }
                                ab_cnt++;
                                m_flag = 0;
                              }
                            }
                            else if(tmp2 > 0){
                              ToMQTT(tmp2);
                              b_flag = 1;
                              b_cnt = 0;
                              ab_cnt = 0;
                              if(m_flag == 1){
                                pre_b = 1;
                              }
                              else if(m_flag == 0){
                                pre_b = 0;
                              }
                            }
                          }
                          
							TT.m = [];
							TT.b = [];
						}
					}
				}
				else
                {
                  if (f.protocol === 'mqtt')
                  {
					D32.write(0);
					troubles++;
                  }
				}
                if (m<1000) troubles += 2;
                //if (block_communications === 1) errors = [];
				if ((troubles>10) || (errors.length > 0))
                {
                    console.log('Detect trouble, need reboot!'); 
                    ESP32.reboot();
                }
                if (distance_setup === 0) SetDistance();
				if (send_type_is_known === 0) ToRS485(request_interval_val);
				else if (send_type_is_known === 1) ToRS485(request_update_type);
  }
}, 3000); //3초 인터벌 이 부분을 웹페이지에서 변수로 받아 넣어야 합니다.

const server=require("http").createServer(onPageRequest);
server.listen(f.ap_port);
////////////////////////////////////////////////////////////////////////////
console.log("End of sketch...");
