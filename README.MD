由于博主是个忠实的英雄联盟粉丝，所以经常观看一些明星大神的直播。而一谈到直播，肯定会看到满屏幕飘来飘去的弹幕。那么问题来了，这些视频弹幕网站如何做到实时同步的？PHP如何开发一个类似的网站？

首先要搞定的是前端页面，最起码得有个框，让弹幕飞起来吧。一想到前台，博主头就大（毕竟我不喜欢去扣前端代码，而且做出来的东西还巨丑）。那咱们就百度一下吧，看看有什么好用的弹幕插件，现在开源的东西那么多。

经过搜索，找到了一个jQuery.danmu.js的开源项目。看了一下star的人还挺多。https://github.com/chiruom/jquery.danmu.js

于是乎，管他三七二十一，先down下来再说。

```
git clone https://github.com/chiruom/jquery.danmu.js.git
```

大致一看目录结构如下：

![图示](http://img.blog.csdn.net/20161208130717798)

进入demo目录，先运行一下例子看看结果呗。

果然，点开以后出现了一个高大上的页面，略看一下功能还挺多。但是问题来了，为啥我点击开始，一点反应也木有呢。

寻找原因ing。

原来是源文件中的jQuery插件的问题。在src目录下，并没有该文件

```
  <script src="../src/jquery-2.1.4.min.js"></script>
```

算了还是调用百度的在线jQuery插件吧

```
  <script src="http://libs.baidu.com/jquery/2.1.4/jquery.min.js"></script>
```

再一刷新，不出预料，成功运行。

![图示](http://img.blog.csdn.net/20161208132710326)

很有意思，有木有，很激动有木有。然而重点才刚刚开始。

后端，那就先来说说弹幕的原理吧。弹幕，就相当于一个公共聊天室，都是一个客户端发送消息给服务端，服务端再将收到的消息广播给其他的客户端。

用传统的ajax轮询吗？不行，这样效率太低，想想各大火爆的直播平台都是同一时间几万人在线，几千人同时发弹幕，如果靠ajax轮询一个PHP接口的话服务器会吃不消的。且弹幕消息存储方案略显复杂，有人问为什么要存储呢？因为ajax使用的HTTP协议是无状态协议，A客户端和B客户端之间对于服务器来说没有任何标志，如果服务器要确保A客户端和B客户端分别在两次请求的时候服务器只返回这两个客户端没有获取过的弹幕消息，那么服务器端就必须使用一个缓存来标识某某客户端看过哪条弹幕消息。综上所述ajax可以实现小规模的弹幕通信方案，但是很麻烦。

好在最新的HTML5中加入了WebSocket协议，我们可以通WebSocket这种基于HTTP协议之上的即时通信协议来替代ajax这种传统的我问你答的老旧通信模式。而我们是PHPer，对于我们这种只懂PHP的人该如何编写WebSocket服务端呢？好在我们又得知PHP有一个Swoole扩展，我们在PHP语言中使用它可以很方便的构建一个WebSocket服务端。

关于Swoole，下面这段是其官网上的话：

> PHP的异步、并行、高性能网络通信引擎，使用纯C语言编写，提供了PHP语言的异步多线程服务器，异步TCP/UDP网络客户端，异步MySQL，异步Redis，数据库连接池，AsyncTask，消息队列，毫秒定时器，异步文件读写，异步DNS查询。 Swoole内置了Http/WebSocket服务器端/客户端、Http2.0服务器端。
>Swoole可以广泛应用于互联网、移动通信、企业软件、云计算、网络游戏、物联网（IOT）、车联网、智能家居等领域。 使用PHP+Swoole作为网络通信框架，可以使企业IT研发团队的效率大大提升，更加专注于开发创新产品。 

跟详细的东西请自行参考官网文档。这里就不在废话了。

http://wiki.swoole.com/wiki/page/479.html

还有一个问题需要解决，那就是，这个jquery.danmu.js是基于弹幕运行时间的一个插件。那又要如何做到实时呢。开始博主想的是在服务器端规定一个时间（即其连接时间），当有客户端连接时，返回服务器的当前时间戳，然后以此为依据开始计时。但是遇到的问题如下：

- 该弹幕插件是按十分之秒计时制度。
- 各浏览器上js的定时器的运行时间略有差异。
- 时间不能完全同步。

好吧，博主走弯路子了（没做过这方面的东西，缺乏经验）。这个时候，就需要转变一种思路了。

websocket是实时通信的，哎，那所有客户端的时间，不一致就不一致吧，弹幕发的时间根据各个客户端的为准呗，都以当前各个客户端的时间来发，websocket只传递不包含时间的数据（好吧有点绕，我自己都感觉说饶了），咱们直接来上代码吧。

index.html
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport"    content="width=device-width, initial-scale=1.0">
  <title>弹幕made by diligentyang</title>
  <style>
    body {
      font-family: "Microsoft YaHei" ! important;
      font-color:#222;
    }
    pre {
      line-height: 2em;
      font-family: "Microsoft YaHei" ! important;
    }
    h4 {
      line-height: 2em;
    }
    #danmuarea {
      position: relative;
      background: #222;
      width:800px;
      height: 445px;
      margin-left: auto;
      margin-right: auto;
    }
    .center {
      text-align: center;
    }
    .ctr {
      font-size: 1em;
      line-height: 2em;
    }
  </style>
  <script src="http://libs.baidu.com/jquery/2.1.4/jquery.min.js"></script>
  <script src="../dist/jquery.danmu.min.js"></script>
</head>

<body class="center">
Demo<br><br>
<!--黑背景和弹幕区-->
<div id="danmuarea">
  <div id="danmu" >
  </div>
</div>
<!--控制区-->
<div class="ctr" >
  <button type="button"  onclick="pauser()">弹幕暂停</button>  &nbsp;&nbsp;&nbsp;&nbsp;
  <button type="button"  onclick="resumer() ">弹幕继续</button>&nbsp;&nbsp;&nbsp;&nbsp;
  显示弹幕:<input type='checkbox' checked='checked' id='ishide' value='is' onchange='changehide()'> &nbsp;&nbsp;&nbsp;&nbsp;
  弹幕透明度:
  <input type="range" name="op" id="op" onchange="op()" value="100"> <br>
  当前弹幕运行时间(秒)：<span id="time"></span>&nbsp;&nbsp;
  <!--设置当前弹幕时间(秒)： <input type="text" id="set_time" max=20 />
  <button type="button"  onclick="settime()">设置</button>-->
  <br>
  发弹幕:
  <select  name="color" id="color" >
    <option value="white">白色</option>
    <option value="red">红色</option>
    <option value="green">绿色</option>
    <option value="blue">蓝色</option>
    <option value="yellow">黄色</option>
  </select>
  <select name="size" id="text_size" >
    <option value="1">大文字</option>
    <option value="0">小文字</option>
  </select>
  <select name="position" id="position"   >
    <option value="0">滚动</option>
    <option value="1">顶端</option>
    <option value="2">底端</option>
  </select>
  <input type="textarea" id="text" max=300 />
  <button type="button"  onclick="send()">发送</button>
</div>
<script>
	//WebSocket
	var wsServer = 'ws://123.206.61.229:9505';
	var websocket= new WebSocket(wsServer);
	
	websocket.onopen = function (evt) {
		console.log("Connected to WebSocket server.");
		/*websocket.send("gaga");*/
		//连上之后就打开弹幕
		$('#danmu').danmu('danmuResume');
	};
	
	websocket.onclose = function (evt) {
		console.log("Disconnected");
	};
		
	websocket.onmessage = function (evt) {
		console.log('Retrieved data from server: ' + evt.data);
		var time = $('#danmu').data("nowTime")+1;
		var text_obj= evt.data +',"time":'+time+'}';//获取加上当前时间
		console.log(text_obj);
		var new_obj=eval('('+text_obj+')');
		$('#danmu').danmu("addDanmu",new_obj);//添加弹幕
	};
	
	websocket.onerror = function (evt, e) {
			console.log('Error occured: ' + evt.data);
	};



  //初始化
  $("#danmu").danmu({
    left:0,
    top:0,
    height:"100%",
    width:"100%",
    speed:20000,
    opacity:1,
    font_size_small:16,
    font_size_big:24,
    top_botton_danmu_time:6000
  });
    //一个定时器，监视弹幕时间并更新到页面上
  function timedCount(){
    $("#time").text($('#danmu').data("nowTime"));

    t=setTimeout("timedCount()",50)

  }
  timedCount();


  function starter(){
    $('#danmu').danmu('danmuStart');
  }
  function pauser(){
    $('#danmu').danmu('danmuPause');
  }
  function resumer(){
    $('#danmu').danmu('danmuResume');
  }
  function stoper(){
    $('#danmu').danmu('danmuStop');
  }
  function getime(){
    alert($('#danmu').data("nowTime"));
  }
  function getpaused(){
    alert($('#danmu').data("paused"));
  }
  
  //发送弹幕，使用了文档README.md第7节中推荐的方法
  function send(){
    var text = document.getElementById('text').value;
    var color = document.getElementById('color').value;
    var position = document.getElementById('position').value;
    //var time = $('#danmu').data("nowTime")+1;
    var size =document.getElementById('text_size').value;
    //var text_obj='{ "text":"'+text+'","color":"'+color+'","size":"'+size+'","position":"'+position+'","time":'+time+'}';
	//为了处理简单，方便后续加time，和isnew，就先酱紫发一半吧。
	//注：time为弹幕出来的时间，isnew为是否加边框，自己发的弹幕，常理上来说是有边框的。
    var text_obj='{ "text":"'+text+'","color":"'+color+'","size":"'+size+'","position":"'+position+'"';
	//利用websocket发送
	websocket.send(text_obj);
	//清空相应的内容
    document.getElementById('text').value='';
  }
  //调整透明度函数
  function op(){
    var op=document.getElementById('op').value;
    $('#danmu').danmu("setOpacity",op/100);
  }

  //调隐藏 显示
  function changehide() {
    var op = document.getElementById('op').value;
    op = op / 100;
    if (document.getElementById("ishide").checked) {
      $("#danmu").danmu("setOpacity",1)
    } else {
      $("#danmu").danmu("setOpacity",0)

    }
  }

  //设置弹幕时间
  function settime(){
    var t=document.getElementById("set_time").value;
    t=parseInt(t)
    $('#danmu').danmu("setTime",t);
  }
</script>

</body>
</html>
```

上述代码需要注意的是websocket的建立和接收，以及send方法中对弹幕的处理。

ws_server.php

```
<?php 
//创建websocket服务器对象，监听0.0.0.0:9505端口
$ws = new swoole_websocket_server("0.0.0.0", 9505);

//监听WebSocket连接打开事件
$ws->on('open', function ($ws, $request) {
    //var_dump($request->fd, $request->get, $request->server);
	//相当于记录一个日志吧，有连接时间和连接ip
    echo $request->fd.'-----time:'.date("Y-m-d H:i:s",$request->server['request_time']).'--IP--'.$request->server['remote_addr'].'-----';
});

//监听WebSocket消息事件
$ws->on('message', function ($ws, $frame) {
	//记录收到的消息，可以写到日志文件中
    echo "Message: {$frame->data}\n";

	//遍历所有连接，循环广播
	foreach($ws->connections as $fd){
		//如果是某个客户端，自己发的则加上isnew属性，否则不加
		if($frame->fd == $fd){
			$ws->push($frame->fd, $frame->data.',"isnew":""');
		}else{
			$ws->push($fd, "{$frame->data}");
		}
    }
});

//监听WebSocket连接关闭事件
$ws->on('close', function ($ws, $fd) {
    echo "client-{$fd} is closed\n";
});

$ws->start();
```

运行方法：

输入`php ws_server.php` 先启动服务器端的websocket。如果要后台运行，且不随用户终端关闭而断开，需要创建一个log.txt用于存取上述输出的东西，然后输入`nohup php ws_server.php > log.txt &` 即可。

然后，

![这里写图片描述](http://img.blog.csdn.net/20161208192322955)

注，如果要用此项目，需要自行修改自己的服务器ip地址。只需要修改`var wsServer = 'ws://123.206.61.229:9505';` 处即可，后台代码不需要做任何处理。

github地址：https://github.com/diligentyang/danmu

原文博主：http://blog.csdn.net/qq_28602957

如需转载请明示。
