# Proxy和Reflect 
## 1. Proxy概述 
Proxy用于修改某些操作的默认行为，等同于在语言层面做出修改，所以属于一种“元编程”（meta programming），即对编程语言进行编程。

Proxy可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。Proxy这个词的原意是代理，用在这里表示由它来“代理”某些操作，可以译为“代理器”。
```
var obj = new Proxy({}, {  
    get: function (target, key, receiver) {    
        console.log(`getting ${key}!`);    
        return Reflect.get(target, key, receiver);  
    },  
    set: function (target, key, value, receiver) { 
        console.log(`setting ${key}!`);    
        return Reflect.set(target, key, value, receiver);  
    } 
});
```
上面代码对一个空对象架设了一层拦截，重定义了属性的读取（get）和设置（set）行为。这里暂时先不解释具体的语法，只看运行结果。对设置了 拦截行为的对象obj，去读写它的属性，就会得到下面的结果。
```
obj.count = 1 
//  setting count! 
++obj.count 
//  getting count! 
//  setting count!
//  2
```
上面代码说明，Proxy实际上重载（overload）了点运算符，即用自己的定义覆盖了语言的原始定义。

ES6原生提供Proxy构造函数，用来生成Proxy实例。
```
var proxy = new Proxy(target, handler);
```
Proxy对象的所有用法，都是上面这种形式，不同的只是handler参数的写法。其中，new Proxy()表示生成一个Proxy实例，target参数表示所要拦截的 目标对象，handler参数也是一个对象，用来定制拦截行为。

下面是另一个拦截读取属性行为的例子。
```
var proxy = new Proxy({}, {  
    get: function(target, property) {    
        return 35;  
    } 
});

proxy.time // 35
proxy.name // 35 
proxy.title // 35
```
上面代码中，作为构造函数，Proxy接受两个参数。第一个参数是所要代理的目标对象（上例是一个空对象），即如果没有Proxy的介入，操作原来要 访问的就是这个对象；第二个参数是一个配置对象，对于每一个被代理的操作，需要提供一个对应的处理函数，该函数将拦截对应的操作。比如，上 面代码中，配置对象有一个get方法，用来拦截对目标对象属性的访问请求。get方法的两个参数分别是目标对象和所要访问的属性。可以看到，由于 拦截函数总是返回35，所以访问任何属性都得到35。

注意，要使得Proxy起作用，必须针对Proxy实例（上例是proxy对象）进行操作，而不是针对目标对象（上例是空对象）进行操作。

如果handler没有设置任何拦截，那就等同于直接通向原对象。
```
var target = {}; 
var handler = {}; 
var proxy = new Proxy(target, handler); 
proxy.a = 'b'; 
target.a // "b"
```
上面代码中，handler是一个空对象，没有任何拦截效果，访问handeler就等同于访问target。

一个技巧是将Proxy对象，设置到object.proxy属性，从而可以在object对象上调用。
```
var object = { proxy: new Proxy(target, handler) };
```
Proxy实例也可以作为其他对象的原型对象。
```
var proxy = new Proxy({}, {  
    get: function(target, property) {    
        return 35;  
    }   
});

let obj = Object.create(proxy); 
obj.time // 35
```
上面代码中，proxy对象是obj对象的原型，obj对象本身并没有time属性，所以根据原型链，会在proxy对象上读取该属性，导致被拦截。

同一个拦截器函数，可以设置拦截多个操作。
```
var handler = {  
    get: function(target, name) {    
        if (name === 'prototype') {      
            return Object.prototype;    
        }
        return 'Hello, ' + name;  
    },
    apply: function(target, thisBinding, args) {    
        return args[0];  
    },
    construct: function(target, args) {    
        return {value: args[1]};  
    } 
};

var fproxy = new Proxy(function(x, y) {  return x + y; }, handler);

fproxy(1, 2) // 1 
new fproxy(1,2) // {value: 2} 
fproxy.prototype === Object.prototype // true 
fproxy.foo // "Hello, foo"
```
下面是Proxy支持的拦截操作一览。

对于可以设置、但没有设置拦截的操作，则直接落在目标对象上，按照原先的方式产生结果。

（1）get(target, propKey, receiver)拦截对象属性的读取，比如proxy.foo和proxy['foo']。
最后一个参数receiver是一个对象，可选，参见下面Reflect.get的部分。

（2）set(target, propKey, value, receiver)拦截对象属性的设置，比如proxy.foo = v或proxy['foo'] = v，返回一个布尔值。

（3）has(target, propKey)
拦截propKey in proxy的操作，以及对象的hasOwnProperty方法，返回一个布尔值。

（4）deleteProperty(target, propKey)
拦截delete proxy[propKey]的操作，返回一个布尔值。

（5）ownKeys(target)
拦截Object.getOwnPropertyNames(proxy)、Object.getOwnPropertySymbols(proxy)、Object.keys(proxy)，返回一个数组。该方法返回对象所有自身的属性，而Object.keys()仅返回对象可遍历的属性。

（6）getOwnPropertyDescriptor(target, propKey)
拦截Object.getOwnPropertyDescriptor(proxy, propKey)，返回属性的描述对象。

（7）defineProperty(target, propKey, propDesc)
拦截Object.defineProperty(proxy, propKey, propDesc）、Object.defineProperties(proxy, propDescs)，返回一个布尔值。

（8）preventExtensions(target)
拦截Object.preventExtensions(proxy)，返回一个布尔值。

（9）getPrototypeOf(target)
拦截Object.getPrototypeOf(proxy)，返回一个对象。

（10）isExtensible(target)
拦截Object.isExtensible(proxy)，返回一个布尔值。

（11）setPrototypeOf(target, proto)
拦截Object.setPrototypeOf(proxy, proto)，返回一个布尔值。

如果目标对象是函数，那么还有两种额外操作可以拦截。

（12）apply(target, object, args)
拦截Proxy实例作为函数调用的操作，比如proxy(...args)、proxy.call(object, ...args)、proxy.apply(...)。

（13）construct(target, args)
拦截Proxy实例作为构造函数调用的操作，比如new proxy(...args)。 

## 2. Proxy实例的方法 
下面是上面这些拦截方法的详细介绍。
### **2.1 get()**
get方法用于拦截某个属性的读取操作。上文已经有一个例子，下面是另一个拦截读取操作的例子。
```
var person = {  
    name: "张三" 
};

var proxy = new Proxy(person, {  
    get: function(target, property) {    
        if (property in target) {      
            return target[property];    
        } else {      
            throw new ReferenceError("Property \"" + property + "\" does not exist.");    
        }  
    } 
});

proxy.name // "张三" 
proxy.age // 抛出一个错误
```
上面代码表示，如果访问目标对象不存在的属性，会抛出一个错误。如果没有这个拦截函数，访问不存在的属性，只会返回undefined。

get方法可以继承。
```
let proto = new Proxy({}, {  
    get(target, propertyKey, receiver) {    
        console.log('GET '+propertyKey);    
        return target[propertyKey];  
    } 
});

let obj = Object.create(proto); 
obj.xxx // "GET xxx"
```
上面代码中，拦截操作定义在Prototype对象上面，所以如果读取obj对象继承的属性时，拦截会生效。

下面的例子使用get拦截，实现数组读取负数的索引。
```
function createArray(...elements) { 
    let handler = {    
        get(target, propKey, receiver) {      
            let index = Number(propKey);      
            if (index < 0) {        
                propKey = String(target.length + index);      
            }      
            return Reflect.get(target, propKey, receiver);
        }  
    };
    let target = [];  
    target.push(...elements);  
    return new Proxy(target, handler); 
}

let arr = createArray('a', 'b', 'c'); 
arr[-1] // c
```
上面代码中，数组的位置参数是-1，就会输出数组的倒数最后一个成员。

利用Proxy，可以将读取属性的操作（get），转变为执行某个函数，从而实现属性的链式操作。
```
var pipe = (function () {  
    return function (value) {    
        var funcStack = [];    
        var oproxy = new Proxy({} , {      
            get : function (pipeObject, fnName) {        
                if (fnName === 'get') {          
                    return funcStack.reduce(function (val, fn){                
                        return fn(val);          
                    },value);        
                }        
                funcStack.push(window[fnName]);        
                return oproxy;      
            }    
        });
        return oproxy;  
    } 
}());

var double = n => n * 2; 
var pow    = n => n * n; 
var reverseInt = n => n.toString().split("").reverse().join("") | 0;

pipe(3).double.pow.reverseInt.get; // 63
```
上面代码设置Proxy以后，达到了将函数名链式使用的效果。

下面的例子则是利用get拦截，实现一个生成各种DOM节点的通用函数dom。
```
const dom = new Proxy({}, {  
    get(target, property) {    
        return function(attrs = {}, ...children) {      
            const el = document.createElement(property);      
            for (let prop of Object.keys(attrs)) {        
                el.setAttribute(prop, attrs[prop]);      
            }      
            for (let child of children) {        
                if (typeof child === 'string') {              
                    child = document.createTextNode(child);        
                }        
                el.appendChild(child);      
            }      
            return el;    
        }  
    } 
});

const el = dom.div({},  
    'Hello, my name is ',  
    dom.a({href: '//example.com'}, 'Mark'),  '. I like:',
    dom.ul({},    
        dom.li({}, 'The web'),    
        dom.li({}, 'Food'),    
        dom.li({}, '…actually that\'s it')  
    ) 
);

document.body.appendChild(el);
```
### **2.2 set()**
set方法用来拦截某个属性的赋值操作。

假定Person对象有一个age属性，该属性应该是一个不大于200的整数，那么可以使用Proxy保证age的属性值符合要求。
```
let validator = {  
    set: function(obj, prop, value) {    
        if (prop === 'age') {      
            if (!Number.isInteger(value)) {        
                throw new TypeError('The age is not an integer');      
            }      
            if (value > 200) {        
                throw new RangeError('The age seems invalid');      
            }    
        }
        // 对于age以外的属性，直接保存    
        obj[prop] = value;  
    } 
};

let person = new Proxy({}, validator);

person.age = 100;
person.age // 100 
person.age = 'young' // 报错 
person.age = 300 // 报错
```
上面代码中，由于设置了存值函数set，任何不符合要求的age属性赋值，都会抛出一个错误。利用set方法，还可以数据绑定，即每当对象发生变化 时，会自动更新DOM。

有时，我们会在对象上面设置内部属性，属性名的第一个字符使用下划线开头，表示这些属性不应该被外部使用。结合get和set方法，就可以做到防止这些内部属性被外部读写。
```
var handler = {  
    get (target, key) {    
        invariant(key, 'get');    
        return target[key];  
    },  
    set (target, key, value) {    
        invariant(key, 'set');    
        return true;  } 
}; 
function invariant (key, action) {  
    if (key[0] === '_') {    
        throw new Error(`Invalid attempt to ${action} private "${key}" property`);  
    } 
} 
var target = {}; 
var proxy = new Proxy(target, handler); 
proxy._prop // Error: Invalid attempt to get private "_prop"
property proxy._prop = 'c' // Error: Invalid attempt to set private "_prop" property
```
上面代码中，只要读写的属性名的第一个字符是下划线，一律抛错，从而达到禁止读写内部属性的目的。
### **2.3 apply()**
apply方法拦截函数的调用、call和apply操作。
```
var handler = {  
    apply (target, ctx, args) {    
        return Reflect.apply(...arguments);  
    } 
};
```
apply方法可以接受三个参数，分别是目标对象、目标对象的上下文对象（this）和目标对象的参数数组。

下面是一个例子。
```
var target = function () { return 'I am the target'; }; 
var handler = {  
    apply: function () {    
        return 'I am the proxy';  
    } 
};

var p = new Proxy(target, handler);
p()  // "I am the proxy"
```
上面代码中，变量p是Proxy的实例，当它作为函数调用时（p()），就会被apply方法拦截，返回一个字符串。

下面是另外一个例子。
```
var twice = {  
    apply (target, ctx, args) {    
        return Reflect.apply(...arguments) * 2;  
    } 
}; 
function sum (left, right) {  
    return left + right; 
}; 
var proxy = new Proxy(sum, twice); 
proxy(1, 2) // 6 
proxy.call(null, 5, 6) // 22 
proxy.apply(null, [7, 8]) // 30
```
上面代码中，每当执行proxy函数（直接调用或call和apply调用），就会被apply方法拦截。

另外，直接调用Reflect.apply方法，也会被拦截。
```
Reflect.apply(proxy, null, [9, 10]) // 38
```
### **2.4 has()**
has方法用来拦截HasProperty操作，即判断对象是否具有某个属性时，这个方法会生效。典型的操作就是in运算符。

下面的例子使用has方法隐藏某些属性，不被in运算符发现。
```
var handler = {  
    has (target, key) {    
        if (key[0] === '_') {      
            return false;    
        }    
        return key in target;
    } 
}; 
var target = { _prop: 'foo', prop: 'foo' }; 
var proxy = new Proxy(target, handler); 
'_prop' in proxy  // false
```
上面代码中，如果原对象的属性名的第一个字符是下划线，proxy.has就会返回false，从而不会被in运算符发现。

如果原对象不可配置或者禁止扩展，这时has拦截会报错。
```
var obj = { a: 10 }; 
Object.preventExtensions(obj); 
var p = new Proxy(obj, {  
    has: function(target, prop) {    
        return false;  
    } 
});

'a' in p // TypeError is thrown
```
上面代码中，obj对象禁止扩展，结果使用has拦截就会报错。

值得注意的是，has方法拦截的是HasProperty操作，而不是HasOwnProperty操作，即has方法不判断一个属性是对象自身的属性，还是继承的属性。 由于for...in操作内部也会用到HasProperty操作，所以has方法在for...in循环时也会生效。
```
let stu1 = {name: 'Owen', score: 59}; 
let stu2 = {name: 'Mark', score: 99};
let handler = {  
    has(target, prop) {    
        if (prop === 'score' && target[prop] < 60) {
            console.log(`${target.name} 不及格`);      
            return false;    
        }    
        return prop in target;  
    } 
}

let oproxy1 = new Proxy(stu1, handler); 
let oproxy2 = new Proxy(stu2, handler);
for (let a in oproxy1) {  
    console.log(oproxy1[a]); 
} 
// Owen 
// Owen 不及格

for (let b in oproxy2) {  
    console.log(oproxy2[b]); 
} 
// Mark 
// Mark 99
```
上面代码中，for...in循环时，has拦截会生效，导致不符合要求的属性被排除在for...in循环之外。
### **2.5 construct()**
construct方法用于拦截new命令，下面是拦截对象的写法。
```
var handler = {  
    construct (target, args, newTarget) {    
        return new target(...args);  
    } 
};
```
下面是一个例子。
```
var p = new Proxy(function() {}, {  
    construct: function(target, args) {    
        console.log('called: ' + args.join(', '));    
        return { value: args[0] * 10 };  
    } 
});

new p(1).value 
// "called: 1" 
// 10
```
construct方法返回的必须是一个对象，否则会报错。
```
var p = new Proxy(function() {}, {  
    construct: function(target, argumentsList) {    
        return 1;  
    } 
});
new p() // 报错
```
### **2.6 deleteProperty()**
deleteProperty方法用于拦截delete操作，如果这个方法抛出错误或者返回false，当前属性就无法被delete命令删除。
```
var handler = {  
    deleteProperty (target, key) {    
        invariant(key, 'delete');    
        return true;  
    } 
}; 
function invariant (key, action) {  
    if (key[0] === '_') {    
        throw new Error(`Invalid attempt to ${action} private "${key}" property`);  
    } 
}

var target = { _prop: 'foo' }; 
var proxy = new Proxy(target, handler); 
delete proxy._prop 
// Error: Invalid attempt to delete private "_prop" property
```
上面代码中，deleteProperty方法拦截了delete操作符，删除第一个字符为下划线的属性会报错。
### **2.7 defineProperty()**
defineProperty方法拦截了Object.defineProperty操作。
```
var handler = {  
    defineProperty (target, key, descriptor) {    
        return false;  
    } 
}; 
var target = {}; 
var proxy = new Proxy(target, handler); 
proxy.foo = 'bar' 
// TypeError: proxy defineProperty handler returned false for property '"foo"'
```
上面代码中，defineProperty方法返回false，导致添加新属性会抛出错误。
### **2.8 getOwnPropertyDescriptor()**
getOwnPropertyDescriptor方法拦截Object.getOwnPropertyDescriptor，返回一个属性描述对象或者undefined。
```
var handler = {  
    getOwnPropertyDescriptor (target, key) {    
        if (key[0] === '_') {      
            return;    
        }    
        return Object.getOwnPropertyDescriptor(target, key);  
    } 
}; 
var target = { _foo: 'bar', baz: 'tar' }; 
var proxy = new Proxy(target, handler); Object.getOwnPropertyDescriptor(proxy, 'wat') // undefined Object.getOwnPropertyDescriptor(proxy, '_foo') // undefined Object.getOwnPropertyDescriptor(proxy, 'baz') 
// { value: 'tar', writable: true, enumerable: true, configurable: true }
```
上面代码中，handler.getOwnPropertyDescriptor方法对于第一个字符为下划线的属性名会返回undefined。
### **2.9 getPrototypeOf()**
getPrototypeOf方法主要用来拦截Object.getPrototypeOf()运算符，以及其他一些操作。
* Object.prototype.\__proto__ 
* Object.prototype.isPrototypeOf() 
* Object.getPrototypeOf() 
* Reflect.getPrototypeOf() 
* instanceof运算符

下面是一个例子。
```
var proto = {}; 
var p = new Proxy({}, {  
    getPrototypeOf(target) {    
        return proto;  
    } 
}); 
Object.getPrototypeOf(p) === proto // true
```
上面代码中，getPrototypeOf方法拦截Object.getPrototypeOf()，返回proto对象。
### **2.10 isExtensible()**
isExtensible方法拦截Object.isExtensible操作。
```
var p = new Proxy({}, {  
    isExtensible: function(target) {    
        console.log("called");    
        return true;  
    } 
});

Object.isExtensible(p) 
// "called" 
// true
```
上面代码设置了isExtensible方法，在调用Object.isExtensible时会输出called。

这个方法有一个强限制，如果不能满足下面的条件，就会抛出错误。
```
Object.isExtensible(proxy) === Object.isExtensible(target)
```
下面是一个例子。
```
var p = new Proxy({}, {  
    isExtensible: function(target) {    
        return false;  
    } 
});

Object.isExtensible(p) // 报错
```
### **2.11  ownKeys()**
ownKeys方法用来拦截Object.keys()操作。
```
let target = {};
let handler = {  
    ownKeys(target) {    
        return ['hello', 'world'];  
    } 
};
let proxy = new Proxy(target, handler);
Object.keys(proxy) 
// [ 'hello', 'world' ]
```
上面代码拦截了对于target对象的Object.keys()操作，返回预先设定的数组。

下面的例子是拦截第一个字符为下划线的属性名。
```
let target = {  
    _bar: 'foo',  
    _prop: 'bar',  
    prop: 'baz' 
};

let handler = {  
    ownKeys (target) {    
        return Reflect.ownKeys(target).filter(key => key[0] !== '_');  
    } 
};

let proxy = new Proxy(target, handler); 
for (let key of Object.keys(proxy)) {  
    console.log(target[key]); 
} 
// "baz"
```
### **2.12 preventExtensions()**
preventExtensions方法拦截Object.preventExtensions()。该方法必须返回一个布尔值。

这个方法有一个限制，只有当Object.isExtensible(proxy)为false（即不可扩展）时，proxy.preventExtensions才能返回true，否则会报错。
```
var p = new Proxy({}, {  
    preventExtensions: function(target) {    
        return true;  
    } 
});

Object.preventExtensions(p) // 报错
```
上面代码中，proxy.preventExtensions方法返回true，但这时Object.isExtensible(proxy)会返回true，因此报错。

为了防止出现这个问题，通常要在proxy.preventExtensions方法里面，调用一次Object.preventExtensions。
```
var p = new Proxy({}, {  
    preventExtensions: function(target) {    
        console.log("called");    
        Object.preventExtensions(target);    
        return true;  
    } 
});

Object.preventExtensions(p) 
// "called" 
// true
```
### **2.13 setPrototypeOf()**
setPrototypeOf方法主要用来拦截Object.setPrototypeOf方法。

下面是一个例子。
```
var handler = {  
    setPrototypeOf (target, proto) {    
        throw new Error('Changing the prototype is forbidden');
    } 
}; 
var proto = {}; 
var target = function () {}; 
var proxy = new Proxy(target, handler); 
proxy.setPrototypeOf(proxy, proto); 
// Error: Changing the prototype is forbidden
```
上面代码中，只要修改target的原型对象，就会报错。 

## 3. Proxy.revocable()
Proxy.revocable方法返回一个可取消的Proxy实例。
```
let target = {}; 
let handler = {};

let {proxy, revoke} = Proxy.revocable(target, handler);

proxy.foo = 123; 
proxy.foo // 123

revoke(); 
proxy.foo // TypeError: Revoked
```
Proxy.revocable方法返回一个对象，该对象的proxy属性是Proxy实例，revoke属性是一个函数，可以取消Proxy实例。上面代码中，当执行revoke函 数之后，再访问Proxy实例，就会抛出一个错误。
## 4. Reflect概述
Reflect对象与Proxy对象一样，也是ES6为了操作对象而提供的新API。Reflect对象的设计目的有这样几个。

（1）将Object对象的一些明显属于语言内部的方法（比如Object.defineProperty），放到Reflect对象上。现阶段，某些方法同时 在Object和Reflect对象上部署，未来的新方法将只部署在Reflect对象上。

（2）修改某些Object方法的返回结果，让其变得更合理。比如，Object.defineProperty(obj, name, desc)在无法定义属性时，会抛出一个错误， 而Reflect.defineProperty(obj, name, desc)则会返回false。
```
// 老写法 
try {  
    Object.defineProperty(target, property, attributes);  
    // success 
} catch (e) {  
    // failure 
}

// 新写法 
if (Reflect.defineProperty(target, property, attributes)) {
    // success 
} else {  
    // failure 
}
```
（3）让Object操作都变成函数行为。某些Object操作是命令式，比如name in obj和delete obj[name]， 而Reflect.has(obj, name)和Reflect.deleteProperty(obj, name)让它们变成了函数行为。
```
// 老写法 
'assign' in Object // true

// 新写法 
Reflect.has(Object, 'assign') // true
```
（4）Reflect对象的方法与Proxy对象的方法一一对应，只要是Proxy对象的方法，就能在Reflect对象上找到对应的方法。这就让Proxy对象可以方便地调用对应的Reflect方法，完成默认行为，作为修改行为的基础。也就是说，不管Proxy怎么修改默认行为，你总可以在Reflect上获取默认行为。
```
Proxy(target, {  
    set: function(target, name, value, receiver) {    
        var success = Reflect.set(target,name, value, receiver);    
        if (success) {      
            log('property ' + name + ' on ' + target + ' set to ' + value);    
        }    
        return success;  
    } 
});
```
上面代码中，Proxy方法拦截target对象的属性赋值行为。它采用Reflect.set方法将值赋值给对象的属性，然后再部署额外的功能。

下面是另一个例子。
```
var loggedObj = new Proxy(obj, {  
    get(target, name) {    
        console.log('get', target, name);    
        return Reflect.get(target, name);  
    },  
    deleteProperty(target, name) {    
        console.log('delete' + name);    
        return Reflect.deleteProperty(target, name);  
    },  
    has(target, name) {    
        console.log('has' + name);    
        return Reflect.has(target, name);  
    } 
});
```
上面代码中，每一个Proxy对象的拦截操作（get、delete、has），内部都调用对应的Reflect方法，保证原生行为能够正常执行。添加的工作，就是 将每一个操作输出一行日志。

有了Reflect对象以后，很多操作会更易读。
```
// 老写法 
Function.prototype.apply.call(Math.floor, undefined, [1.75]) // 1
// 新写法 
Reflect.apply(Math.floor, undefined, [1.75]) // 1
```
## 5.Reflect对象的方法 
Reflect对象的方法清单如下，共13个。
* Reflect.apply(target,thisArg,args) 
* Reflect.construct(target,args) 
* Reflect.get(target,name,receiver) 
* Reflect.set(target,name,value,receiver) 
* Reflect.defineProperty(target,name,desc) 
* Reflect.deleteProperty(target,name) 
* Reflect.has(target,name) 
* Reflect.ownKeys(target) 
* Reflect.isExtensible(target) 
* Reflect.preventExtensions(target)
* Reflect.getOwnPropertyDescriptor(target, name) 
* Reflect.getPrototypeOf(target) 
* Reflect.setPrototypeOf(target, prototype)

上面这些方法的作用，大部分与Object对象的同名方法的作用都是相同的，而且它与Proxy对象的方法是一一对应的。下面是对其中几个方法的解释。

（1）Reflect.get(target, name, receiver)

查找并返回target对象的name属性，如果没有该属性，则返回undefined。

如果name属性部署了读取函数，则读取函数的this绑定receiver。
```
var obj = {  
    get foo() { return this.bar(); },  
    bar: function() { ... } 
};

// 下面语句会让 this.bar() 
// 变成调用 wrapper.bar() 
Reflect.get(obj, "foo", wrapper);
```
（2）Reflect.set(target, name, value, receiver)

设置target对象的name属性等于value。如果name属性设置了赋值函数，则赋值函数的this绑定receiver。

（3）Reflect.has(obj, name)

等同于name in obj。

（4）Reflect.deleteProperty(obj, name)

等同于delete obj[name]。

（5）Reflect.construct(target, args)

等同于new target(...args)，这提供了一种不使用new，来调用构造函数的方法。

（6）Reflect.getPrototypeOf(obj)

读取对象的__proto__属性，对应Object.getPrototypeOf(obj)。

（7）Reflect.setPrototypeOf(obj, newProto)

设置对象的__proto__属性，对应Object.setPrototypeOf(obj, newProto)。

（8）Reflect.apply(fun,thisArg,args)

等同于Function.prototype.apply.call(fun,thisArg,args)。一般来说，如果要绑定一个函数的this对象，可以这样写fn.apply(obj, args)，但是如果函数定义了自己的apply方法，就只能写成Function.prototype.apply.call(fn, obj, args)，采用Reflect对象可以简化这种操作。

另外，需要注意的是，Reflect.set()、Reflect.defineProperty()、Reflect.freeze()、Reflect.seal()和Reflect.preventExtensions()返回一个布尔值，表示操作是否成功。它们对应的Object方法，失败时都会抛出错误。
```
// 失败时抛出错误 
Object.defineProperty(obj, name, desc); 
// 失败时返回false 
Reflect.defineProperty(obj, name, desc);
```
上面代码中，Reflect.defineProperty方法的作用与Object.defineProperty是一样的，都是为对象定义一个属性。但是，Reflect.defineProperty方法失败时，不会抛出错误，只会返回false。 


