# iOS 13 适配总结

*自从6月份的WWDC大会公布了OS 13的新版本之后，广大开发者朋友又面临着新一轮的系统升级适配工作；随着苹果9月份发布会脚步的临近，对公司的App升级适配势在必行。*

---

经历了下载Xcode13beta版本、升级10.15beta版本系统、升级测试机到iOS13beta系统之后，紧张又激动地运行公司的项目工程，泪崩...一系列的闪退bug和样式适配工作铺面而来(*😆😆又到了大展手脚的时候了*)。

网上看了一些朋友的分享，不足以解决项目运行遇到的问题，立马上苹果开发者网站查找相关资料，终于有所发现。现将项目中遇到的问题和解决方案记录一下：

## iOS 13发现问题回顾

- 禁止用户获取或者设置私有属性：调用`setValue:forKeyPath:`、`valueForKey:`方法引起的App崩溃。例如：`UITextField`修改`_placeholderLabel.textColor`、`UISearchBar`修改`_searchTextField`。
- `UITabBar`部分调整：`UITabBarItem`播放gif显示比例有问题；`UITabBarItem`只显示图片时，图片位置偏移；`Badge`文字显示字体偏大。
- `UITextField`的`leftView`和`rightView`调整：部分视图位置显示异常

## 1. UITextField

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

- ### 子视图`leftView`和`rightView`显示异常
  
  需要将显示的视图包装在一个简单的`UIView`中或者在需要显示的视图子类化中，实现`systemLayoutSizeFittingSize`方法

  示例如下

  ```ObjectiveC
  //  示例
  - (void)buildTextField {
      UITextField *textField;
      //  iOS13之前
      textField.rightView = [self buildButtonForRightView];
      //  iOS13之后
      textField.rightView = [self buildTextFieldRightView];
  }

  //  iOS13之前正常
  - (UIButton *)buildButtonForRightView {
      UIButton *button = [UIButton buttonWithType:UIButtonTypeRoundedRect];
      //  do somethings
      return button;
  }
  ```

  - 将显示的视图包装在一个简单的`UIView`中
  
  ```ObjectiveC
  //  iOS13之后需要封装一层
  - (UIView *)buildTextFieldRightView {
      UIButton *button = [self buildButtonForRightView];
      //  封装一层
      UIView *rightView = [[UIView alloc] initWithFrame:CGRectZero];
      [rightView addSubview:button];

      return rightView;
  }
  ```

  - 在需要显示的视图子类化中，实现`systemLayoutSizeFittingSize`方法

  ```ObjectiveC
  //  CICustomButton.m文件
  #import "CICustomButton.h"

  @implementation CICustomButton

  - (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize {   
      return CGSizeMake(100, 30);
  }

  @end

  ------------------------
  //  Build TextFiled页面

  //  iOS13之后，修改UIButton为CICustomButton
  - (UIButton *)buildButtonForRightView {
      UIButton *button = [CICustomButton buttonWithType:UIButtonTypeRoundedRect];
      //  do somethings
      return button;
  }
  ```

## 2. UISearchBar

- ### 通过`valueForKey`、`setValue: forKeyPath`获取和设置私有属性，`setValue:forKey`没有问题

示例

```ObjectiveC
//  修改searchBar的textField
UISearchBar *searchBar = [[UISearchBar alloc] init];

//  iOS13之前可直接获取，然后修改textField相关属性
UITextField *searchTextField = [searchBar valueForKey:@"_searchField"];

//  iOS13之后，可遍历searchBar的所有子视图，找到指定的UITextField类型的子视图，然后上述UITextField的相关方法修改属性。

```

## 3. UITabBarItem

- ### 播放gif，设置ImageView图片时注意scale比例

- ### 单独放置图片的位置调整

`IOS_VERSION >= 13.0` 不需要调整`imageInsets`会自动居中，设置会造成位置偏移，可测试

`IOS_VERSION < 13.0`时，需要设置 `viewController.tabBarItem.imageInsets = UIEdgeInsetsMake(5, 0, -5, 0);`

## 4. UITableView

`iOS13`设置`contentView.backgroundColor`会影响`cell`的`selected`或者`highlighted`时的效果；

如果设置了`cell.selectedBackgroundView`，修改`contentView.backgroundColor`为不透明颜色就会受影响；
`contentView`会遮盖`cell.selectedBackgroundView`。

## 5. Tabbar

- ### `Badge`文字显示不正常
  
`badge`默认字体大小，`iOS13`从13号字体变为17号字体

- ### 字体颜色异常

`iOS13`不设置`self.tabBar.barTintColor = [UIColor clearColor];`字体颜色会显示蓝色

## 6. 第三方SDK相关

- ### 友盟社会化分享SDK：使用“新浪微博完整版”闪退

  替换新浪微博最新的SDK版本
  [Github地址](https://github.com/sinaweibosdk/weibo_ios_sdk/releases)，友盟集成SDK暂时未更新

- ### 高德地图相关等其他第三方SDK更新

  查看相关Github或者官方SDK下载地址，更新最新的SDK即可

### 参考内容

1. [iOS & iPadOS 13 Beta 6 Release Notes](https://developer.apple.com/documentation/ios_ipados_release_notes/ios_ipados_13_beta_6_release_notes?preferredLanguage=occ)
2. [友盟+推出全新SDK，适配iOS 13](https://info.umeng.com/detail?id=177&&cateId=1)
  