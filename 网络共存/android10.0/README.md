## android 10 wifi以太网共存
sdk默认以太网 70分，wifi 60分，telephone 50分，想让wifi优先级最高，可交接WIFI与以太网默认分数验证。
下面是把wifi 改成70分，以太网改为60分参考。
### 补丁
```
frameworks/base$ git diff
diff --git a/services/core/java/com/android/server/ConnectivityService.java b/services/core/java/com/android/server/ConnectivityService.java
index e5fddef..bd2af61 100644
--- a/services/core/java/com/android/server/ConnectivityService.java
+++ b/services/core/java/com/android/server/ConnectivityService.java
@@ -238,11 +238,13 @@ public class ConnectivityService extends IConnectivityManager.Stub
     private static final String REQUEST_ARG = "requests";

     private static final boolean DBG = true;
-    private static final boolean DDBG = Log.isLoggable(TAG, Log.DEBUG);
-    private static final boolean VDBG = Log.isLoggable(TAG, Log.VERBOSE);
+    private static final boolean DDBG = true;//Log.isLoggable(TAG, Log.DEBUG);
+    private static final boolean VDBG = true;//Log.isLoggable(TAG, Log.VERBOSE);

     private static final boolean LOGD_BLOCKED_NETWORKINFO = true;

+    private static final boolean ENABLE_NETWORK_COEXIST = true;
+
     /**
      * Default URL to use for {@link #getCaptivePortalServerUrl()}. This should not be changed
      * by OEMs for configuration purposes, as this value is overridden by
@@ -6108,7 +6110,12 @@ public class ConnectivityService extends IConnectivityManager.Stub
                 break;
             }
         }
-        nai.asyncChannel.disconnect();
+        if (ENABLE_NETWORK_COEXIST) {
+          log("Skip teardownUnneededNetwork: " + nai.name ());
+        }else{
+               log("execute teardownUnneededNetwork: " + nai.name ());
+             nai.asyncChannel.disconnect();
+        }
     }

     private void handleLingerComplete(NetworkAgentInfo oldNetwork) {
     
frameworks/opt/net/wifi$ git diff
diff --git a/service/java/com/android/server/wifi/WifiNetworkFactory.java b/service/java/com/android/server/wifi/WifiNetworkFactory.java
index 4b5866b..ab86ed7 100644
--- a/service/java/com/android/server/wifi/WifiNetworkFactory.java
+++ b/service/java/com/android/server/wifi/WifiNetworkFactory.java
@@ -80,7 +80,7 @@ import java.util.Set;
 public class WifiNetworkFactory extends NetworkFactory {
     private static final String TAG = "WifiNetworkFactory";
     @VisibleForTesting
-    private static final int SCORE_FILTER = 60;
+    private static final int SCORE_FILTER = 70;
     @VisibleForTesting
     public static final int CACHED_SCAN_RESULTS_MAX_AGE_IN_MILLIS = 20 * 1000;  // 20 seconds
     @VisibleForTesting


frameworks/opt/net/ethernet$ git diff
diff --git a/java/com/android/server/ethernet/EthernetNetworkFactory.java b/java/com/android/server/ethernet/EthernetNetworkFactory.java
index f70e885..f41b6ed 100644
--- a/java/com/android/server/ethernet/EthernetNetworkFactory.java
+++ b/java/com/android/server/ethernet/EthernetNetworkFactory.java
@@ -70,7 +70,7 @@ public class EthernetNetworkFactory extends NetworkFactory {
     private final static String TAG = EthernetNetworkFactory.class.getSimpleName();
     final static boolean DBG = true;

-    private final static int NETWORK_SCORE = 70;
+    private final static int NETWORK_SCORE = 60;
     private static final String NETWORK_TYPE = "Ethernet";

     private final ConcurrentHashMap<String, NetworkInterfaceState> mTrackingInterfaces =
@@ -389,13 +389,13 @@ public class EthernetNetworkFactory extends NetworkFactory {
                     new TransportInfo(ConnectivityManager.TYPE_NONE, 1));
             // EthernetNetworkFactory.NETWORK_SCORE
             sTransports.put(NetworkCapabilities.TRANSPORT_ETHERNET,
-                    new TransportInfo(ConnectivityManager.TYPE_ETHERNET, 70));
+                    new TransportInfo(ConnectivityManager.TYPE_ETHERNET, 60));
             // BluetoothTetheringNetworkFactory.NETWORK_SCORE
             sTransports.put(NetworkCapabilities.TRANSPORT_BLUETOOTH,
                     new TransportInfo(ConnectivityManager.TYPE_BLUETOOTH, 69));
             // WifiNetworkFactory.SCORE_FILTER / NetworkAgent.WIFI_BASE_SCORE
             sTransports.put(NetworkCapabilities.TRANSPORT_WIFI,
-                    new TransportInfo(ConnectivityManager.TYPE_WIFI, 60));
+                    new TransportInfo(ConnectivityManager.TYPE_WIFI, 70));
             // TelephonyNetworkFactory.TELEPHONY_NETWORK_SCORE
             sTransports.put(NetworkCapabilities.TRANSPORT_CELLULAR,
                     new TransportInfo(ConnectivityManager.TYPE_MOBILE, 50));

```
## 测试
wifi连手机热点，以太网连路由，可以不在同一个网段测试
![image](./1.png)
![image](./2.png)

## 防止score更新
```
setprop log.tag.NativeDaemonConnector D
setprop log.tag.NetworkManagement D
setprop log.tag.NetworkAgentInfo D
setprop log.tag.ConnectivityService D
setprop log.tag.netd D
setprop log.tag.EthernetNetworkFactory D
setprop log.tag.NetworkFactory D
setprop log.tag.NetworkAgent D
setprop log.tag.DnsManager D
```
