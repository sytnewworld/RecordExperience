<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-08-27 09:20:34
 * @LastEditTime: 2019-08-27 12:11:23
 * @LastEditors: Please set LastEditors
 -->

# iOS 13 适配总结

> *自从6月份的**WWDC**大会展示了`iOS 13`的新版本之后，广大开发者朋友又面临着新一轮的系统升级适配工作；随着苹果9月份发布会脚步的临近，对公司的App升级适配势在必行。*

## iOS 13发现问题回顾

- 禁止用户获取或直接设置私有属性：调用`setValue:forKeyPath:`、`valueForKey:`方法会引起App崩溃。例如：`UITextField`修改`_placeholderLabel.textColor`、`UISearchBar`修改`_searchField`
- `UITextField`的`leftView`和`rightView`调整：部分视图位置显示异常
- `UITabBar`部分调整：`UITabBarItem`播放gif显示比例有问题；`UITabBarItem`只显示图片时，图片位置偏移；`Badge`文字显示偏大
- `UITableView`的`cell`选中样式失效
- 第三方SDK的闪退兼容问题

### 将所有问题归纳总结，得出以下几点解决方案的建议和示例代码，记录一下

## 1. UITextField

- ### 设置`placeholder`引起的闪退
  
  **在`iOS 13`之前，设置`placeholder`有三种方案：**
  
  - 基于`KVO`简单粗暴的修改私有属性(`iOS 13`禁用)

    ```ObjectiveC
    [textField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor"];
    [textField setValue:[UIFont boldSystemFontOfSize:16] forKeyPath:@"_placeholderLabel.font"];
    ```

  - 设置`attributedPlaceholder`属性
  
    ```ObjectiveC
    textField.attributedPlaceholder = [[NSAttributedString alloc] initWithString:@"placeholder" attributes:@{NSForegroundColorAttributeName: [UIColor darkGrayColor], NSFontAttributeName: [UIFont systemFontOfSize:13]}];
    ```

  - 重写`UITextField`子类的`drawPlaceholderInRect:`方法

    ```ObjectiveC
    - (void)drawPlaceholderInRect:(CGRect)rect {
  
      [self.placeholder drawInRect:rect withAttributes:@{NSForegroundColorAttributeName: [UIColor purpleColor], NSFontAttributeName: [UIFont systemFontOfSize:13]}];
    }
    ```
  
  适配`iOS 13`时，可根据实际情况选取后两种方案解决闪退问题。如果项目中重复使用了同一种`UITextField`的样式，推荐使用第三种方案，创建`UITextField`的子类。

  **修改建议： 采用第二种方案，创建`UITextField`的`Category`文件，里面封装好修改`placeholder`的方法，后续修改都可统一直接调用这些方法**

  ```ObjectiveC
  // UITextField+CIPlaceholder.m文件

  #import "UITextField+CIPlaceholder.h"

  @implementation UITextField (CIPlaceholder)

  - (void)setPlaceholderFont:(UIFont *)font {

      [self setPlaceholderColor:nil font:font];
  }

  - (void)setPlaceholderColor:(UIColor *)color {

      [self setPlaceholderColor:color font:nil];
  }

  - (void)setPlaceholderColor:(nullable UIColor *)color font:(nullable UIFont *)font {

      if ([self checkPlaceholderEmpty]) {
          return;
      }

      NSMutableAttributedString *placeholderAttriString = [[NSMutableAttributedString alloc] initWithString:self.placeholder];
      if (color) {
          [placeholderAttriString addAttribute:NSForegroundColorAttributeName value:color range:NSMakeRange(0, self.placeholder.length)];
      }
      if (font) {
          [placeholderAttriString addAttribute:NSFontAttributeName value:font range:NSMakeRange(0, self.placeholder.length)];
      }

      [self setAttributedPlaceholder:placeholderAttriString];
  }

  - (BOOL)checkPlaceholderEmpty {

      return IsStrEmpty(self.placeholder);
  }

  @end
  ```

- ### 子视图`leftView`和`rightView`显示异常
  
  **修改建议：将需要显示的视图包装在一个简单的`UIView`中或者在需要显示的自定义视图子类里，实现`systemLayoutSizeFittingSize:`方法**
  
  ```ObjectiveC
  //  示例代码
  - (void)buildTextField {
      UITextField *textField;
      //  iOS 13之前
      textField.rightView = [self buildButtonForRightView];
      //  iOS 13之后
      textField.rightView = [self buildTextFieldRightView];
  }

  //  iOS 13之前正常
  - (UIButton *)buildButtonForRightView {
      UIButton *button = [UIButton buttonWithType:UIButtonTypeRoundedRect];
      //  do somethings
      return button;
  }
  ```

  - 将显示的视图包装在一个简单的`UIView`中
  
    ```ObjectiveC
    //  iOS 13之后需要封装一层
    - (UIView *)buildTextFieldRightView {
        UIButton *button = [self buildButtonForRightView];
        //  封装一层
        UIView *rightView = [[UIView alloc] initWithFrame:button.bounds];
        [rightView addSubview:button];

        return rightView;
    }
    ```

  - 在需要显示的自定义视图子类里，实现`systemLayoutSizeFittingSize:`方法

    ```ObjectiveC
    //  CICustomButton.m文件
    #import "CICustomButton.h"

    @implementation CICustomButton

    //  iOS 13之后需要实现该方法
    - (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize {
        return CGSizeMake(100, 30);
    }

    @end

    ------------------------
    //  Build TextFiled页面

    //  iOS 13之后，修改UIButton为CICustomButton
    - (UIButton *)buildButtonForRightView {
        UIButton *button = [CICustomButton buttonWithType:UIButtonTypeRoundedRect];
        //  do somethings
        return button;
    }
    ```

## 2. UISearchBar

### 通过`valueForKey`、`setValue: forKeyPath`获取和设置私有属性程序崩溃

```ObjectiveC
//  修改searchBar的textField
UISearchBar *searchBar = [[UISearchBar alloc] init];

//  iOS 13之前可直接获取，然后修改textField相关属性
UITextField *searchTextField = [searchBar valueForKey:@"_searchField"];

```

**修改建议：可遍历`searchBar`的所有子视图，找到指定的`UITextField`类型的子视图，根据上述`UITextField`的相关方法修改属性；也可根据`UITextField`自定义`UISearchBar`的显示**

```ObjectiveC
//  UISearchBar+CIChangePrivateSubviews.m文件

//  修改SearchBar的系统私有子视图

#import "UISearchBar+CIChangePrivateSubviews.h"

@implementation UISearchBar (CIChangePrivateSubviews)

/// 修改SearchBar系统自带的TextField
- (void)changeSearchTextFieldWithCompletionBlock:(void(^)(UITextField *textField))completionBlock {

    if (!completionBlock) {
        return;
    }

    UITextField *textField = [self findTextFieldWithView:self];
    if (textField) {
        completionBlock(textField);
    }
}

/// 递归遍历UISearchBar的子视图，找到UITextField
- (UITextField *)findTextFieldWithView:(UIView *)view {

    for (UIView *subview in view.subviews) {
        if ([subview isKindOfClass:[UITextField class]]) {
            return (UITextField *)subview;
        }else if (!IsArrEmpty(subview.subviews)) {
            return [self findTextFieldWithView:subview];
        }
    }
    return nil;
}

@end

```

## 3. UITableView

### `iOS 13`设置`cell.contentView.backgroundColor`会影响`cell`的`selected`或者`highlighted`时的效果

例如：如果设置`cell.selectedBackgroundView`为自定义选中背景视图，并修改`cell.contentView.backgroundColor`为某种不透明颜色；`contentView`就会遮盖`cell.selectedBackgroundView`，最终导致无法看到自定义的`selectedBackgroundView`效果。

**修改建议：不设置`contentView.backgroundColor`时，默认值为`nil`；改为直接设置`cell`本身背景色**

```ObjectiveC
//  自定义cell.m文件

//  iOS 13之前正常
self.contentView.backgroundColor = [UIColor blueColor];
//  iOS 13改为
self.backgroundColor = [UIColor blueColor];
```

备注：`iOS 13`对`UITableView`还做了一些其他的修改，详细内容可查阅最底部 [参考内容1](https://developer.apple.com/documentation/ios_ipados_release_notes/ios_ipados_13_beta_6_release_notes?preferredLanguage=occ)，整个网页搜索`UITableViewCell`即可

## 4. UITabbar

- ### `Badge`文字显示不正常
  
  `Badge`默认字体大小，`iOS 13`从之前13号字体变为17号字体

  **修改建议：在初始化TabBarController时，在需要显示`Badge`的`ViewController`处调用`setBadgeTextAttributes:forState:`方法**

  ```ObjectiveC
  //  iOS 13需要添加
  if (@available(iOS 13, *)) {
      [viewController.tabBarItem setBadgeTextAttributes:@{NSFontAttributeName: CI_FONT_13} forState:UIControlStateNormal];
      [viewController.tabBarItem setBadgeTextAttributes:@{NSFontAttributeName: CI_FONT_13} forState:UIControlStateSelected];
  }
  ```

- ### 字体颜色异常

  `iOS 13`不设置`self.tabBar.barTintColor = [UIColor clearColor];`时字体颜色会显示蓝色，`iOS 13`之前设置与否无影响

  **修改建议：设置`tabBar.barTintColor`颜色为`UIColor clearColor]`**

  ```ObjectiveC
  //  自定义TabBarController.m文件
  - (void)viewDidLoad {
    [super viewDidLoad];

    self.tabBar.barTintColor = [UIColor clearColor];
    //  整个项目只有一种TabBarController时或者多个相同风格时可设置全局风格(其他情况不推荐)
    // [[UITabBar appearance] setBarTintColor:[UIColor clearColor]];
  }
  ```

## 5. UITabBarItem

- ### 播放gif，设置`ImageView`图片时注意设置图片的`scale`比例

  ```ObjectiveC
  //  根据路径取gif数据
  NSData *data = [NSData dataWithContentsOfFile:path];
  CGImageSourceRef gifSource = CGImageSourceCreateWithData(CFBridgingRetain(data), nil);
  size_t gifCount = CGImageSourceGetCount(gifSource);
  CGImageRef imageRef = CGImageSourceCreateImageAtIndex(gifSource, i,NULL);

  //  iOS 13之前
  UIImage *image = [UIImage imageWithCGImage:imageRef]
  //  iOS 13需要注意：添加scale比例(该imageView将展示该动图效果)
  UIImage *image = [UIImage imageWithCGImage:imageRef scale:image.size.width / CGRectGetWidth(imageView.frame) orientation:UIImageOrientationUp];

  CGImageRelease(imageRef);
  ```

- ### 播放gif，需找到设置的`ImageView`视图

    ```ObjectiveC
    //  UITabBarItem+CIChangePrivateSubviews.m文件
    //  修改TabbarItem系统私有子视图

    #import "UITabBarItem+CIChangePrivateSubviews.h"

    @implementation UITabBarItem (CIChangePrivateSubviews)

    /// 修改UITabBarSwappableImageView
    - (void)changeSwappableImageViewWithCompletionBlock:(void(^)(UIImageView *imageView))completionBlock {

        if (!completionBlock) {
            return;
        }

        UIView *tabBarButton = [self valueForKey:@"view"];
        for (UIView *subviews in tabBarButton.subviews) {
            if ([NSStringFromClass([subviews class]) isEqualToString:@"UITabBarSwappableImageView"]) {
                completionBlock((UIImageView *)subviews);
            }
        }
    }
    ```

- ### 只显示图片，没有文字时图片的位置调整

  `iOS 13`不需要调整`imageInsets`，图片会自动居中显示，如果设置会造成图片位置有些许偏移

  **修改建议：添加版本限制条件，只在`iOS 13`之前调用设置方法**

  ```ObjectiveC
  if (IOS_VERSION < 13.0) {
      viewController.tabBarItem.imageInsets = UIEdgeInsetsMake(5, 0, -5, 0);
  }
  ```

## 6. 弹出ViewController样式变化

### 模态展示`UIModalPresentationStyle`类型新增`UIModalPresentationAutomatic API_AVAILABLE(ios(13.0)) = -2`

用户调用`presentViewController:animated:completion:`方法弹出视图时，`iOS 13`效果变化更炫酷，可以在`iOS 13`系统App中体验到这种变化；
如果不希望使用这种效果，可利用`Runtime`方法，恢复设置`modalPresentationStyle`为`UIModalPresentationFullScreen`

```ObjectiveC
//  UIViewController+CIChangePresentStyle.m文件

#import "UIViewController+CIChangePresentStyle.h"

@implementation UIViewController (CIChangePresentStyle)

+ (void)load {

    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        //替换方法
        SEL originalSelector = @selector(presentViewController:animated:completion:);
        SEL swizzledSelector = @selector(ci_presentViewController:animated:completion:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);;
        BOOL didAddMethod =
        class_addMethod(class,
                        originalSelector,
                        method_getImplementation(swizzledMethod),
                        method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));

        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)ci_presentViewController:(UIViewController *)viewControllerToPresent animated:(BOOL)flag completion:(void (^)(void))completion {

    viewControllerToPresent.modalPresentationStyle = UIModalPresentationFullScreen;
    [self ci_presentViewController:viewControllerToPresent animated:flag completion:completion];
}

@end
```

## 7. 第三方SDK相关

- ### 友盟社会化分享SDK：使用“新浪微博完整版”闪退

  替换新浪微博最新的SDK版本[Github地址](https://github.com/sinaweibosdk/weibo_ios_sdk/releases)，友盟集成SDK暂时未更新

- ### 高德地图相关等其他第三方SDK更新

  查看相关Github或者官方SDK下载地址，更新最新的SDK即可

以上就是适配`iOS 13`时遇到问题的一些解决方案的建议。如果各位朋友也有一些新问题和解决方案，也可以在评论区留言，希望大家各抒己见，共同帮助解决问题和完善`iOS 13`适配总结。
  
## 参考内容

1. [iOS & iPadOS 13 Beta 6 Release Notes](https://developer.apple.com/documentation/ios_ipados_release_notes/ios_ipados_13_beta_6_release_notes?preferredLanguage=occ)
2. [友盟+推出全新SDK，适配iOS 13](https://info.umeng.com/detail?id=177&&cateId=1)
3. [新浪微博 IOS SDK](https://github.com/sinaweibosdk/weibo_ios_sdk)
