widget是flutter提供的基础组件。所有页面中的组件都继承自的widget。
material提供了一些常用的widget和主题风格。
每个我们创建的widget都必须@override一个build方法，返回一个widget类型。
以下是material中提供的常用widget
# 1 MaterialApp
MaterialApp用于作为根widget包裹整个app。提供了一些常用属性：
```dart
MaterialApp(
  title, //String  标题，可以不设置
  theme, //ThemeData  主题
  home, //Widget 主页面，首次进入app时渲染的页面
  routes, //Map<String, WidgetBuilder>  路由
  navigatorKey, //外部导航
  builder, //TransitionBuilder  可以给每个页面设置一个外层的widget
);
```
# 2 Scaffold
用于构建页面的基础组件。常用属性：
```dart
Scaffold(
  appBar, //AppBar  页面上方的bar，可以放置标题和导航按钮
  body, //Widget  页面主体内容
  bottomNavigationBar, //BottomNavigationBar 页面底部导航栏
  floatingActionButton, //FloatingActionButton 悬浮按钮
  backgroundColor, //Color 背景颜色
)
```
# 3 无状态组件与有状态组件
## 3.1 无状态组件 StatelessWidget
无状态组件内部不能修改自己的状态，只能从外部修改。适用于没有用户交互只需要展示的组件。
实现：
```dart
class MyWidget extends StatelessWidget {
  const MyWidget({super.key}); //keyw为widget的唯一标识，可以不写但最好写上
  @override
  Widget build (BuildContext context) {
    return Scaffold(
      body: Container(
	    appBar: AppBar(
		  text: Text("hello world"),
	    )
      )
    );
  }
}
```
生命周期：
build()：当初次渲染或者父组件状态变化导致其变化时触发
## 3.2 状态组件StatefulWidget
可在内部改变其状态，常用于交互式组件如表单。
状态组件需要创建两个类：
一个StatefulWidget类，用于接收和定义参数
一个State类，负责管理状态和交互逻辑，并渲染视图
```dart
class _NumberPlus extends State {
  int _number = 0;
  void _plusOne() {
    setState(() { this._number ++; }) 
  }
  @override
  Widget build(BuildContext context) {
    return Scaffold(
	  body: Center(
	    child: Container(
	      child: Button(
	        text: Text("加一"),
	        tapClick: _plusOne
          ),
	    )
	  )
    );
  }
}
class MainPage extends StatefulWidget {
  @override
  State<_NumberPlus> createState() {
    return _NumberPlus();
  }
}
```
生命周期：
createState()：初始化执行一次，用于创建state对象
initState()：state创建完成后，将state对象插入Widget树
didChangeDependencies()：initState后以及inheritWidget变化时触发
build()：首次渲染或didUpdateWidget()后触发
didUpdateWidget()：state更新后触发
deactivate、dispose：销毁

# 4 基础容器
## 4.1 Conainer

## 4.2 Center

## 4.3 Align

## 4.4 Padding
# 5 布局组件
## 5.1 Row

## 5.2 Column

## 5.3 弹性布局Flex、Expanded、Flexible

## 5.4 流式布局Wrap、Flow

## 5.5 固定布局Stack、Positioned

## 5.6 列表视图ListView、GirdView

## 5.7 滚动视图PageView、SingleChildScrollView、CustomScrollView
# 6 内容组件
## 6.1 Text

## 6.2 Image

## 7 表单组件
## 7.1 TextField

## 8


