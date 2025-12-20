# 1手势事件
## 1.1 点击事件
有些widget创建时可以传入点击事件参数，例如：
ElevatedButton、TextButton、OutlineButton、FloatingActionButton
对于没有点击事件参数的widget，可以使用GestureDetetor这个widget来将其包裹并绑定事件
```dart
Container(
  child: Center(
    child: GestureDetetor(
      onTap: () {
	    print("点击了")
      },
      onDoubleTap: () {
        print("双击了")
      },
      child: Text("请点击此处")
    )
  )
)
```
# 2 状态更新
setState