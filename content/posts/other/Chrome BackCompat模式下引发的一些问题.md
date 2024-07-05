+++
title = 'Chrome BackCompat模式下引发的一些问题'
date = 2017-11-08 14:22:32
draft = true
tags = ["program","frontend"]
categories = ["program"] 
+++
　　今天在使用vue和layer写一个页面的时候遇到一个诡异的问题，在调用layer.open()时弹出层并没有出现，但是背后的遮罩层出现了，在查看html代码后发现弹出层style的top值有问题，于是翻阅layer的代码，发现计算弹出层位置的坐标是通过调用jquery的height()函数计算当前窗口可视的高度，再减去弹出层的高度除以2得来的。

```JavaScript
  var that = this, config = that.config, layero = that.layero;
  var area = [layero.outerWidth(), layero.outerHeight()];
  var type = typeof config.offset === 'object';
  that.offsetTop = (win.height() - area[1])/2;
  that.offsetLeft = (win.width() - area[0])/2;
```
　　在控制台中使用$(window).height()测试后发现结果与实际不符，猜测可能是jquery版本太低有bug或者jquery与我某个js冲突了，去翻阅jquery的代码，jquery中height()的实现代码如下：

```JavaScript
jQuery.each({ Height: "height", Width: "width" }, function(name, type) {
  jQuery.each({ padding: "inner" + name, content: type, "": "outer" + name }, function(defaultExtra, funcName) {
    // margin is only for outerHeight, outerWidth
    jQuery.fn[funcName] = function(margin, value) {
      var chainable = arguments.length && (defaultExtra || typeof margin !== "boolean"),
        extra = defaultExtra || (margin === true || value === true ? "margin" : "border");

      return access(this, function(elem, type, value) {
        var doc;

        if (jQuery.isWindow(elem)) {
          // As of 5/8/2012 this will yield incorrect results for Mobile Safari, but there
          // isn't a whole lot we can do. See pull request at this URL for discussion:
          // https://github.com/jquery/jquery/pull/764
          return elem.document.documentElement["client" + name];
        }

        // Get document width or height
        if (elem.nodeType === 9) {
          doc = elem.documentElement;
          // Either scroll[Width/Height] or offset[Width/Height] or client[Width/Height], whichever is greatest
          // unfortunately, this causes bug #3838 in IE6/8 only, but there is currently no good, small way to fix it.
          return Math.max(
            elem.body["scroll" + name], doc["scroll" + name],
            elem.body["offset" + name], doc["offset" + name],
            doc["client" + name]
          );
        }

        return value === undefined ?
          // Get width or height on the element, requesting but not forcing parseFloat
          jQuery.css(elem, type, extra) :

          // Set width or height on the element
          jQuery.style(elem, type, value, extra);
      }, type, chainable ? margin : undefined, chainable, null);
    };
  });
});
```
　　因为layer使用的是$(window).height(),所以对应执行的是
```JavaScript
if (jQuery.isWindow(elem)) {
    // As of 5/8/2012 this will yield incorrect results for Mobile Safari, but there
    // isn't a whole lot we can do. See pull request at this URL for discussion:
    // https://github.com/jquery/jquery/pull/764
    return elem.document.documentElement["client" + name];
}
```
　　换言之，jquery直接调用了**window.document.documentElement.clientHeight**来获取当前窗口显示的高度。  
　　那么问题就变成了为什么document.documentElement.clientHeight返回的结果是个错误的值。经过查询后发现错误原因是因为我未在\<html>标签中加入`!DOCTYPE`导致浏览器启用了怪异模式(quirks mode)。于是在html标签中加入!DOCTYPE后问题成功解决。

附CSSOM上关于clientHeight的处理顺序：
> The clientHeight attribute must run these steps:
> 1. If the element has no associated CSS layout box or if the CSS layout box is inline, return zero.
> 2. If the element is the root element and the element’s node document is not in quirks mode, or if the element is the HTML body element and the element’s node document is in quirks mode, return the viewport height excluding the size of a rendered scroll bar (if any).
> 3. Return the height of the padding edge excluding the height of any rendered scrollbar between the padding edge and the border edge, ignoring any transforms that apply to the element and its ancestors."  
> 
> CSSOM View Module  https://drafts.csswg.org/cssom-view/#dom-element-clientheight

但在实际测试过程中发现：  
在怪异模式下，document.body.clientHeight均显示可视高度。但当可视高度大于body高度时，document.documentElement.clientHeight显示为可视高度；当可视高度小于等于body高度时，document.documentElement.clientHeight显示为body高度，原因还有待去考究。  
**测试浏览器：Chrome 版本 61.0.3163.100（正式版本） （64 位）**
**测试系统：Window 10.0.15063**

总结一下:

1. 不添加DOCTYPE标识会使浏览器进入怪异模式，在怪异模式下浏览器的显示方法会与标准模式有许多的不同。怪异模式的具体差异详见——[Quirks Mode](https://quirks.spec.whatwg.org/)
2. 怪异模式下，document.body.clientHeight显示为可视高度。当可视高度大于body高度时，document.documentElement.clientHeight显示为可视高度。当可视高度小于等于body高度时，，document.documentElement.clientHeight显示为body高度。