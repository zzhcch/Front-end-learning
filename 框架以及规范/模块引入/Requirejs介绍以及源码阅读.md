# Reuqirejs介绍以及源码阅读
Requirejs是AMD规范的比较好的实践，在看Requirejs之前先看下AMD的规范

## AMD规范
看的是github上的[中文翻译的规范](https://github.com/amdjs/amdjs-api/wiki/AMD-(%E4%B8%AD%E6%96%87%E7%89%88))。

就是很简单的一个define方法

```javascript
  define(id?, dependencies?, factory);
```

 - id为这个定义的模块的名字，如果不提供，默认为指定的脚本的名字，如果提供了，必须是唯一的
 - 二参为依赖，就是模块需要先执行的模块。如果不提供或者提供默认的，就默认是["require", "exports", "module"]，就会扫描工厂方法的require来获得依赖。就是CommonJS的写法了(这个不就是CMD吗？)
 - 三参为函数，在所有的依赖被加载后会自动执行

任何全局函数必须有一个amd属性来标识遵循AMD编程接口。

## Requirejs
这个加载器主要解决的问题：

 - 大量JS引入页面的时候会导致页面假死，用了之后就变成异步的了
 - 还有处理各模块之间的依赖，因为不用的话可能JS的引入顺序需要自己控制好

### Requirejs使用
就是引一下requirejs，然后data-main指明入口文件。就像这样：

```html
<script src="require.js" data-main="main.js"></script>
```

然后我们在入口文件里面就可以申明依赖以及函数了。

```javascript
//main.js
//引了一个模块sec
requirejs(["sec"],function(sec){
   sec.action()
});
//sec.js
//这里按照commonjs的形式让requirejs自己去搜一遍，再引入third.js
define(function (require, exports, module) {
    var third =require('third');
    third();
    module.exports.action = function () {console.log(22)};
});
//third.js
define(function (require, exports, module) {
    module.exports = function () {console.log(11)};
});
```

入口文件里写requirejs或者define其实都行的。只是requirejs显示是在入口文件。这一这里的sec.js等等都是异步加载的，不会导致页面死在那里。

### 文件的路径问题
我们可以在主入口js中使用requirejs.config来进行配置。

```javascript
requirejs.config({
   baseUrl: "js/lib",
　 paths: {
　　  "sec": "sec.min"
　 }
})
```

我们可以配置基本的url，可以针对特别的模块单独定义路径，这里的路径还可以是网络请求的位置。

### 加载非amd规范的的模块
我们引入的模块都必须是define申明好的，如果不是的话，得在require里面申明一下。exports为对外输出的变量

```javascript
   require.config({
　　　　shim: {
　　　　　　'underscore':{
　　　　　　　　exports: '_'
　　　　　　},
　　　　　　'backbone': {
　　　　　　　　deps: ['underscore', 'jquery'],
　　　　　　　　exports: 'Backbone'
　　　　　　}
　　　　}
　　});
```

tudo:这个的写法得去看一看

### require.js的插件
提供了domready这种类似的插件

tudo:这些个插件也可以看下怎么写的

```javascript
require(['domready!'], function (doc){
　　　　// called once the DOM is ready
});
```

### 写一个require
在阅读源码之前，先试着实现一把。[链接](https://github.com/panyifei/Front-end-learning/tree/master/Demo)

先实现data-main的主入口，很简单就是新建了一个script，然后加载data-mian的申明

```javascript
var scripts = document.getElementsByTagName('script');
var sLength = scripts.length;
var mainJs;
for(var i=0;i<sLength;i++){
    var name = scripts[i].getAttribute("data-main");
    if(name){
        mainJs = name;
    }
}
var mainScript = document.createElement('script');
mainScript.src = mainJs;
document.body.appendChild(mainScript);
```

然后就是声明define方法，我最开始的时候使用的是onload来通知，这样子可以解决一层的依赖。就是如果main依赖了a和b，是可以的。但是如果b依赖了c，这种方法就失败了。

```javascript
//allModule用来保存所有加载的模块
var allModule = {};
//主要的define方法
var define = function(id,array,cb){
    var length = array.length;
    if(length > 0){
        var i = 0;
        array.forEach(function(value,index,array){
            var tempScript = document.createElement('script');
            tempScript.src = array[index]+'.js';
            //目前用的onload来监听加载完毕，但是如果有依赖的话，就不行了
            tempScript.onload = function(){
                i++;
                finish();
            }
            document.body.appendChild(tempScript);
        })
        function finish(){
            if(i == length){
                var modules =[];
                for(var x = 0 ; x<array.length;x++){
                    modules[x] = allModule[array[x]];
                }
                allModule[id] = cb.apply(null ,modules);
            }
        }
    }else{
        allModule[id] = cb();
    }
};
```

于是我不用onload来通知，选择在不依赖其他的模块调用完成时来进行通知，并且引用过他的模块尝试执行callback，如果这个模块所需要的都加载完了就可以执行。一层层的向上通知。

```javascript
//allModule用来保存所有加载的模块
var allModule = {};
//主要的define方法
function _registerModule(id,dependence,father){
    if(!allModule[id]){
        allModule[id]= {};
        var newModule = allModule[id];
        newModule.func = undefined;//模块的结果
        newModule.dependence = dependence;//依赖的模块
        newModule.dependenceLoadNum = 0;//已经依赖的模块
        newModule.finishLoad = function(){};//完成load之后触发的方法
        if(father){
            if(newModule.referrer){
                newModule.referrer.push(father);
            }else{
                newModule.referrer = [];//被引用到的模块
                newModule.referrer.push(father);
            }
        }
    }else{
        var newModule = allModule[id];
        if(father){
            newModule.referrer.push(father);
        }
        if(dependence){
            newModule.dependence = dependence;
            newModule.dependenceLoadNum = 0;
        }
    }
}
var define = function(id,array,cb){
    _registerModule(id,array,'',cb);
    array.forEach(function(value,index,array){
        _registerModule(array[index],[],id,cb);
    });
    var thisModule = allModule[id];
    if(array.length > 0){
        array.forEach(function(value,index,array){
            var tempScript = document.createElement('script');
            tempScript.src = array[index]+'.js';
            document.body.appendChild(tempScript);
        })
        thisModule.finishLoad = _finish;
    }else{
        thisModule.func = cb();
        _refererFinish();
    }
    function _finish(){
        thisModule.dependenceLoadNum++;
        if(thisModule.dependenceLoadNum == thisModule.dependence.length){
            var modules =[];
            for(var x = 0 ; x < array.length; x++){
                modules[x] = allModule[array[x]].func;
            }
            thisModule.func = cb.apply(null ,modules);
        }
        _refererFinish();
    }
    function _refererFinish(){
        if(thisModule.referrer){
            thisModule.referrer.forEach(function(value,index,array){
                allModule[array[index]].finishLoad();
            });
        }
    }
};
```

这样子基本实现了API，现在再看下requirejs是如何做的。

### 看源码



参考：

  http://www.ruanyifeng.com/blog/2012/11/require_js.html
