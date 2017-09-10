## 测试浏览器

**Chrome**: 61.0.3163.73

**Safari**: 10.0 （IOS 10.3.3）

## 1. IOS overflow: scroll 全屏滚动出界

### 1.1 出现场景

滑动到最顶部(最底部)的时候，停下，然后继续向上滑动(向下滑动)

### 1.2 解决方案

+ 手动设置滑到边界时的`scrollTop`（[scrollFix](https://github.com/joelambert/ScrollFix)）

当快滑到上边界或者下边界的值时，手动设置`scrollTop`与达到边界时相差一像素**(上边界时：`scrollTop = 1`, 下边界时：`scrollTop = elem.scrollHeight - elem.offsetHeight - 1`)**，这样就不会触发出界的极限条件。

具体实现：

```javascript
var ScrollFix = function(elem) {
	// Variables to track inputs
	var startY, startTopScroll;
	
	elem = elem || document.querySelector(elem);
	
	// If there is no element, then do nothing	
	if(!elem)
		return;

	// Handle the start of interactions
	elem.addEventListener('touchstart', function(event){
		startY = event.touches[0].pageY;
		startTopScroll = elem.scrollTop;
		
		if(startTopScroll <= 0)
			elem.scrollTop = 1;

		if(startTopScroll + elem.offsetHeight >= elem.scrollHeight)
			elem.scrollTop = elem.scrollHeight - elem.offsetHeight - 1;
	}, false);
};
```

**注：1. 这个方法只能部分防止，在某些时候还是会触发出界。2. 有说在全局滚动下和局部滚动下会有差异，但是就我测试的情况来说，差异并不是特别大。可能是版本太高的原因，具体结论还待测试更多机型。**

- 头部由`static`变为`fixed`（测试效果貌似更好）

```css
.toolbar {
	-webkit-box-sizing: border-box;
	padding: 1em;
	background: #222;
	color: #fff;
	font-weight: bold;
	text-align: center;
	height: 50px;
  
  	/* 添加fixed头部 */
	position: fixed;
	top: 0;
	z-index: 1;
	width: 100%;
}
```

## 2. IOS通过脚本使输入框聚焦，无法弹出键盘

### 2.1 出现场景

看如下代码：

```javascript
// html
<input type="email" class="form-control" id="inputEmail3" placeholder="Email">
<button id="submitBtn" class="btn btn-default">Sign in</button>
  
// script
var inputEmail3 = document.querySelector('#inputEmail3');
var submitBtn = document.querySelector('#submitBtn');

// way1
setTimeout(() => {
  	inputEmail3.focus();
}, 2000);
```

这种方式下：在`IOS`上输入框聚焦确没有办法弹出键盘

### 2.2 解决方案

爬墙爬到这么一个[issue](https://github.com/jquery/jquery-mobile/issues/3016)，3楼`eddiemonge`老哥说到了，在`IOS`下除非用户手动触发了输入框的`focus`事件，才会触发键盘，至于设置定时器也是不管用的；**但是**，手动点击一个按钮，在按钮的操作中再来执行`focus`事件倒是管用的。他还给出了一个[http://jsbin.com/inunis/8/edit?html,js,output](http://jsbin.com/inunis/8/edit?html,js,output)

```javascript
// 这样是可以弹出键盘的
submitBtn.onclick = function(e) {
  	e.preventDefault();
  	inputEmail3.focus();
}
```

## 3. IOS光标不跟随输入框移动

### 3.1 艰辛历程

我为什么会关注这个问题：那是因为我**（这里省略一万个草泥马）也遇到了这个问题呀，容我细细说来。

我有一个登录页面，在聚焦之后需要往上弹一下，`android`上正常，然后`IOS`上还同时引出了一个`BUG`：输入框上去了，但是光标却在下面闪。怎么办呢？当然是靠想办法解决呀，后来我就想在输入框上贴一层蒙版，点击了之后消失，同时在点击操作中，等到动画结束之后再执行输入框的`focus`，行不行呢？好期待。。。

`html`代码是这样的：

```javascript
// ... 这里省略若干
<div class="col-sm-10">
  	<input type="email" class="form-control" id="inputEmail3" placeholder="Email" />
    <div class="input-overlay" id="overlay"></div>
</div>
```

样式：

```css
.input-overlay {
  	position: absolute;
  	top: 0;
  	left: 0;
  	width: 100%;
  	height: 100%;
  	background-color: rgba(255, 0, 0, 0.3);
  	z-index: 2;
}
```

脚本：

```javascript
overlay.addEventListener('touchstart', function(e) {
  	e.preventDefault();
  	testForm.style.transform = 'translate3d(0, 0, 0)';
			
  	overlay.style.display = 'none';
  	overlay.style.zIndex = -1;

  	setTimeout(function() {
        inputEmail3.focus();
    }, 200);
});
```

正所谓想象是好样的，但是实际行使起来却不是那么令人满意。**是的，毫无效果**。后来我想，是不是可以模拟一个事件，再触发一次点击，然后代码是这样的：

```javascript
function mockEvent(fn) {
    var createDiv = document.createElement('div');
    createDiv.style.display = 'none';
    document.body.appendChild(createDiv);

  	// 未做兼容
    var e = document.createEvent('MouseEvent'); 
    e.initEvent('xx', true, true); 

    createDiv.addEventListener('xx', function() {
      fn && fn();
      createDiv.remove();
    });

    createDiv.dispatchEvent(e); 
}

overlay.addEventListener('click', function(e) {
  	// ...

  	setTimeout(function() {
        mockEvent(function() {
            inputEmail3.focus();
        });
    }, 200);
});
```

答案依然是：**不行**。然后我想，是不是`setTimeout`的原因，只要存在延迟的情况下就不行。结果我去这么试了一下，将之前的按钮直接点击方式改为200`ms`之后执行`focus`。

```javascript
submitBtn.onclick = function(e) {
  	e.preventDefault();
  	setTimeout(function() {
        inputEmail3.focus();
    });	
}
```

果然，只要设置延迟就不起效果了。顿时突然想到**移动端点透事件**貌似有个`300ms`延迟执行。虽然点透事件在移动端会被处理掉，然而我只是想验证一下我的猜想。然后我又这么写：

```javascript
// html
<div class="col-sm-10">
  	<input type="email" class="form-control" id="inputEmail3" placeholder="Email" />
    <div class="input-overlay" id="overlay"></div>
	<a class="input-link" href="javascript:;" id="link">link</a>
</div>
```

在`overlay`下面放一个`link`，然后在`overlay`上绑定`touchstart`事件，在`link`上绑定`click`事件。这样在上层的遮罩去掉之后，就可以`300ms`后执行下面的`link`层中的事情，那么也算是用户真正地触发的点击行为，美滋滋。结果我在代码中加了这个东西：

```javascript
overlay.addEventListener('touchstart', function(e) {
    // ...
});

link.addEventListener('click', function() {
    link.style.display = 'none';
    link.style.zIndex = -1;

    inputEmail3.focus();
});
```

尼玛呀，还是不行，绝望了。

然而。。。

天生不死心，又去爬墙呀。输入`inupt move while cursor to stay where it was`。下面讲解决方案。

### 3.2 解决方案

我找到了这样的一个[issue](https://github.com/cubiq/iscroll/issues/178)。在其中的描述是：他的内容中有一输入框，然后`focus`，当滑动内容时，光标不跟随移动，而在此输入的时候，光标又会回到输入框中。情况应该和我类似。

`robby says`

> I also have this problem.
> It is apparently related to the use of css transforms.
>
> I have fixed it with this hack workaround that forces redrawing as you scroll to eliminte this issue:
> CSS:
>
> ```
> input {
>   text-shadow: rgba(0,0,0,0) 0px 0px 0px;
> }
> input.force-redraw {
>   text-shadow: none;
> }
>
> ```
>
> JS:
>
> ```
> myScroll = new iScroll('wrapper', { 
>   onScrollMove: function() {
>     $('input').toggleClass('force-redraw');
>   }
> });
> ```

是的，有木有很激动。于是我这样写：

```javascript
// css
input {
  	text-shadow: rgba(0,0,0,0) 0px 0px 0px;
}
input.force-redraw {
  	text-shadow: none;
}

// javascript
inputEmail3.addEventListener('focus', function() {
    testForm.style.transform = 'translate3d(0, 0, 0)';
    setTimeout(() => {
      inputEmail3.className = 'form-control force-redraw';
    }, 300);
});
```

**效果大体上实现了**，但是仍然有瑕疵。就是必须设置延迟`300ms`以上，不然，光标重绘不正常，而且光标有明显的移动过程。**所以如果童鞋们如果发现有什么更好的办法，还望不吝赐教。**

另外，如果一个页面中有输入框，聚焦之后，滑动过程中在`IOS`上可能会出现不流畅的问题，其实可以这么做：监测页面的`touchmove`事件，如果当前页面存在着输入框被`active`，那么直接让其`blur`，保证滑动过程中没有输入框被聚焦。（不过以我的测试情况来看，在`chrome`和`safari`上滑动的时候输入框不再被激活，类似在`PC`端滑动的时候采用了蒙版或者`points-event: none;`的效果）

```javascript
var thisFocus;
var content = document.querySelector('#content');

var inputs = document.getElementsByTagName('input');
for (var i = 0; i < inputs.length; i++) {
    var input = inputs[i];
    input.onfocus = focused;
}

function focused(e) {
  	thisFocus = e.target;
}

content.addEventListener('touchmove', function() {
    if (thisFocus) {
        thisFocus.blur();
        thisFocus = null;
    }
});
```

## 4. IOS输入框聚焦后页面整体上移，头部顶出

### 4.1 出现场景

页面中有`fixed`头部，输入框，并且输入框靠下时，当输入框`focus`的时候，会将整个页面上移，导致头部被顶出去。[fixed position div freezes on page](https://stackoverflow.com/questions/10659891/fixed-position-div-freezes-on-page-ipad)

### 4.2 解决方案

原因大致是：`ios`自带的**输入居中**效果，而带有`fixed`头部在页面被顶上去的同时没有重新计算位置，导致显示错误。那么可以具体分这几步来解决：

- 没有`focus`的时候采用`fixed`固定头部
- 不要让用户进行缩放
- 当输入框`focus`时，采用绝对定位头部，同时使用`window.pageYOffset`来计算滑动的距离，设置头部的`top`值
- 滑动的时候，监听`scroll`方法，动态设置头部`top`值
- 失去焦点的时候，重新将头部恢复至`fixed`定位
- 滑动时，如果头部结构太复杂，可能会引起固定不流畅(会跟着滚动)

代码请往这里看：

```javascript
var isFocused, focusedResizing, ticking = false;

window.tryfix = function() {
    var inputs = document.getElementsByTagName('input');
    for (var i = 0; i < inputs.length; i++) {
        input = inputs[i];
        input.onfocus = focused;
        input.onblur = blured;
    }
    window.onscroll = onScroll;
}

function focused(event) {
    isFocused = true;
    scrolled();
}

function blured(event) {
    isFocused = false;
    var headStyle = document.getElementById('header').style;
    var footStyle = document.getElementById('footer').style;
    if (focusedResizing) {
        focusedResizing = false;
        headStyle.position = 'fixed';
        headStyle.top = 0;
        footStyle.display = 'block';
    }
}

function onScroll() {
    if (!ticking) {
        requestAnimationFrame(scrolled);
        ticking = true;
    }
}

function scrolled() {
    var headStyle = document.getElementById('header').style;
    var foot = document.getElementById('footer');
    var footStyle = foot.style;
    if (isFocused) {
        if (!focusedResizing) {
            focusedResizing = true;
            headStyle.position = 'absolute';
            footStyle.display = 'none';
        } 
        headStyle.top = window.pageYOffset + 'px';
        // window.innerHeight wrong
        //var footTop = window.pageYOffset + window.innerHeight - foot.offsetHeight;
        //footStyle.bottom = (document.body.offsetHeight - footTop) + 'px';
    }
    ticking = false;
}

tryfix();
```

另外如果页面缩放，也会引起头部定位不正常。详情可以看[这里](http://bradfrost.com/demo/fixed/index.html)，关于`anroid`上`fixed`的支持情况，可以看[这里](http://bradfrost.com/blog/mobile/fixed-position/)

