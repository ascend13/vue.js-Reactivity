### 实现思路
首先需要知道vue实现双向绑定的原理是基于数据劫持的，通过Object.defineProperty()的getter和setter对属性的劫持，并结合订阅/发布模式，当数据变化时对订阅者发出通知，触发订阅者的更新方法来更新视图。

### 劫持子节点
因为我们最终的效果要和原生vue保持一致，vue的模板语法如下
```html
<div id="app">
    <input type="text" v-model="text">
    {{ text }}
</div>
```
通过DocumentFragment（文档片段）最终生成浏览器能够识别的html,可以将DocumentFragment看成一个容器，它可以包含多个子节点，当我们将它插入到 DOM 中时，只有它的子节点会插入目标节点，并且速度和性能也远远优于直接操作DOM，经过处理后，再将 DocumentFragment 整体返回插入挂载目标。

```javascript
function nodeToFragment (node, vm) {
        var flag = document.createDocumentFragment();
        var child;
        //appendChild会删除源节点，接着firstChild就指向了第二个节点，达到劫持所有子节点的目的,赋值语句也有返回值
        while (child = node.firstChild) {
             compile(child, vm);
            flag.appendChild(child); //劫持node的所有子节点
        }
        return flag;
    }
```
nodeToFragment函数接受挂载节点和vue实例为参数，当我们将app节点传入时，先生成一个DocumentFragment容器，while语句中，将app的第一个子节点传给child，赋值语句也有返回值，通过compile解析模板，并将其插入到flag时appendchild会删除源节点，因此while循环下一次就指向了第二个节点，进而达到劫持所有子节点的目的。

### 实现compile功能
在劫持到子节点后实现compile功能解析模板指令，并作初始化数据绑定

```javascript
function compile (node, vm) {
        var reg = /\{\{(.*)\}\}/;
        //节点类型为元素
        if (node.nodeType === 1) {
            var attr = node.attributes;
            //解析属性
            for (var i = 0; i < attr.length; i++) {
                if (attr[i].nodeName == 'v-model') {
                    var name = attr[i].nodeValue;//获取v-model绑定的属性名
                    node.addEventListener('input', function (e) {
                        //给相应的data属性赋值，进而触发该属性的set方法
                        vm[name] = e.target.value;
                    });                
                    node.value = vm[name];  //将data的值赋给该node
                    node.removeAttribute('v-model');
                }
            };
        }
        //节点类型为text
        if (node.nodeType === 3) {
            if (reg.test(node.nodeValue)) {
                var name = RegExp.$1;//获取匹配到的字符串
                name = name.trim();
                node.nodeValue = vm[name];  //将data的值赋给该node,触发属性的get方法
            }
        }
    }
```

### 响应式绑定的实现
接着定义defineReactive函数进行属性劫持
```javascript
function defineReactive (obj, key, val) {
        Object.defineProperty(obj, key, {
            get: function () {
                return val;
            },
            set: function (newVal) {
                if (newVal === val) return
                val = newVal;
            }
        });
    }

```
getter:在读取属性时调用的函数，返回值用作属性值
setter：在写入属性时调用的函数，该方法接受唯一参数，并将改参数的新值分配给该属性

实现observer数据监听器
```javascript
function observer (obj, vm) {
        Object.keys(obj).forEach(function (key) {
            defineReactive(vm, key, obj[key]);
        });
    }
```
>Object.keys() 方法会返回一个由一个给定对象的自身可枚举属性组成的数组，数组中属性名的排列顺序和使用 for...in 循环遍历该对象时返回的顺序一致 （两者的主要区别是 一个 for-in 循环还会枚举其原型链上的属性）。

Vue构造函数
```javascript
function Vue (options) {
            this.data = options.data;
            var data = this.data;
            observer(data, this);
            var id = options.el;
            var dom = nodeToFragment(document.getElementById(id), this);
            //编译完成后,将dom返回到app中
            document.getElementById(id).appendChild(dom);
        }
```
当我们new一个vue实例，实际上是调用vue构造函数，作为数据绑定的入口，进行初始化。

### 订阅发布模式
此时虽然已经做了响应式的数据绑定，但是当我们改变内容时，但是文本节点内容并未变化，setter只是将改变的新值赋给defineReactive全局变量val，这里通过订阅发布模式来实现。
>订阅发布模式又叫观察者模式，它定义了对象间的一种一对多的关系，让多个观察者对象同时监听某一个主题对象，当一个对象发生改变时，所有依赖于它的对象都将得到通知。

实现Wacher
```javascript
function Watcher (vm, node, name) {
        Dep.target = this;
        this.name = name;
        this.node = node;
        this.vm = vm;
        this.update();
        Dep.target = null;
    }

    Watcher.prototype = {
        update: function () {
            this.get();
            this.node.nodeValue = this.value;
        },

        //获取data中的属性值
        get: function () {
            this.value = this.vm[this.name];//触发相应属性的get
        }
    }
```
watcher订阅者作为连接observer和compile的桥梁,主要任务为：
+ 在自身实例化时往属性订阅器(dep)里面添加自己
+ 自身必须有一个update()方法,来更新视图
+ 待属性变动dep.notice()通知时，能调用自身的update()方法，并触发Compile中绑定的回调

将自己赋给了一个全局变量 Dep.target，其次，执行了 update 方法，进而执行了 get 方法，get 的方法读取了 vm 的访问器属性，从而触发了访问器属性的 get 方法，get 方法中将该 watcher 添加到了对应访问器属性的 dep 中； 最后，将 Dep.target 设为空。因为它是全局变量，也是 watcher 与 dep 关联的唯一桥梁，任何时刻都必须保证 Dep.target 只有一个值。

```javascript
function defineReactive (obj, key, val) {

    var dep = new Dep();

        Object.defineProperty(obj, key, {
            get: function () {
                //添加订阅者watcher到主题对象Dep
                if (Dep.target) {
                    dep.addSub(Dep.target);
                }
                return val;
            },
            set: function (newVal) {
                if (newVal === val) return
                val = newVal;
                //作为发布者发出通知
                dep.notify();
            }
        });
    }
```
我们是在input节点改变数据，text节点更新视图，因此在text节点添加订阅者，期望接受数据改变同通知。
```javascript
if (node.nodeType === 3) {
            if (reg.test(node.nodeValue)) {
                var name = RegExp.$1;//获取匹配到的字符串
                name = name.trim();
                node.nodeValue = vm[name];  //将data的值赋给该node,触发属性的get方法

                new Watcher(vm, node, name);
            }
        }
```
最后完成调度中心Dep，用来存储所有的订阅者并关联添加订阅者和通知（notify）功能
```javascript
function Dep () {
        this.subs = []
    }

    Dep.prototype = {
        addSub: function(sub) {
            this.subs.push(sub);
        },

        notify: function() {
            this.subs.forEach(function(sub) {
                sub.update();
            })
        }
    }
```
至此双向绑定已完成，完整代码[点我](https://www.baidu.com/)