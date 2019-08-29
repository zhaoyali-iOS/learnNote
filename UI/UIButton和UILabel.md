## UIButton
### UIButton的宽高自动撑开
正常：UIButton的UIButtonTypeCustomer类型，在设置好title或image之后，button在展示的时候会自动撑开系统内部设置frame的宽和高；<br/>
异常：BUtton作为UIBarButtonItem的customerView时就不会自动撑开，需要在barButtonItem展示前手动调用sizeToFit方法才可以正常展示<br/>
异常原因：

### UIButton的TitleEdgeInsets


## UILabel

### Label填充内容自动撑开
在iOS12和iOS12以后在任何情况下都能撑开，但在iOS12以前版本会有问题<br/>
情景：并排放置两个lable，两个label的numberOfLine都是0，lable1约束：lable1.left = super.left + 15,label1.top = super.top + 10
label2的约束：label2.top = label1.top,label2.left = label1.right + 5, label2.right <= supre.right - 15,labele2.bottom = supre.bottom<br/>
<br>
正常：在iOS12及以上都可以自动撑开换行展示<br>
异常：在iOS12以前，放在cell中展示异常，在viewController的view上展示正常<br>
异常现象：<br>
* 两个label的top虽然对其，但baseLine没有对其；text比较多时不能换行，只展示一行的内容
* 在xcode的Debug View Hierarchy中显示label1的width is ambiguous ,label2提示Horizontal position is ambiguous
* 当给table设置预估高度的值时真是就正常

<br>
<br>
基础知识：iOS中能自动撑开的控件都重写了UIView的属性方法intrinsicContentSize

```c
Custom views typically have content that they display of which the layout system is unaware.
...
This intrinsic size must be independent of the content frame, because there’s no way to dynamically communicate a changed width to the layout system based on a changed height, for example.
```
从文档的解释可以知道，这个属性的设置在layout之前，且必须依赖其frame。
比如UIImageView的size就是根据图片本身大小就是intrinsicContentSize；给label根据text、font、numberOfLine=1就可以确定intrinsicContentSize，当numberOfLine不为1时还要根据preferredMaxLayoutWidth(如果没有设置就根据label的superView的frame.width)计算intrinsicContentSize<br>
这解释了为什么在view中可以正常展示在cell中就异常展示<br>
异常原因：<br>
* 从xcode的Debug View Hierarchy中可以知道label的intrinsicContentSize值不能正常计算出来。
* cell在初始化的时候frame应该时zero，所以label的intrinsicContentSize就不能确定，进而label2的X不能确定，本质上label的width也不能测算
* intrinsicContentSize和约束不能实时同步，所以就会导致label的展示异常
* 尝试在cell的initWithStyle方法和prepareForReuse强制给cell设置了fame，结果还是会展示异常，估计内部做了其他处理
* 设置table的estimatedRowHeight时，table先给将要展示cell设置了frame，当更新cell内容时label就能测算出两个label的intrinsicContentSize，展示也就正常了；
        
       
