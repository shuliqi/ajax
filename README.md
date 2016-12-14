#问题再现：
在写看点账号和关键词监控的微博文章部分时，有一个上拉加载的功能，基本实现后在测试时发现：每当下拉的过快或者做重复下拉的操作，ajax就会请求不到数据并返回canceled的状态码。

#原因探究
经过排查，ajax请求被canceled是因为在ajax请求资源的时间内，后续的操作继续触发了ajax请求资源，因此请求操作被取消，所以解决问题关键就是如何防止重复发送ajax请求。

#解决方法
##1 节制型提交(throttle) 无论提交如何频繁，任意两次有效提交的间隔时间必定会大于或等于某一时间间隔；即以一定频率提交。
``` javascript
module.submit = throttle(150, function() {
  // todo
})
```
throttle释义：

throttle在英文中是节流阀的意思，顾名思义，就像原本一直流水的龙头，现在每隔一定时间开一下，以节流。也就是会说预先设定一个执行周期，当调用动作的时刻大于等于执行周期则执行该动作，然后进入下一个新周期。

使用场景： 鼠标移动，mousemove 事件；DOM 元素动态定位，window对象的resize和scroll 事件等。

demo:
``` javascript
    window.addEventListener("resize", throttle(callback, 3000, {leading:true}));
    window.addEventListener("resize", throttle(callback2, 0, {leading:false}));
    function callback ()  { console.count("Throttled"); }
    function callback2 () { console.count("Not Throttled"); }
    /**
    * 频率控制函数， fn执行次数不超过 1 次/delay
    * @param fn{Function}     传入的函数
    * @param delay{Number}    时间间隔
    * @returns {Function}     返回调用函数
    */
    function throttle(fn,delay,options) {
         var wait=false;
        if (!options) options = {};
        return function(){
            var that = this;
             if(!wait){
                 if (options.leading === false){
                     // 非截流操作
                     fn.apply(that)
                    }
                 else { 
                    // 截流操作
                    wait=true;
                    setTimeout(function () {
                       fn.apply(that);
                       wait=false;
                    },delay);
                }
            }
        }
    }
```
##2 懒惰型提交(debounce) 任意两次提交的间隔时间，必须大于一个指定时间，才会促成有效提交；即不给休息不干活。
``` javascript
module.submit = debounce(150, function() {  
  // todo
})
```
debounce释义：

bounce是名词弹力，或者动词反弹的意思， de-表否定/相反的前缀，形象低表示一直持续低压抑住不让反弹，当然最后还是得松手，那么反弹一次。 也就是说当调用动作n毫秒后，才会执行该动作，若在这n毫秒内又调用此动作则将重新计算执行时间。

使用场景： 文本输入keydown 事件，keyup 事件等。

demo：
``` javascript
    var resizeTimer=null;
    $(window).on('resize',function(){
           if(resizeTimer){
               clearTimeout(resizeTimer)
           }
           resizeTimer=setTimeout(function(){
               console.log("window resize");
           },1000);
       }
    );
 ```
throttle和debounce的区别理解
福兮,祸之所倚；祸兮,福之所伏。 ——《老子》

##二者的不同之处： 
throttle 可以想象成阀门一样定时打开来调节流量。 debounce可以想象成把很多事件压缩组合成了一个事件。

有个生动的类比，假设你在乘电梯，每次进一个人需等待10秒钟，不考虑电梯容量限制，那么两种不同的策略是：

debounce 你在进入电梯后发现这时不远处走来了一个人，等10秒钟，这个人进电梯后不远处又有个妹纸姗姗来迟，怎么办，再等10秒，于是妹纸上电梯时又来了一对好基友...，作为感动中国好码农，你要每进一个人就等10秒，直到没有人进来，10秒超时，电梯开动。

throttle 电梯很忙，每次就只等10秒，不管是来了妹纸还是好基友，电梯每隔10秒准时送一波人。

因此，简单来说，debounce适合只执行一次的情况，例如 搜索框中的自动完成。在停止输入后提交一次ajax请求; 而throttle适合指定每隔一定时间间隔内执行不超过一次的情况，例如拖动滚动条，移动鼠标的事件处理等。

旁礴万物以为一。 ——《逍遥游》

##二者的相同在于：

首先我们的直观感觉是使用 debounce 方法相比于 throttle 方法事件触发的频率更低，但实际上不能这么理解。首先需要了解 debounce 和 throttle 的原理。

当我们阅读lodash，underscore等js工具库对这两种方法的代码实现时会发现，throttle 方法不过是 debounce 方法的一个修饰。也就是说，_.throttle和_.debounce最终都会都会调用 debounce 方法。那么 debounce 究竟是如何工作的呢？ 首先看 _.debounce 的 API： 
_.debounce(func, delay, options); 

func 是需要被调用的目标函数，delay 是时间限制，options 是一系列的配置，为了简化问题，这里不多提。

当调用 *_.debounce *后，会返回一个函数，这个函数在被调用时会生成一个 setTimeout(delayed, delay)。其中 delayed 又是一个内部方法，在 delayed 被调用时进行如下检测：当前时间 - 上次func被调用事件 是否 小于 0 或 大于 delay ？如果是则执行一次 func，记录并返回执行结果，同时更新上次被调用时间；如果不是则调用 setTimeout 进行下一次的判断。

因此，当你像下面这样绑定事件，

``` javascript
$(window).on('mousemove', _.debounce(moveHandler, 500));
```
并频繁移动鼠标时，你会发现 moveHandler 压根没有被调用过！直到你停下来后，moveHandler 才会被调用一次。

至于_.throttle方法，只不过是多给 debounce 传了一个 maxWait 选项，这个选项的意思是至少保证在每 maxWait 时间让 func 被调用一次。

说到这儿，我们就明白了为什么会出现上图那样的调用情况。如果你频繁的移动鼠标，throttle 会保证在每 maxWait 时间调用 func 一次，而 debounce 如果没有明确设置 maxWait，是一直不会调用 func 直到你停止移动鼠标后才会调用一次。
