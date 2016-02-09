多人在线游戏是很复杂的，直到最近看到agar.io，原来多人在线游戏也可以这么简单（至少实现一个DEMO不难）。趁过年期间有点空隙，花了两个小时写个DEMO，供有兴趣的朋友参考。

为了简化起见，服务器端完全用[Pomelo](https://github.com/NetEase/pomelo/wiki/Chat%E6%BA%90%E7%A0%81%E4%B8%8B%E8%BD%BD%E4%B8%8E%E5%AE%89%E8%A3%85)的DEMO了。服务器我们不做任何修改，用聊天的消息作为同步游戏状态的载体。

1.先实现登录界面。这个很简单，放三个编辑器，分别用来输入服务器、聊天室和用户名，再放一个按钮用来登录就行了。

![login](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/pomelo-mmorpg/login.png)

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
    this.openWindow("win-main", null, true, initData);
}

ChatClient.login(hostname, username, roomname, onLogin.bind(this));
```

2.实现游戏场景。这里只是一个空的场景，每个玩家进入后可以通过虚拟摇杆移动自己。虚拟摇杆的实现请参考[https://github.com/Holaverse/holastudio-components/tree/master/joystick](https://github.com/Holaverse/holastudio-components/tree/master/joystick)。

![scene](https://raw.githubusercontent.com/Holaverse/holastudio-demos/master/pomelo-mmorpg/scene.png)


创建一个js文件main-scene.js，把公共代码放到里面：
```
function MainScene() {
    
}

//玩家离开时，把Ta场景中删除。
MainScene.onUserLeft = function(data) {
    var win = MainScene.win;
    var figure = win.find(data.user);
    if(figure) {
        figure.remove(true, true);
    }
}

//其他玩家加入时，在自己的场景中创建新玩家的角色，同时在新玩家的场景中更新自己的位置。
MainScene.onUserEnter = function(data) {
    var win = MainScene.win;
    var newPlayer = win.dupChild("figure");
    newPlayer.setPosition(0, 0).setText(data.user).setName(data.user);
    
    var figure = MainScene.figure;
    MainScene.update(figure.x, figure.y, figure.getRotation());
}

//其他玩家位置移动时，更新该玩家在自己的场景中的位置。
MainScene.onMessage = function(data) {
    var win = MainScene.win;
    var figureState = JSON.parse(data.msg);
    
    if(figureState.type === "update-state") {
        var figure = win.find(figureState.name);
        if(figure) {
            figure.setPosition(figureState.x, figureState.y).setRotation(figureState.angle);
        }
    }
}

//自己的位置有变化时，通知其它玩家更新。
MainScene.update = function(x, y, angle) {
    var figureState = {type:"update-state", name:MainScene.username, x:x, y:y, angle:angle};
    ChatClient.send("*", JSON.stringify(figureState), function(username, target, msage) {});
    
    return;
}

//通过虚拟摇杆移动自己的位置。
MainScene.onJoyStickKeyPressed = function(args) {
    var dx = args.dx*2;
    var dy = args.dy*2;
    var angle = args.angle*Math.PI/180;
    
    var x = MainScene.figure.x + dx;
    var y = MainScene.figure.y + dy;
    
    MainScene.update(x, y, angle);
}

//初始化时，创建自己和场景中已经有的玩家。
MainScene.init = function(win, initData) {
    MainScene.win = win;
    MainScene.username = initData.username;
    MainScene.roomname = initData.roomname;
    
    MainScene.figure = win.dupChild("figure");
    MainScene.figure.setName(MainScene.username).setText(MainScene.username);
    
    pomelo.on('onChat', MainScene.onMessage.bind(MainScene));
    pomelo.on('onAdd', MainScene.onUserEnter.bind(MainScene));
    pomelo.on('onLeave', MainScene.onUserLeft.bind(MainScene));

    var users = initData.data.users;
    for(var i = 0; i < users.length; i++) {
        var name = users[i];
        if(name === MainScene.username) continue;
        var figure = win.dupChild("figure");
        figure.setName(name).setText(name);
    }
    
    MainScene.update(0, 0, 0);
    
    return;
}

```

在场景的onOpen函数中：
```
MainScene.init(this, initData);
```

在摇杆的onKeyPressed事件中：
```
MainScene.onJoyStickKeyPressed(args);
```

还有几个对pomelo的包装函数，放到client.js中：
```
function ChatClient() {
    
}

ChatClient.queryEntry = function(uid, callback) {
    var route = 'gate.gateHandler.queryEntry';
    pomelo.init({
        host: ChatClient.hostname,
        port: 3014,
        log: true
    }, function() {
        pomelo.request(route, {
            uid: uid
        }, function(data) {
            pomelo.disconnect();
            if(data.code === 500) {
                return;
            }
            callback(data.host, data.port);
        });
    });
}

ChatClient.login = function(hostname, username, rid, callback) {
    ChatClient.hostname = hostname;
    
    ChatClient.queryEntry(username, function(host, port) {
        pomelo.init({
            host: host,
            port: port,
            log: true
        }, function() {
            var route = "connector.entryHandler.enter";
            pomelo.request(route, {
                username: username,
                rid: rid
            }, function(data) {
                if(data.error) {
                    return;
                }
                callback(data);
                ChatClient.rid = rid;
                ChatClient.username = username;                
            });
        });
    });
}

ChatClient.send = function(target, msg, callback) {
    var rid = ChatClient.rid;
    var username = ChatClient.username;
    var route = "chat.chatHandler.send";
    
    pomelo.request(route, {
                rid: rid,
                content: msg,
                from: username,
                target: target
            }, function(data) {
                if(target != '*' && target != username) {
                    callback(username, target, msg);
                }
    });
}

require('boot');
```

3.最后还要在项目中引用pomelo的库，它的位置取决于pomelo服务器的名称了，我的pomelo安装在本机：
http://192.168.168.108:3001/js/lib/build/build.js
