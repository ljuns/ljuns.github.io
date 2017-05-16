---
title: 学习总结--Activity 的生命周期和启动模式
date: 2017-02-15 20:00:57
categories: 学习笔记
tags: 生命周期 启动模式

---
可能是因为处于比较安逸的环境（目前团队移动端项目不多），心态有点浮躁，想要提高自己却又始终没有行动起来，为了改变现状同时也改变自己，前两天定下个目标：每周产出一篇文章，作为自己成长道路上的垫脚石。

这将是一个系列，“学习总结”的系列，主要是对《Android 开发艺术探索》学习的总结，如无特别说明，该系列中的引用都来自《Android 开发艺术探索》。

<!-- more -->

## Activity 的生命周期
### 典型情况下的生命周期
Activity 的生命周期是老生常谈了，不知道大家在面试的时候是否有遇到过 Activity 生命周期的问题，反正我是有遇见过。这个知识点虽然很基础，但也很重要，无论从事什么行业什么工作，基础都是最重要的，难道不是么？先来看张图：

![](http://oaydqd1yy.bkt.clouddn.com/png/activity_lifecycle.png)

这是官网上描述 Activity 生命周期的图，也就是这里说的典型的 Activity 生命周期。关于回调方法这里便不再说，相信大家都可以很轻松的从各种途径获取到。
### 异常情况下的生命周期
Activity 除了受用户操作所导致的正常的生命周期方法调度，还有一些异常情况，比如当资源相关的系统配置发生改变以及系统内存不足时，Activity 就可能被杀死。

1. #### 资源相关的系统配置发生改变时Activity 的生命周期变化。

  比如说当前 Activity 处于竖屏状态，如果旋转屏幕，系统配置就会发生改变，默认情况下 Activity 就会被销毁并且重新创建。那么这种情况下的生命周期又是怎样的呢？来个示例：首先启动一个 Activity，随便输入点东西：</br>
![](http://oaydqd1yy.bkt.clouddn.com/png/tmp03a4f365.png)</br>
此时的生命周期是这样的：</br>
![](http://oaydqd1yy.bkt.clouddn.com/png/tmp548eee0b.png)</br>
然后旋转一下屏幕：</br> 
![](http://oaydqd1yy.bkt.clouddn.com/png/tmp42676dc1.png)</br>
这时的生命周期又是这样的：</br>
![](http://oaydqd1yy.bkt.clouddn.com/png/tmp0d354993.png)</br>
也就是说默认情况下，系统配置发生改变后 Activity 会被销毁并重新创建，同时系统会回调 onSaveInstanceState() 方法来保存当前的状态，用 onRestoreInstanceState() 方法来恢复之前保存的状态。所以会发现尽管 Activity 重新创建了，但是一开始输入的数据并没有消失，需要强调的是：__onSaveInstanceState() 这个方法只会在 Activity 被异常终止的情况下回调__。</br>
其实细心点还能发现 onCreate() 方法和 onRestoreInstanceState() 方法都有一个相同的 Bundle 参数，所以在 onCreate() 方法中也同样可以恢复一些数据。它们的区别在于：onCreate() 方法不管是正常情况还是异常情况都会被回调，所以需要判断参数是否为 null；但是只要系统回调了 onRestoreInstanceState() 方法就说明参数一定不为 null，所以官方的建议是在 onRestoreInstanceState() 方法中做一些恢复数据的工作。

2. #### 系统内存不足导致 Activity 被杀死

  这种情况不好模拟，但是它的数据存储和恢复过程和前面一样。下面了解一下 Activity 的优先级情况，可以分为以下三种：
  1. 前台 Activity：正在和用户交互的 Activity，优先级最高；
  2. 可见但非前台 Activity：比如 Activity 中弹出一个对话框，导致 Activity 可见但是失去焦点无法和用户直接交互；
  3. 后台 Activity：已经被停止的 Activity，优先级最低。

  当系统内存不足时，系统会按照上述优先级排列从最低优先级开始去杀死 Activity 所在的进程，并在后续通过 onSaveInstanceState() 和 onRestoreInstanceState() 来存储和恢复数据。

## Activity 的启动模式
一个 Android 应用程序功能通常会被拆分为多个 Activity，各个 Activity 之间通过 Intent 进行连接，而 Android 系统则通过栈结构来保存整个应用程序的 Activity，这个栈就被叫做任务栈，栈底元素是整个任务栈的发起者。
### standard：标准模式
这是系统默认的启动模式，每次启动一个 Activity 都会创建一个新的实例，不管这个实例是否已经存在。在这种模式下，谁启动了这个 Activity，那么这个 Activity 就运行在启动它的那个 Activity 所在的栈中。

<font color=red size=4>注意：</font> 如果是用 ApplicationContext 去启动 standard 模式的 Activity 会报错：

``` java
E/AndroidRuntime: FATAL EXCEPTION: main
  android.util.AndroidRuntimeException:Calling startActivity() from 
  outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK 
  flag. Is this really what you want?
```

这是因为非 Activity 类型的 Context（ApplicationContext）并没有所谓的任务栈。出现这种情况时给待启动的 Activity 添加 `FLAG_ACTIVITY_NEW_TASK` 这个标记便可以解决问题。

### singleTop：栈顶复用模式
在这种模式下，如果 Activity 已经处于栈顶位置，当启动该 Activity 时不会重新创建实例，同时onCreate()、onStart() 方法也不会被系统调用，但是系统会在 Activity 启动时回调它的 onNewInstance() 方法。这种情况下的生命周期如下图：</br>
![](http://oaydqd1yy.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-18%2012.50.37.png)</br>
如果 Activity 没有处于栈顶位置，启动该 Activity 就会重新创建新的实例。

举个例子：假设当前任务栈中有 A、B、C、D 四个 Activity，其中 A 位于栈底，D 位于栈顶。如果 D 的启动模式是 singTop，此时 D 位于栈顶，当再次启动 D 的时候栈内的情况仍为 ABCD；如果 B 的启动模式是 singTop，此时 B 不是位于栈顶，当再次启动 B 的时候栈内的情况就变为 ABCDB。

### singleTask：栈内复用模式
这里先要了解一下：什么是 Activity 需要的任务栈？默认情况下，一个应用中所有的 Activity 需要的任务栈的名字是该应用的包名。但是也可以为每个 Activity 单独指定 TaskAffinity 这个属性，这个属性值必须不能和包名相同。TaskAffinity 属性主要和 singleTask 启动模式或者和 allowTaskReparenting 属性配对使用，在其他情况下没有意义。

在 singleTask 这种模式下，只要 Activity 在任何一个栈中存在，那么不管重复启动多少次该 Activity 也不会重新创建实例，同时系统也会回调它的 onNewInstance() 方法。具体如下：</br>
![](http://oaydqd1yy.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-18%2015.01.19.png)</br>
下面举几个例子来说明一下：

> 
* 比如目前任务栈 S1 中的情况为 ABC，这个时候 Activity D 以 singleTask 模式请求启动，其所需要的任务栈为 S2，由于 S2 和 D 的实例都不存在，所以系统会先创建任务栈，然后再创建 D 的实例并将其入栈到 S2。
* 另外一种情况，假设 D 所需的任务栈为 S1，其他情况如前面所示，那么由于 S1 已经存在，所以系统会直接创建 D 的实例并将其入栈道 S1。
* 如果 D 所需的任务栈为 S1，并且当前任务栈 S1 的情况为 ADBC，根据栈内复用的原则，此时 D 不会重新创建，系统会把 D 切换到栈顶并调用其 onNewInstance() 方法，同时由于 singleTask 默认具有 clearTop 的效果，会导致栈内所有在 D 上面的 Activity 全部出栈，于是最终 S1 中的情况为 AD 。

### singleInstance：单实例模式
这是一种加强的 singleTask 模式，具有此种模式的 Activity 拥有 singleTask 模式所有的特性，同时具有该模式的 Activity 只能单独地位于一个任务栈中。

举个例子：有 A、B、C 三个 Activity，其中 A 和 C 的启动模式为 standard，B 的启动模式为 singleInstance，启动顺序为 A -> B -> C，此时会发现 AC 会位于同一个任务栈中，而 B 位于另外一个任务栈。当按下 Back 键进行返回时，C 直接返回到了 A，再按下 Back 键就返回到 B。这是因为 A 和 C 所在的任务栈已经空了，再按下 Back 键时就会显示另一个任务栈的 Activity（B）。

## IntentFilter 的匹配规则
启动 Activity 分为显示启动和隐式启动两种，显示启动需要明确指定被启动对象的组件信息，包括包名和类名，而隐式启动则不需要明确指定组件信息。原则上一个 Intent 不应该既是显示启动又是隐式启动，__如果一个 Intent 同时存在显示启动和隐式启动则以显示启动为主__。

隐式启动需要 Intent 能够匹配目标组件的 intent-filter 中所设置的过滤信息，包括 action、category、data。一个 Intent 只有同时匹配 action、category、data 才算完全匹配，才能成功启动目标 Activity。另外一点，一个 Activity 中可以有多个 intent-filter，一个 Intent 只要能完全匹配任何一个 intent-filter 就可以启动对应的 Activity。

### action 的匹配规则
系统预定义了一些 action，同时我们也可以在应用中定义自己的 action。一个过滤规则中可以有多个action，__只有当 Intent 中的 action 和过滤规则中的其中一个 action 完全相同(包括大小写字符串)才算是匹配成功__。

### category 的匹配规则
category 的匹配规则和 action 的匹配规则不同，Intent 中可以没有 category，如果有就必须和过滤规则中的任何一个 category 完全匹配。其实系统在调用 startActivity() 方法或 startActivityForResult() 方法的时候会默认为 Intent 加上 `android.intent.category.DEFAULT` 这个 category。

### data 的匹配规则
首先了解下 data 的结构，data 由两部分组成：mimeType 和 URI。mimeType 指媒体类型，比如 image／jpeg、audio／mpeg4-generic 和 video／* 等，可以表示图片、文本、视频等不同的媒体格式；URI 的结构：</br>
`<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]`</br>

* Scheme：URI 的模式，如果 URI 中没有指定 scheme 意味着 URI 无效；
* Host：URI 的主机名，如果 URI 中没有指定 host 意味着 URI 无效；
* Port：URI 的端口号；
* Path、pathPattern、pathPrefix：表示路径信息。

实际的例子：</br>

``` java
content://com.example.project:200/folder/subfolder/etc
http://www.baidu.com:80/search/info
```

匹配规则：如果过滤规则中定义了 data，那么 Intent 中必须也要定义可完全匹配过滤规则中的任何一个 data。<font color=red size=4>注意：</font> Intent 的 setData() 方法和 setType() 方法会彼此清除对方的值，所以如果要指定完整的 data 必须使用 setDataAndType() 方法。

“Activity 的生命周期和启动模式” 的学习总结就到这了，如果有发现错误或有更好的见解再来补充说明。