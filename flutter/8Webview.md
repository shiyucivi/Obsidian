## 2 Webview通信
flutter监听：
```dart
_webViewController = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..loadRequest(Uri.parse('http://192.168.45.233:3022/app-view')) // 默认加载页面
      ..addJavaScriptChannel("FlutterBridge", onMessageReceived: onMessageReceived);
```
flutter端调用js代码：
```dart
_webViewController.runJavaScript('addShips()');
```
webview提供调用方法：
```javascript
window.addShips = function () {}
```
webview向flutter发送信息：
```javascript
window.FlutterBridge.postMessage("Message")
```