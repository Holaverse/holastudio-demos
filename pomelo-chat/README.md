用hola studio实现微信风格的聊天软件(Pomelo版)。
---------------------------------------------

为了简化起见，服务器端完全用[Pomelo](https://github.com/NetEase/pomelo/wiki/Chat%E6%BA%90%E7%A0%81%E4%B8%8B%E8%BD%BD%E4%B8%8E%E5%AE%89%E8%A3%85)的DEMO了。

1.先实现登录界面。这个很简单，放三个编辑器，分别用来输入服务器、聊天室和用户名，再放一个按钮用来登录就行了。

![login](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/pomelo-chat/pomelo_chat_login.png)

登录的实现代码：
```
var hostname = this.win.getValueOf("hostname");
var roomname = this.win.getValueOf("roomname");
var username = this.win.getValueOf("username");

if(!username || !roomname || !hostname) {
    return;
}

var initData = {
    roomname:roomname,
    username:username
};

function onLogin(data) {
    initData.data = data;
    this.openWindow("chat", null, true, initData);
}

ChatClient.login(hostname, username, roomname, onLogin.bind(this));

```

2.聊天界面。聊天界面需要自定义控件[请参考](https://github.com/Holaverse/holastudio-components/tree/master/wechat)。在[socket.io的例子](https://github.com/Holaverse/holastudio-demos/tree/master/websocket-chat)也有改进版本，这里就不多说了。

![chat](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/pomelo-chat/pomelo_chat.png)


打开聊天窗口时，注册相关事件。
```
this.username = initData.username;
this.roomname = initData.roomname;

var str = this.roomname + "(" + this.username + ")";
this.setValueOf("title", str);
ChatClient.addEventListeners(this);
```

编辑器onChanged事件发送消息：
```
if(!value) {
    return;
}

var str = value;
var win = this.win;
var list = win.find("list-view");

this.setText("");
list.addRightItem("images/logos/holaverse-32.jpg", win.username, str);     
ChatClient.send("*", str, function(username, target, msage) {});

```
其它一些公共代码：
```
```

