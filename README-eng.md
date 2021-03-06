#Android Ptr Comparison
Only compare star > 1500 repos in github.

##Repo
|Repo|Owner|Star<br/>(2015.12.5)|version|Snap shot|
|:--:|:--:|:------:|:---:|:--:|
|[Android-PullToRefresh][3]<br/>(Author deprecated)|[chrisbanes][4]|6014|latest|![chrisbanes](/demo_gif/chrisbanes.gif)|
|[android-Ultra-Pull-To-Refresh][1]|[liaohuqiu][2]|3413|1.0.11|![liaohuqiu](/demo_gif/liaohuqiu.gif)|
|[android-pulltorefresh][5]<br/>(Author deprecated)|[johannilsson][6]|2414|latest|![johannilsson](/demo_gif/johan.gif)|
|[Phoenix][7]|[Yalantis][8]|1897|1.2.3|![yalantis](/demo_gif/yalantis.gif)|
|[FlyRefresh][9]|[race604][10]|1843|2.0.0|![flyrefresh](/demo_gif/flyrefresh.gif)|
|[SwipeRefreshLayout][11]|Android <br/> Support v4|None|19.1.0|![swipe_refresh](/demo_gif/swipe.gif)|

##Expansion

|Repo|Custom header view|Support content view|
|:--:|:--:|:--:|:---:|:--:|:--:|
|[Android-PullToRefresh][3]| Nope. Only support it's `LoadingLayout` as header view, it would take many work to custom.| Any view, build-in support:`GridView`<br/>`ListView`,`HorizontalScrollView`<br/> `ScrollView` ,`WebView`|
|[android-Ultra-Pull-To-Refresh][1]| Any view. Extend `PtrUIHandler` and call `PtrFrameLayout.addPtrUIHandler()` would make yoru header response to "pull to refresh" event.|Any View|
|[android-pulltorefresh][5]| Nope. | Only `ListView`. |
|[Phoenix][7]| Nope.  Header animation is it's feature. | Any view. |
|[FlyRefresh][9]| Nope. Header animation is it's feature. | Any view. |
|[SwipeRefreshLayout][11]| Nope. It use material style header.| Any view. |

##Easy of use

|Repo|Use by gradle|Pull up to load|Auto refresh|Move resistance <br/>(View translation / Finger move)|
|:--:|:--:|:------:|:---:|:--:|:--:|:--:|
|[Android-PullToRefresh][3]|×|√|×|1/2|
|[android-Ultra-Pull-To-Refresh][1]|√|×|√|√|
|[android-pulltorefresh][5]|×|×|×|1/1.7|
|[Phoenix][7]|√|×|×|1/2|
|[FlyRefresh][9]|√|×|×|×|
|[SwipeRefreshLayout][11]|√|×|×|1/2|

##Performance

Analysed by systrace of following action in 1 second：

![trace_operation](trace_operation.gif)

> remark: Many repo can not put custom header view directly, so the comparison lose precision. 

**Analysis**：

As you see, this library have perfect cost time distribution. Unfortunately, it's author do not maintain it anymore.可惜的是作者已经不再维护，顶部视图的扩展性比较差，并且gradle中也无法使用。在本次demo这类层级比较简单的环境中，几乎都达到了60fps，可以与后面的trace对比。

###2. liaohuqiu's Ptr

滑动实现方式：触摸造成的下拉均是`View.offsetTopAndBottom()`实现的；在松手之后，触发`Scroller.startScroll()`计算回滚，使用`View.post(Runnable)`不停地监视`Scroller`的计算结果，从而实现视图变化(此处依然是`View.offsetTopAndBottom()`完成视图移动)。

trace snapshot:

![trace_liaohuqiu](/traces/liaohuqiu.PNG)

**Analysis**：

This library could be the best in expansion ability. You can make your header implement `PtrUIHandler` and add it to `PtrFrameLayout`。I've noticed that "measure" time period take significant time when "pull to refresh" state change. After working around it's source I found `PtrClassicFrameLayout`'s header take responsibility for this. Let's look at it's layout resource:

![liaohuqiu_header](/liaohuqiu_ptr_header.PNG)

So many `wrap_content`! That would cause `View.requestLayout()`. Don't look down this little operation of a little view! `View.requestLayout()` would walk through this route：`View.requestLayout()`->`ViewParent.requestLayout()`->...->`ViewRootImpl.requestLayout()`->`ViewRootImpl.doTraversal()`=>**MEASURE**(ViewGroup)=>**MEASURE**(ChildView of ViewGroup)!

It would cost tremendous time in measure when the app has complicated view hierarchy. That means your "pull down" action would be not so smooth.

I made all `layout_width` and `layout_height` be a fixed dimension, and "measure" time disappear in trace magically! systrace after modify:

![trace_liaohuqiu_new](/traces/liaohuqiu_new.PNG)

###3. johannilsson's Ptr

滑动实现方式：初始时`setSelection(1)`隐藏顶部视图（使用这个下拉刷新控件注意将滚动栏隐藏，否则会露馅）。在拉下来超过header view的measure高度之前，均是`ListView`自有的滚动；在下拉超过header measure高度之后，对header使用`View.setPadding()`让header继续下移。

trace snapshot:

![trace_johan](/traces/johan.PNG)

**分析**：

通过顶视图调用`View.setPadding()`来实现的滑动，在下拉距离超过header高度后，会造成不断的`requestLayout()`!这就解释了为什么图中UI线程的蓝色块时间(measure时间)很明显。**当你在视图层级比较复杂的app中使用它时，下拉动作所造成的开销会非常明显，卡顿是必然结果。**

###4. Yalantis's Ptr

滑动实现方式：通过`View.topAndBottomOffset()`移动视图，在松手之后启动一个`Animation`执行回滚动画，内容视图的移动也使用`View.offsetTopAndBottom()`实现。为了保证内容视图的padding在移动视图之后与布局文件中的padding一致，它额外调用了`View.setPadding()`实时计算与设置padding。

顶部动效实现方式：`Drawable`的`draw()`中，为`Canvas`中设置“太阳”偏移量及背景缩放。

trace snapshot:

![trace_yalantis](/traces/yalantis.PNG)

**分析**：

此开源库动画效果非常柔和，且顶部视图全部是通过draw去更新，不会造成第三个开源库那样的大开销问题。可惜的是比较难以去自定义顶部视图，不好在线上产品中使用，不过这个开源库是一个好的练手与学习的对象。由于顶部动效实现开销不大，它的性能同样非常好。

它的在松手后回滚动画时调用的`View.setPadding()`可能会造成measure开销比较大，于是我特地测了一下松手回滚的trace，一看确实measure时间非常可观：

![trace_yalantis_scroll_back](/traces/yalantis_back.PNG)

确实它如果要保证展示内容视图的padding与布局文件中一致，是必须这么做的（调用`View.setPadding()`），因为通过`View.offsetTopAndBottom()`向下移动视图会影响底部的padding。但是很有意思，它向下移动的时候没有这么设置，拉下来的时候底部padding就没了。回滚动画的时候才设了padding，就显得没那么必要了。我在demo中也进行了实践，确实是这样的：

![yalantis_padding](/demo_gif/yalantis_padding.gif)

实际上，由于这个库是一个嵌套视图，可以尝试在父视图的`onLayout`中进行处理由于位置变化带来的padding影响，这么处理，只是在`layout`阶段处理，不会造成`measure`的大量开销。

我粗略的做了一点点改动，只在父视图中处理padding而不是在子视图里面做，就可以在上拉、下拉的时候保持padding不变，并且性能有了很大提高。不过好像逻辑就有点问题，还需要再做改动，已经跟作者提出issue。

改动后松手回滚trace，已经没有了measure时间：

![yalantis_back_trace_new](/traces/yalantis_back_new.PNG)

改动后的样子，上下拉动时padding不会变动。不过每次回滚的时候顶部会多往里面滚一点，还需作者针对issue完善：

![yalantis_padding_new](/demo_gif/yalantis_padding_new.gif)

###5. race604's Ptr

滑动实现方式：`View.topAndBottomOffset()`

顶部动效实现方式：

- **飞机滑动** `ObjectAnimator`.
- **山体移动、树木弯曲** 通过移动距离计算山体偏移、树木轮廓，得出`Path`后进行draw.

trace snapshot:

![trace_flyrefresh](/traces/flyrefresh.PNG)

**分析**：每次拖动都会重新计算背景"山体"与"树木"的`Path`，造成了draw时间过长。效果不错，也是一个好的学习对象，相比`Yalantis`的下拉刷新性能上就差一些了，它的draw中的计算量太多。使用起来疑似有bug：拖动到顶部，无法再往上拖动，并且会出现拖动异常。

###6. SwipeRefreshLayout

滑动实现方式：内容固定，仅有顶部动效。

顶部动效实现方式：

- **上下移动** `View.bringToFront()` + `View.offsetTopAndBottom()`.
- **动效** 通过移动偏移量计算弧形曲线的角度、三角形的位置，使用`drawArc`, `drawTriangle`将他们画到`Canvas`上。

trace snapshot:

![trace_swipe](/traces/swipe.PNG)

**分析**：官方的下拉刷新组件，动画十分美观简洁，API构造清晰明了。但是为什么每次的移动都会有一段明显的measure时间呢？我研究了一下代码，发现罪魁祸首是`View.bringToFront()`，它在每一次滑动的时候都会对顶部动效视图调用这个函数。仔细追朔这个函数源码，它会走到下面这段代码中：

`ViewGroup.java`
```
    public void bringChildToFront(View child) {
        final int index = indexOfChild(child);
        if (index >= 0) {
            removeFromArray(index);
            addInArray(child, mChildrenCount);
            child.mParent = this;
            requestLayout();
            invalidate();
        }
    }
```

看，它是会触发`View.requestLayout()`的！这个函数会造成的后果我们在之前已经解释了，它会造成大量的UI线程开销。实际上我认为这个函数是没有调用的必要的，`SwipeRefreshLayout`明明在重写`onLayout()`的时候，header会被layout到child之上，没有必要再`bringToFront()`。

于是我copy了一份代码，将这一行注了(对应代码ptr-source-lib/src/main/java/com/android/support/SwipeRefreshLayout.java)，再次编译，measure时间确实没掉了，对功能毫无影响，性能却有了很大优化：

![trace_swipe](/traces/swipe_new.PNG)

这样一来就不会每一次拉动，都会触发measure。若有同学知道这个`bringToFront()`在其中有其他我未探测到的功效，请issue指点:) 

##Conclusion

|Repo|性能|拓展性|综合建议|
|:--:|:--:|:--:|:--:|
|[Android-PullToRefresh][3]|★★★★★|★★★|由于作者不再维护，无法在gradle中配置，顶部视图难以拓展，不建议放入工程中使用|
|[android-Ultra-Pull-To-Refresh][1]|★★★★★|★★★★★|如之前分析，`PtrClassicFrameLayout`性能有缺陷；建议使用`PtrFrameLayout`，性能较好。这套库自定义能力很强，建议使用。|
|[android-pulltorefresh][5]|★|★|实现方式上有缺陷，拓展性也很差。优点就是代码非常简单，只能作为反面例子。|
|[Phoenix][7]|★★★★|★★|效果非常好，性能不错，可惜比较难拓展顶部视图，为了适配布局padding造成了性能损失，有优化空间。可以作为学习与练手的对象。|
|[FlyRefresh][9]|★★★★|★★|效果很新颖，可惜的是顶部视图计算动效上开销太大，优化空间较少，可以作为学习与练手的对象。|
|[SwipeRefreshLayout][11]|★★★|★★|官方出品，更新有保障，但是如上分析，其实性能上还是有点缺陷的，拓展性比较差，不建议放入工程中使用。|

##附录-知识点参考

1. [为你的应用加速 - 安卓优化指南](https://github.com/bboyfeiyu/android-tech-frontier/blob/master/issue-27/%E4%B8%BA%E4%BD%A0%E7%9A%84%E5%BA%94%E7%94%A8%E5%8A%A0%E9%80%9F%20-%20%E5%AE%89%E5%8D%93%E4%BC%98%E5%8C%96%E6%8C%87%E5%8D%97.md)
2. [使用Systrace分析UI性能](https://github.com/bboyfeiyu/android-tech-frontier/blob/b7e3f1715158fb9f2bbb0f349c4ec3da3db81342/issue-26/%E4%BD%BF%E7%94%A8Systrace%E5%88%86%E6%9E%90UI%E6%80%A7%E8%83%BD.md)

[4]: https://github.com/chrisbanes
[3]: https://github.com/chrisbanes/Android-PullToRefresh
[2]: https://github.com/liaohuqiu
[1]: https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh
[5]: https://github.com/johannilsson/android-pulltorefresh
[6]: https://github.com/johannilsson
[7]: https://github.com/Yalantis/Phoenix
[8]: https://github.com/Yalantis
[9]: https://github.com/race604/FlyRefresh
[10]: https://github.com/race604
[11]: http://developer.android.com/reference/android/support/v4/widget/SwipeRefreshLayout.html