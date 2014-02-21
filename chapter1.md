# Core methods核心方法（一）：$() #

	$()
	
	$(selector, [context])  ⇒ collection
	$(<Zepto collection>)  ⇒ same collection
	$(<DOM nodes>)  ⇒ collection
	$(htmlString)  ⇒ collection
	$(htmlString, attributes)  ⇒ collection v1.0+
	Zepto(function($){ ... }) 

## $()主要做以下几件事： ##

- domready
- dom选择器
- 将html字符串转化为zepto对象

### domready ###

**DEMO链接：**[/demo/domready.html](https://github.com/zzyss86/zepto-learn/blob/master/demo/domready.html"/demo/domready.html")

	var readyRE = /complete|loaded|interactive/;
	$.fn = {
	    ready: function(callback){
	      // IE下需要检测document.body元素已经创建
	      if (readyRE.test(document.readyState) && document.body) callback($)
	      else document.addEventListener('DOMContentLoaded', function(){ callback($) }, false)
	      return this
	    }
	}

IE下需要检测document.readyState为“complete|loaded|interactive”的一个，并且body元素已经成功创建，才能视为domready；而其它浏览器，直接监听document的DOMContentLoaded事件即可。

### dom选择器 及 将html字符串转化为zepto对象 ###

**代码如下（源码顺序有调整）：**

	// `$` will be the base `Zepto` object. When calling this
	// function just call `$.zepto.init, which makes the implementation
	// details of selecting nodes and creating Zepto collections
	// patchable in plugins.
	$ = function(selector, context){
	  return zepto.init(selector, context)
	}
	
	// `$.zepto.init` is Zepto's counterpart to jQuery's `$.fn.init` and
	// takes a CSS selector and an optional context (and handles various
	// special cases).
	// This method can be overriden in plugins.
	zepto.init = function(selector, context) {
	  var dom
	  // 参数为空时，返回空的zepto对象集
	  if (!selector) return zepto.Z()
	  // 优化字符串选择器
	  else if (typeof selector == 'string') {
		selector = selector.trim()
		// If it's a html fragment, create nodes from it
		// Note: In both Chrome 21 and Firefox 15, DOM error 12
		// is thrown if the fragment doesn't begin with <
		if (selector[0] == '<' && fragmentRE.test(selector))
		  dom = zepto.fragment(selector, RegExp.$1, context), selector = null
		// 如果有第二个参数context（上下文），在上下文中查找
		else if (context !== undefined) return $(context).find(selector)
		// CSS选择器
		else dom = zepto.qsa(document, selector)
	  }
	  // 如果传递的参数是函数，调用domready
	  else if (isFunction(selector)) return $(document).ready(selector)
	  // 如果返回的是zepto对象集，直接返回
	  else if (zepto.isZ(selector)) return selector
	  else {
		// 如果参数为数组，并过滤为null的成员项
		if (isArray(selector)) dom = compact(selector)
		// 如果参数为Object，用数组包裹
		else if (isObject(selector))
		  dom = [selector], selector = null
		// 如果参数是html代码片断，创建节点
		else if (fragmentRE.test(selector))
		  dom = zepto.fragment(selector.trim(), RegExp.$1, context), selector = null
		// 如果有第二个参数context（上下文），在上下文中查找
		else if (context !== undefined) return $(context).find(selector)
		// And last but no least, if it's a CSS selector, use it to select nodes.
		else dom = zepto.qsa(document, selector)
	  }
	  // 返回新的zepto对象集
	  return zepto.Z(dom, selector)
	}
	
	//返回zepto对象集
	// `$.zepto.Z` swaps out the prototype of the given `dom` array
	// of nodes with `$.fn` and thus supplying all the Zepto functions
	// to the array. Note that `__proto__` is not supported on Internet
	// Explorer. This method can be overriden in plugins.
	zepto.Z = function(dom, selector) {
	  dom = dom || []
	  dom.__proto__ = $.fn
	  dom.selector = selector || ''
	  return dom
	}
	
	// 判断对象是否是zepto对象集
	zepto.isZ = function(object) {
	  return object instanceof zepto.Z
	}
	
	//Array.filter()在数组中的每个项上运行一个函数，并将函数返回真值的项作为数组返回。简单的说就是用一个条件过滤掉不符合的数组元素，剩下的符合条件的元素组合成新的数组返回。
	var filter = [].filter;
	function compact(array) { return filter.call(array, function(item){ return item != null }) }
	
	//匹配html标签
	var fragmentRE = /^\s*<(\w+|!)[^>]*>/
	
	// `$.zepto.fragment` takes a html string and an optional tag name
	// to generate DOM nodes nodes from the given html string.
	// The generated DOM nodes are returned as an array.
	// This function can be overriden in plugins for example to make
	// it compatible with browsers that don't support the DOM fully.
	zepto.fragment = function(html, name, properties) {
	  var dom, nodes, container
	
	  // A special case optimization for a single tag
	  if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))
	
	  if (!dom) {
		if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
		if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
		if (!(name in containers)) name = '*'
	
		container = containers[name]
		container.innerHTML = '' + html
		dom = $.each(slice.call(container.childNodes), function(){
		  container.removeChild(this)
		})
	  }
	
	  if (isPlainObject(properties)) {
		nodes = $(dom)
		$.each(properties, function(key, value) {
		  if (methodAttributes.indexOf(key) > -1) nodes[key](value)
		  else nodes.attr(key, value)
		})
	  }
	
	  return dom
	}
	
	//CSS选择器
	// `$.zepto.qsa` is Zepto's CSS selector implementation which
	// uses `document.querySelectorAll` and optimizes for some special cases, like `#id`.
	// This method can be overriden in plugins.
	zepto.qsa = function(element, selector){
	  var found,
		  maybeID = selector[0] == '#',
		  maybeClass = !maybeID && selector[0] == '.',
		  nameOnly = maybeID || maybeClass ? selector.slice(1) : selector, // Ensure that a 1 char tag name still gets checked
		  isSimple = simpleSelectorRE.test(nameOnly)
	  return (isDocument(element) && isSimple && maybeID) ?
		( (found = element.getElementById(nameOnly)) ? [found] : [] ) :
		(element.nodeType !== 1 && element.nodeType !== 9) ? [] :
		slice.call(
		  isSimple && !maybeID ?
			maybeClass ? element.getElementsByClassName(nameOnly) : // If it's simple, it could be a class
			element.getElementsByTagName(selector) : // Or a tag
			element.querySelectorAll(selector) // Or it's not simple, and we need to query all
		)
	}


**涉及的功能点：**

1、CSS选择器

核心方法zepto.qsa(element, selector)大至过程：

1.1判断选择器为ID且为单个节点的选择器，直接调用document.getElementById

1.2判断nodeType不为1或9的父节点，返回空数组。nodeType==1为Element，nodeType==9为Document

1.3判断单节点选择器且为class的，调用element.getElementsByClassName

1.4判断单节点选择器为标签的，调用element.getElementsByTagName

1.5其它的复合选择器（带子节点，如 a > .class），调用element.querySelectorAll

**DEMO链接：**[/demo/qsa.html](https://github.com/zzyss86/zepto-learn/blob/master/demo/qsa.html"/demo/qsa.html")

**简化后的代码：**

	<div class="wrap">
		<a id="btn" href="#">click me</a>
	</div>
	<script type="text/javascript">
	var slice = [].slice; //相当于Array.prototype.slice
	var zepto = {};
	//匹配数字，字母，下划线，连接线"-"
	var simpleSelectorRE = /^[\w-]*$/; 
	//判断节点为document
	function isDocument(obj)   { return obj != null && obj.nodeType == obj.DOCUMENT_NODE }
	//选择器
	zepto.qsa = function(element, selector){
	  var found,
		  maybeID = selector[0] == '#',
		  maybeClass = !maybeID && selector[0] == '.',
		  nameOnly = maybeID || maybeClass ? selector.slice(1) : selector, // Ensure that a 1 char tag name still gets checked
		  isSimple = simpleSelectorRE.test(nameOnly) //判定当前选择器是简单的单个选择器
	  return (isDocument(element) && isSimple && maybeID) ?
		( (found = element.getElementById(nameOnly)) ? [found] : [] ) :
		(element.nodeType !== 1 && element.nodeType !== 9) ? [] :
		slice.call( //将nodelist转换为数组
		  isSimple && !maybeID ?
			maybeClass ? element.getElementsByClassName(nameOnly) : // If it's simple, it could be a class
			element.getElementsByTagName(selector) : // Or a tag
			element.querySelectorAll(selector) // Or it's not simple, and we need to query all
		)
	}
	
	var dom = zepto.qsa(document, '.wrap > #btn');
	alert(dom.length);
	</script>


2、html代码代码转换为dom节点并返回zepto对象集

核心方法zepto.fragment的大至过程：

