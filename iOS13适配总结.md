# iOS 13 适配总结

*自从6月份的WWDC大会公布了OS 13的新版本之后，广大开发者朋友又面临着新一轮的系统升级适配工作；随着苹果9月份发布会脚步的临近，对公司的App升级适配势在必行。*

---

经历了下载Xcode13beta版本、升级10.15beta版本系统、升级测试机到iOS13beta系统之后，紧张又激动地运行公司的项目工程，泪崩...一系列的闪退bug和样式适配工作铺面而来(*😆😆又到了大展手脚的时候了*)。

网上看了一些朋友的分享，不足以解决项目运行遇到的问题，立马上苹果开发者网站查找相关资料，终于有所发现。现将项目中遇到的问题和解决方案记录一下：

## iOS 13发现问题回顾

- 禁止用户获取或者设置私有属性：调用`setValue:forKeyPath:`、`valueForKey:`方法引起的App崩溃。例如：`UITextField`修改`_placeholderLabel.textColor`、`UISearchBar`修改`_searchTextField`。
- `UITabBar`部分调整：`UITabBarItem`播放gif显示比例有问题；`UITabBarItem`只显示图片时，图片位置偏移；`Badge`文字显示字体偏大。
- `UITextField`的`leftView`和`rightView`调整：部分视图位置显示异常

## 1.UITextField

- ### 设置placeholder引起的闪退
  
  在iOS 13之前，设置placeholder有三种方案：
  
  - 基于KVO简单粗暴的修改私有属性(<font color="warning">iOS 13禁止使用</font>)

    ```ObjectiveC
    [textField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor"];
    [textField setValue:[UIFont boldSystemFontOfSize:16] forKeyPath:@"_placeholderLabel.font"]
    ```

  - 通过`attributedPlaceholder`属性修改
  
    ```ObjectiveC
    textField.attributedPlaceholder = [[NSAttributedString alloc] initWithString:@"placeholder" attributes:@{NSForegroundColorAttributeName: [UIColor darkGrayColor], NSFontAttributeName: [UIFont systemFontOfSize:13]}];
    ```

  - 重写`UITextField`的`drawPlaceholderInRect`方法

    ```ObjectiveC
    - (void)drawPlaceholderInRect:(CGRect)rect {
  
      [self.placeholder drawInRect:rect withAttributes:@{NSForegroundColorAttributeName: [UIColor purpleColor], NSFontAttributeName: [UIFont systemFontOfSize:13]}];
    }
    ```
  
  适配iOS 13时，按照实际情况选取后两种方案解决闪退问题

  1. 如果项目中重复使用了同一种`UITextField`的样式，推荐第三种，创建`UITextField`的子类，

- ### `leftView`和`rightView`显示异常
  
  需要将显示的视图包装在一个简单的`UIView`中或者在需要显示的视图子类化中，实现`systemLayoutSizeFittingSize`方法
  
### 参考内容

1. [iOS & iPadOS 13 Beta 6 Release Notes](https://developer.apple.com/documentation/ios_ipados_release_notes/ios_ipados_13_beta_6_release_notes?preferredLanguage=occ)
