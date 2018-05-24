---
layout: post
title:  在 TAC 上使用 Tangram4TAC SDK
author: Longerian

---

## 背景

Tangram 客户端 SDK 对后端数据格式有标准化的需求，而后端接口格式千千万万，总不会是每个接口格式都能如你所愿，Tangram4TAC 就是为了解决这种问题构建，以标准化模型来在 TAC 在构建一个 Tangram 页面。 

## SDK 介绍

![](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/jpeg/95701d8d-61fb-44ae-bbbe-70dda875ead5.jpeg)

一期 SDK 非常简单，尽量精简抽象，主要提供了 Tangram 组件模型的定义和默认的数据转换输出服务（称之为 render）。如上图所示，橘色部分为 SDK 提供的核心模型对象，Cell 是组件定义，Container 是容器定义，Style 是样式定义。浅绿色部分为 Tangram 内置的布局类型组件定义。白色部分为业务自定义组件，不集成在 SDK 内部，用户可以自己扩展任意部分来实现自己的组件 model 对象。

![](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/jpeg/db9d3cda-73dc-4a2d-9ff2-5a76a278c4a6.jpeg)

Cell、Container、Style 的属性定义也是按照现有 Tangram 的规范来定义，业务字段需要用户自己继承对应的类来扩展。

当我们用这些模型对象构建好 Tangram 组件树之后，就需要对外输出数据，一般情况下对外输出的不是原始的对象，可能是需要转换成 JSON 或者 Map 对象，过程中还需要对字段进行一些处理。SDK 内提供了几个辅助工具：

`@FieldExcluder` 注解：model 内部可能会有一些用来保存状态，传递上下文信息的字段，并不需要最终输出属性，通过此注解将这些属性在转换过程中过滤掉，减少冗余数据。
`@FieldNameMapper` 注解：默认情况下，我们期望 model 内部的属性定义名和输出字段名一致，但也有存在不兼容的情况，比如 Style 里的背景色字段要求按照 CSS 的规范输出 background-color 属性名，而 JAVA 的变量名不支持这种规范，所以可以通过该注解提供一个别名，比如：

```
@FieldNameMapper(key = "background-color")
protected String backgroundColor;
```

SDK 默认的 render 类 `DefaultRender` 遵循上述原则，并将 Cell 对象转换成 Map 结构。用户也可以提供自定义的 render 对象给 Cell 更改转换逻辑。

## 举例说明
上述就是 SDK 的核心功能，非常的精简，下面用一个例子说明使用流程。以天猫首页 icon 区改造为例。

1.实现一个 TmallCell 继承自 Cell，内部封装天猫客户端环境下扩展的公共业务字段。 

```
abstract public class TmallCell<T extends Style> extends Cell<T> {

    protected String action;

    protected String title;

    protected String imgUrl;

    public TmallCell() {

    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getImgUrl() {
        return imgUrl;
    }

    public void setImgUrl(String imgUrl) {
        this.imgUrl = imgUrl;
    }

    public String getAction() {
        return action;
    }

    public void setAction(String action) {
        this.action = action;
    }

}
```

2.实现 Icon 继承自 TmallCell，封装 icon 自身的数据。 

```
public class Icon extends TmallCell<IconStyle> {

    private String title;

    private String imgUrl;

    private String bizId;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getImgUrl() {
        return imgUrl;
    }

    public void setImgUrl(String imgUrl) {
        this.imgUrl = imgUrl;
    }

    public String getBizId() {
        return bizId;
    }

    public void setBizId(String bizId) {
        this.bizId = bizId;
    }

    @Override
    public String getType() {
        return "icon";
    }
}
```

3.实现 IconStyle 继承自 Style，封装 icon 自身的特有样式属性。 

```
public class IconStyle extends Style {

    protected String titleColor;

    public String getTitleColor() {
        return titleColor;
    }

    public void setTitleColor(String titleColor) {
        this.titleColor = titleColor;
    }
}
```

4.调用接口获取业务数据后，准备构造组件树，先构造布局容器对象，设置样式，提供一个默认的 render 对象。 

```
FiveColumnContainer fiveColumnContainer = new FiveColumnContainer();
fiveColumnContainer.setId("icon");
FlowStyle style = new FlowStyle();
style.setBackgroundColor("#FFFFFF");
style.setPadding(6, 12, 11, 12);
style.setMargin(-80, 0, -26, 0);
style.setCols(new float[]{10.1f, 102.1f});
style.sethGap(6);
style.setzIndex(5);
fiveColumnContainer.setStyle(style);

```

5.根据数据构造 Icon 组件对象，并设置样式和属性。 

```
for (int i = 0; i < 5; i++) {
    IconStyle iconStyle = new IconStyle();
    iconStyle.setTitleColor("#FF0000");
    if (i == 1) {
        iconStyle.setDisableReuse(true);
    }
    Icon icon = new Icon();
    icon.setStyle(iconStyle);
    icon.setId(String.valueOf(i));
    icon.setTitle("icon_" + i);
    icon.setImgUrl("https://gw.alicdn.com/tfs/TB1ISdWSFXXXXbFXXXXXXXXXXXX-146-147.png");
    icon.setAction("https://www.tmall.com/");
    fiveColumnContainer.addChild(icon);
}
```

6.调用 render 方法输出数据返回给 TAC。

```
Map<String, Object> result = (Map<String, Object>) fiveColumnContainer.render();
resultList.add(result);
```

上述流程总结起来如图：
![](https://private-alipayobjects.alipay.com/alipay-rmsdeploy-image/skylark/png/9ea7a752-e8b5-4c4c-bce1-6edbbb478480.png)



