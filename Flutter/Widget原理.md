首先我们需要明白，Widget 是什么？这里有一个 *“总所周知”* 的答就是：**Widget 并不真正的渲染对象** 。是的，事实上在 Flutter 中渲染是经历了从 `Widget` 到 `Element` 再到 `RenderObject` 的过程。

我们都知道 Widget 是不可变的，那么 Widget 是如何在不可变中去构建画面的？上面我们知道，`Widget` 是需要转化为 `Element` 去渲染的，而从下图注释可以看到，事实上 **Widget 只是 Element 的一个配置描述** ，告诉 Element 这个实例如何去渲染。

那么 Widget 和 Element 之间是怎样的对应关系呢？从上图注释也可知： **Widget 和 Element 之间是一对多的关系** 。实际上渲染树是由 Element 实例的节点构成的树，而作为配置文件的 Widget 可能被复用到树的多个部分，对应产生多个 Element 对象。

那么`RenderObject` 又是什么？它和上述两个的关系是什么？从源码注释写着 `An object in the render tree` 可以看出到 `RenderObject` 才是实际的渲染对象，而通过 Element 源码我们可以看出：**Element 持有 RenderObject 和 Widget。**

```dart
abstract class Widget extends DiagnosticableTree {
  /// Initializes [key] for subclasses.
  const Widget({ this.key });

  /// Inflates this configuration to a concrete instance.
  ///
  /// A given widget can be included in the tree zero or more times. In particular
  /// a given widget can be placed in the tree multiple times. Each time a widget
  /// is placed in the tree, it is inflated into an [Element], which means a
  /// widget that is incorporated into the tree multiple times will be inflated
  /// multiple times.
  @protected
  @factory
  Element createElement();
```

```dart
bstract class Element extends DiagnosticableTree implements BuildContext {
  /// Creates an element that uses the given widget as its configuration.
  ///
  /// Typically called by an override of [Widget.createElement].
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;
    
     /// The render object at (or below) this location in the tree.
  ///
  /// If this object is a [RenderObjectElement], the render object is the one at
  /// this location in the tree. Otherwise, this getter will walk down the tree
  /// until it finds a [RenderObjectElement].
  RenderObject? get renderObject {
    RenderObject? result;
    void visit(Element element) {
      assert(result == null); // this verifies that there's only one child
      if (element._lifecycleState == _ElementLifecycle.defunct) {
        return;
      } else if (element is RenderObjectElement) {
        result = element.renderObject;
      } else {
        element.visitChildren(visit);
      }
    }
    visit(this);
    return result;
  }
 }
```

再结合下图，可以大致总结出三者的关系是：**配置文件 Widget 生成了 Element，而后创建 RenderObject 关联到 Element 的内部 `renderObject` 对象上，最后 Flutter 通过 RenderObject 数据来布局和绘制。** 理论上你也可以认为 RenderObject 是最终给 Flutter 的渲染数据，它保存了大小和位置等信息，Flutter 通过它去绘制出画面。

说到 `RenderObject` ，就不得不说 **`RenderBox`** ：`A render object in a 2D Cartesian coordinate system`，从源码注释可以看出，它是在继承 `RenderObject` 基础的布局和绘制功能上，实现了 “笛卡尔坐标系”：以 Top、Left 为基点，通过宽高两个轴实现布局和嵌套的。

RenderBox 避免了直接使用 `RenderObject` 的麻烦场景，其中 `RenderBox` 的布局和计算大小是在 **performLayout()** 和 **performResize()**这两个方法中去处理，很多时候我们更多的是选择继承 `RenderBox` 去实现自定义。

### 以RenderErrorBox 为例：

```dart
// Copyright 2014 The Flutter Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

import 'dart:ui' as ui show Paragraph, ParagraphBuilder, ParagraphConstraints, ParagraphStyle, TextStyle;

import 'box.dart';
import 'object.dart';

const double _kMaxWidth = 100000.0;
const double _kMaxHeight = 100000.0;

class RenderErrorBox extends RenderBox {
  RenderErrorBox([ this.message = '' ]) {
    try {
      if (message != '') {
        final ui.ParagraphBuilder builder = ui.ParagraphBuilder(paragraphStyle);
        builder.pushStyle(textStyle);
        builder.addText(message);
        _paragraph = builder.build();
      } else {
        _paragraph = null;
      }
    } catch (error) {
      // If an error happens here we're in a terrible state, so we really should
      // just forget about it and let the developer deal with the already-reported
      // errors. It's unlikely that these errors are going to help with that.
    }
  }

  /// The message to attempt to display at paint time.
  final String message;

  late final ui.Paragraph? _paragraph;

  @override
  double computeMaxIntrinsicWidth(double height) {
    return _kMaxWidth;
  }

  @override
  double computeMaxIntrinsicHeight(double width) {
    return _kMaxHeight;
  }

  @override
  bool get sizedByParent => true;

  @override
  bool hitTestSelf(Offset position) => true;

  @override
  Size computeDryLayout(BoxConstraints constraints) {
    return constraints.constrain(const Size(_kMaxWidth, _kMaxHeight));
  }

 
  static EdgeInsets padding = const EdgeInsets.fromLTRB(64.0, 96.0, 64.0, 12.0);

  /// The width below which the horizontal padding is not applied.
  ///
  /// If the left and right padding would reduce the available width to less than
  /// this value, then the text is rendered flush with the left edge.
  static double minimumWidth = 200.0;

  /// The color to use when painting the background of [RenderErrorBox] objects.
  ///
  /// Defaults to red in debug mode, a light gray otherwise.
  static Color backgroundColor = _initBackgroundColor();

  static Color _initBackgroundColor() {
    Color result = const Color(0xF0C0C0C0);
    assert(() {
      result = const Color(0xF0900000);
      return true;
    }());
    return result;
  }

 
  static ui.TextStyle textStyle = _initTextStyle();

  static ui.TextStyle _initTextStyle() {
    ui.TextStyle result = ui.TextStyle(
      color: const Color(0xFF303030),
      fontFamily: 'sans-serif',
      fontSize: 18.0,
    );
    assert(() {
      result = ui.TextStyle(
        color: const Color(0xFFFFFF66),
        fontFamily: 'monospace',
        fontSize: 14.0,
        fontWeight: FontWeight.bold,
      );
      return true;
    }());
    return result;
  }

  /// The paragraph style to use when painting [RenderErrorBox] objects.
  static ui.ParagraphStyle paragraphStyle = ui.ParagraphStyle(
    textDirection: TextDirection.ltr,
    textAlign: TextAlign.left,
  );

  @override
  void paint(PaintingContext context, Offset offset) {
    try {
      context.canvas.drawRect(offset & size, Paint() .. color = backgroundColor);
      if (_paragraph != null) {
        double width = size.width;
        double left = 0.0;
        double top = 0.0;
        if (width > padding.left + minimumWidth + padding.right) {
          width -= padding.left + padding.right;
          left += padding.left;
        }
        _paragraph!.layout(ui.ParagraphConstraints(width: width));
        if (size.height > padding.top + _paragraph!.height + padding.bottom) {
          top += padding.top;
        }
        context.canvas.drawParagraph(_paragraph!, offset + Offset(left, top));
      }
    } catch (error) {
      // If an error happens here we're in a terrible state, so we really should
      // just forget about it and let the developer deal with the already-reported
      // errors. It's unlikely that these errors are going to help with that.
    }
  }
}

```

RenderErrorBox  继承 RenderBox ，最主要的代码在paint(PaintingContext context, Offset offset)  方法中，使用Canvas进行界面的绘制

综合上述情况，我们知道：

- Widget 只是显示的数据配置，所以相对而言是轻量级的存在，而 Flutter 中对 Widget 的也做了一定的优化，所以每次改变状态导致的 Widget 重构并不会有太大的问题。
- RenderObject 就不同了，RenderObject 涉及到布局、计算、绘制等流程，要是每次都全部重新创建开销就比较大了。

所以针对是否每次都需要创建出新的 Element 和 RenderObject 对象，Widget 都做了对应的判断以便于复用，比如：在 `newWidget` 与`oldWidget` 的 *runtimeType* 和 *key* 相等时会选择使用 `newWidget` 去更新已经存在的 Element 对象，不然就选择重新创建新的 Element。

由此可知：**Widget 重新创建，Element 树和 RenderObject 树并不会完全重新创建。具体的可以查看绘制原理一文**

看到这，说个题外话：*那一般我们可以怎么获取布局的大小和位置呢？*

首先这里需要用到我们前文中提过的 `GlobalKey` ，通过 key 去获取到控件对象的 `BuildContext`，而我们也知道 `BuildContext` 的实现其实是 `Element`，而`Element`持有 `RenderObject` 。So，我们知道的 `RenderObject` ，实际上获取到的就是 `RenderBox` ，那么通过 `RenderBox` 我们就只大小和位置了。

```dart
  showSizes() {
    RenderBox renderBoxRed = fileListKey.currentContext.findRenderObject();
    print(renderBoxRed.size);
  }

  showPositions() {
    RenderBox renderBoxRed = fileListKey.currentContext.findRenderObject();
    print(renderBoxRed.localToGlobal(Offset.zero));
  }
```