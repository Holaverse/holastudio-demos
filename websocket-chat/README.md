用hola studio实现微信风格的聊天软件。
---------------------------------------------

为了简化起见，服务器端完全用[socket.io](https://github.com/socketio/socket.io)的DEMO了。

1.先实现登录界面。这个很简单，放两个编辑器，分别用来输入服务器和用户名，放一个按钮用来登录就行了。

![login](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/websocket-chat/login.png)

登录的实现代码：
```
var username = this.win.getValueOf("username");
var servername = this.win.getValueOf("servername");

if(!username || !servername) {
    return;
}

var socket = io(servername);
var initData = {
    socket:socket,
    username:username
}

socket.emit('add user', username);
this.openWindow("chat", null, true, initData);

```

2.聊天界面。聊天界面需要自定义控件[请参考](https://github.com/Holaverse/holastudio-components/tree/master/wechat)。

![chat](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/websocket-chat/chat.png)

自定义列表视图控件，这里稍微调整一下，支持显示系统事件。
```
var templateItem = this.getChild(0);
var json =  templateItem.toJson();
templateItem.remove();

this.addItem = function(icon, name, text, atLeft) {
    var item = this.addChildWithJson(json);
    var image = item.find("image");
    var tips = item.find("tips");
    
    tips.y = 0;
    image.y = 0;
    tips.w = this.w - image.w - 60;
    image.style.setFontSize(10);
    image.setValue(icon).setText(name).setTextColor("Orange");
    tips.setText(text).fitToTextContent();
    item.w = this.w;
    item.h = Math.max(image.h, tips.h);
    
    if(atLeft) {
        image.x = 2;
        tips.x = image.w + 20;
        tips.setPointer(-10, 20).setFillColor("White").setLineColor("#d0d0d0")
    }
    else {
        image.x = item.w - image.w - 2;
        tips.x = image.x - tips.w - 20;
        tips.setPointer(tips.w+10, 20).setLineColor("#83d45a").setFillColor("#a0e75a");
    }
    this.relayoutChildren();
    this.scrollToEnd();
    
    return item;
}

this.addCenterItem = function(text) {
    var item = this.addChildWithJson(json);
    var image = item.find("image").remove();
    
    var tips = item.find("tips");
    
    tips.w = item.w;
    tips.setTextAlign("center");
    tips.setPosition(0, 0).setText(text).setTextColor("Orange");
    tips.setPointer(0, 0).setFillColor("White").setLineColor("White");
    
    this.relayoutChildren();
    this.scrollToEnd();
    
    return item;
}

this.addLeftItem = function(icon, name, text) {
    return this.addItem(icon, name, text, true);
}

this.addRightItem = function(icon, name, text) {
    return this.addItem(icon, name, text, false);
}
```

打开聊天窗口时，注册相关事件。
```
this.socket = initData.socket;
this.username = initData.username;

var socket = this.socket;
var list = this.find("list-view");

socket.on('new message', function (data) {
    list.addLeftItem("images/logos/facebook.png", data.username, data.message);
});

socket.on('user joined', function (data) {
   list.addCenterItem(data.username + ' joined');
});

socket.on('user left', function (data) {
   list.addCenterItem(data.username + ' left');
});
```

编辑器onChanged事件发送消息：
```
var str = value;
this.win.socket.emit('new message', str);

var list = this.win.find("list-view");
list.addRightItem("images/logos/holaverse-32.jpg", this.win.username, str);
```
