# JavaScript/JQuery屏蔽网页鼠标右键菜单及禁止选择复制
原作者：王晔 地址：
###粗暴版的限制选择复制或者禁止鼠标右键###
如何粗暴的限制或者禁止浏览者复制网页上的文字，正常的防止浏览者复制文字，肯定是想到禁用用户的某些特定的操作，比如鼠标右键，选择，复制等等，而这些操作对应了相应的脚本事件，只要给这些事件加上一个方法，让其返回false就可以“吃”掉这个操作了，一般粗暴的禁止复制的脚本代码如下：
```
window.onload = function() {
    with(document.body) {
      oncontextmenu=function(){return false}
      ondragstart=function(){return false}
      onselectstart=function(){return false}
      onbeforecopy=function(){return false}
      onselect=function(){document.selection.empty()}
      oncopy=function(){document.selection.empty()}
    }
}
```
为什么称这个方法为粗暴版的呢？因为使用这个方法禁止鼠标右键后你会发现网页上任何控件都无法右击或者选择了，网页似乎成了死板的图片，也许你会觉得无所谓，但是对于input、textarea文本框这类字符输入控件就有很大的关系了，这些地方不能限制用户的右键及选择复制操作。

###合理判断要限制的HTML标签元素###
如何判断当前处理的层所在的元素标签呢，也就是说得到鼠标当前所在的HTML Tag，这里我们以oncontextmenu为例，其实在document.body.oncontextmenu传入的函数有一个参数我们略去了，完整的写法应该是document.body.oncontextmenu=function(e){}这里的e是JavaScript中的Event事件对象，在IE中可能是通过window.event获取的。通过这个事件对象可以获取触发事件时鼠标所在的HTML Tag，我们可以判断是不是我们要忽略处理元素标签，这里我提供一个函数如下：
```
var isTagName = function(e, whitelists) {
      e = e || window.event;
      var target = e.target || e.srcElement;
      Array.prototype.contains = function(elem)
        {
           for (var i in this)
           {
               if (this[i].toString().toUpperCase() == elem.toString().toUpperCase()) return true;
           }
           return false;
        }
      if (whitelists && !whitelists.contains(target.tagName)) {
        return false;
      }
      return true;
};
```
这里的e是事件对象event，target是事件对象所引用的元素对象，当然这里两个变量都采取了兼容IE的写法，具体可以参考《How can I make event.srcElement work in Firefox and what does it mean?》；这里的whitelists是白名单HTML元素标签Tag名，比如['INPUT', 'TEXTAREA']，表示将输入文本框input和textarea加入判断，如果当前事件元素是的话就返回true，这样我们通过下面的代码可以选择性的屏蔽鼠标右键了：
```
document.body.oncontextmenu=function(e){
     return isTagName(e, ['A', 'TEXTAREA']);
}
```
###JQuery版的选择性屏蔽禁止文本选择###
同样的对于其他的操作也可以选择性的屏蔽，在JQuery支持的环境中我在StackOverflow找到了这么一篇文章《How to disable text selection using jQuery?》，虽然是讲解的禁止选择的，不过可以借鉴一下，具体代码如下：
```
(function($){
 
  $.fn.ctrlCmd = function(key) {
 
    var allowDefault = true;
 
    if (!$.isArray(key)) {
       key = [key];
    }
 
    return this.keydown(function(e) {
        for (var i = 0, l = key.length; i < l; i++) {
            if(e.keyCode === key[i].toUpperCase().charCodeAt(0) && e.metaKey) {
                allowDefault = false;
            }
        };
        return allowDefault;
    });
};
 
 
$.fn.disableSelection = function() {
 
    this.ctrlCmd(['a', 'c']);
 
    return this.attr('unselectable', 'on')
               .css({'-moz-user-select':'-moz-none',
                     '-moz-user-select':'none',
                     '-o-user-select':'none',
                     '-khtml-user-select':'none',
                     '-webkit-user-select':'none',
                     '-ms-user-select':'none',
                     'user-select':'none'})
               .bind('selectstart', false);
};
 
})(jQuery);
```
在使用上采取下面的代码：
```
$(':not(input,select,textarea)').disableSelection();
```
这样就可以除去input、select、textarea外禁止选择了，这段代码的技巧是除了采取selectstart外还给相关元素添加了某些特殊浏览器支持的CSS特性，这样基本可以实现大多数浏览器的兼容，同时这段代码还屏蔽了键盘按键选择Ctrl+A和Ctrl+C，不得不佩服作者周到的考虑。

###进一步完善：屏蔽鼠标点击操作###
我在测试这段代码时遇到一个问题就是点击除input、select、textarea外的元素时会全部选择页面，原文作者给出一个简单的方法就是在使用代码上附加.on('mousedown', false)，这样就屏蔽了鼠标的单击，使用代码变成如下：
```
$(':not(input,select,textarea)').disableSelection().on('mousedown', false);
```
但是问题又来了，我发现采取上述代码后，input,select,textarea也开始变得不正常起来，看样子屏蔽mousedown的特性应用到所有元素上了。现在转换一下思路，结合刚才我提出的方案，判断event对象来实现选择性屏蔽，我将代码修正如下：
```
$(':not(input,select,textarea)').disableSelection().on('mousedown', function(e) {
    var event = $.event.fix(e);
    var elem = event.target || e.srcElement;
    if (elem.tagName.toUpperCase() != 'TEXTAREA' && elem.tagName.toUpperCase() != 'INPUT') {
        e.preventDefault();
        return false;
    }
    return true;
});
```
这样textarea和input就不会限制mousedown了，我们将这段代码抽出为函数：
```
function jQuery_isTagName(e, whitelists) {
      e = $.event.fix(e);
      var target = e.target || e.srcElement;
      if (whitelists && $.inArray(target.tagName.toString().toUpperCase(), whitelists) == -1) {
        return false;
      }
      return true;
}
 
$(':not(input,select,textarea)').disableSelection().on('mousedown', function(e) {
    if (!jQuery_isTagName(e, ['INPUT', 'TEXTAREA'])) {
        e.preventDefault();
        return false;
    }
    return true;
});
```
JQuery版的选择性屏蔽禁止鼠标右键
对于右键菜单，我们可以这样处理：
```
$(document).bind("contextmenu",function(e){
    if (!jQuery_isTagName(e, ['INPUT', 'TEXTAREA'])) {
        e.preventDefault();
        return false;
    }
    return true;
});
```
