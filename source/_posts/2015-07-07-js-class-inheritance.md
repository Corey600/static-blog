---
layout: post
title: JS的类和继承的实现方法比较
category : 备忘
tagline: "备忘"
tags : [javascript,类,继承]
excerpt_separator: <!--more-->
---

#### 1.创建对象的各种方式对比

##### 工厂模式

```
(function(){
    console.info("工厂模式");
    // 工厂模式
 
    function createPerson(name, age, job){
        var o = new Object();
        o.name = name;
        o.age = age;
        o.job = job;
        o.sayName = function(){
            console.log(this.name);
        };
        return o;
    }
 
    var person1 = createPerson("name1", 29, "job1");
    person1.sayName();
    var person2 = createPerson("name2", 30, "job2");
    person2.sayName();
 
    /**
     * 优点：能创建相似对象。
     * 缺点：不能识别对象。
     */
}());
```

<!--more-->

##### 构造函数模式

```
(function(){
    console.info("构造函数模式");
    // 构造函数模式
 
    function Person(name, age, job){
        this.name = name;
        this.age = age;
        this.job = job;
        this.sayName = function(){
            console.log(this.name);
        };
    }
 
    var person1 = new Person("name1", 29, "job1");
    person1.sayName();
    var person2 = new Person("name2", 30, "job2");
    person2.sayName();
 
    console.log(person1.constructor == Person); // true
    console.log(person1.constructor == Person); // true
 
    console.log(person1 instanceof Person); // true
    console.log(person1 instanceof Object); // true
    console.log(person2 instanceof Person); // true
    console.log(person2 instanceof Object); // true
 
    /**
     * 优点：创建自定义的构造函数意味着可以将它的实例标识为一种特定的类型。
     * 缺点：每个方法都要在每个实例上重新创建一遍。
     */
}());
```

##### 另一种函数构造模式

```
(function(){
    console.info("另一种函数构造模式");
    // 另一种函数构造模式
 
    function Person(name, age, job){
        this.name = name;
        this.age = age;
        this.job = job;
        this.sayName = sayName;
    }
 
    function sayName(){
        console.log(this.name);
    }
 
    var person1 = new Person("name1", 29, "job1");
    person1.sayName();
    var person2 = new Person("name2", 30, "job2");
    person2.sayName();
 
    /**
     * 优点：不同对象能够共享在全局作用域中定义的同一个函数。
     * 缺点：
     * (1) 在全局作用域中定义的函数实际上只能被某个对象调用，这让全局作用域名不副实。
     * (2) 如果对象定义很多方法，那么就要定义很多个全局函数，于是自定义的引用类型就失去了封
     * 装性。
     */
}());
```

##### 原型模式

``` 
(function(){
    console.info("原型模式");
    // 原型模式
 
    function Person(){
    }
 
    Person.prototype.name = "name1";
    Person.prototype.age = 29;
    Person.prototype.job = "job1";
    Person.prototype.friends = ["f1", "f2"];
    Person.prototype.sayName = function(){
        console.log(this.name);
    };
 
    var person1 = new Person();
    person1.sayName(); // name1
    var person2 = new Person();
    person2.sayName(); // name1
 
    console.log(person1.sayName == person2.sayName); // true
 
    // 确定是否是原型中的属性
    function isPrototypeProperty(object, name){
        return (!object.hasOwnProperty(name)) && (name in object);
    }
    console.log(isPrototypeProperty(person1, "name")); // true
 
    person1.name = "name2";
    console.log(person1.name); // name2 —— 来自实例
    console.log(person2.name); // name1 —— 来自原型
    console.log(isPrototypeProperty(person1, "name")); // false
 
    delete person1.name;
    console.log(person1.name); // name1 —— 来自原型
 
    console.log(person1.friends); // ["f1", "f2"]
    console.log(person2.friends); // ["f1", "f2"]
    person1.friends.push("f3");
    console.log(person1.friends); // ["f1", "f2", "f3"]
    console.log(person2.friends); // ["f1", "f2", "f3"]
    console.log(person1.friends == person2.friends); // true
 
    /**
     * 优点：所有对象实例能够共享原型的属性和方法。
     * 缺点：
     * (1) 不能通过对象实例重写原型中的值，如果在实例中添加一个同名属性，就是在实例中创建了
     * 该属性，屏蔽了原型中的那个属性。删除该同名属性可以恢复对原型中属性的链接。
     * (2) 所有实例在默认情况下都将取得相同的属性值。
     * (3) 原型中的所有属性都是被实例共享的。这个问题对于包含引用类型值的属性来说比较突出。
     */
}());
```

##### 更简单的原型语法

```
(function(){
    console.info("更简单的原型语法");
    // 更简单的原型语法
 
    function Person(){
    }
 
    var prePerson = new Person();
 
    // 重写原型对象
    Person.prototype = {
        name : "name1",
        age : 29,
        job : "job1",
        sayName : function(){
            console.log(this.name);
        }
    };
 
    // prePerson.sayName(); // 出错
 
    var person = new Person();
 
    console.log(person instanceof Object); // true
    console.log(person instanceof Person); // true
    console.log(person.constructor == Object); // true
    console.log(person.constructor == Person); // false
 
    // 重新设置constructor的值
    Person.prototype.constructor = Person;
    console.log(person.constructor == Person); // true
 
    // 以下代码只适用于ECMAScript5 兼容的浏览器
    console.log(Object.keys(Person.prototype)); // ["name", "age", "job", "sayName", "constructor"]
    Object.defineProperty(Person.prototype, "constructor", {
        enumerable : false,
        value : Person
    });
    console.log(Object.keys(Person.prototype)); // ["name", "age", "job", "sayName"]
 
    /**
     * 优点：相比于之前的原型语法，减少了不必要的输入，从视觉上更好地封装了原型的功能。
     * 缺点：
     * (1) 具有之前的原型模式的全部三个缺点。
     * (2) 本质上重写了默认的prototype对象，因此constructor属性也变成了新的对象的
     * constructor属性(指向Object构造函数)， 不再指向Person函数。当然也可以重新设置
     * constructor值，但是这样会导致它的[[Enumerable]]特性被设置为true，单是原生的
     * constructor属性是不可枚举的。此时就需要使用ECMAScript5的特性来重设。
     * (3) 重写原型对象切断了现有原型和任何之前已经存在的对象实例之间的联系，在重写原型
     * 之前创建的对象实例引用的仍然是最初的原型（原型的动态性）。
     */
}());
```

##### 组合使用构造函数和原型模式

```
(function(){
    console.info("组合使用构造函数和原型模式");
    // 组合使用构造函数和原型模式
 
    function Person(name, age, job){
        this.name = name;
        this.age = age;
        this.job = job;
        this.friends = ["f1", "f2"];
    }
 
    Person.prototype = {
        constructor : Person,
        sayName : function(){
            console.log(this.name);
        }
    };
 
    var person1 = new Person("name1", 29, "job1");
    person1.sayName();
    var person2 = new Person("name2", 30, "job2");
    person2.sayName();
 
    console.log(person1.friends); // ["f1", "f2"]
    console.log(person2.friends); // ["f1", "f2"]
    person1.friends.push("f3");
    console.log(person1.friends); // ["f1", "f2", "f3"]
    console.log(person2.friends); // ["f1", "f2"]
    console.log(person1.friends == person2.friends); // false
 
    /**
     * 优点：实例属性在构造函数中定义，所有实例共享的属性和方法在原型中定义。
     * 缺点：独立的构造函数和原型使人困惑。
     */
}());
```

##### 动态原型模式 （推荐方式）

```
(function(){
    console.info("动态原型模式");
    // 动态原型模式 （推荐方式）
 
    function Person(name, age, job){
 
        // 属性
        this.name = name;
        this.age = age;
        this.job = job;
 
        // 方法
        if(typeof this.sayName != "function"){
            // 不能使用字面量重写原型，否则会切断现有实例与新原型之间的联系
            Person.prototype.sayName = function(){
                console.log(this.name);
            };
        }
    }
 
    var person = new Person("name1", 29, "job1");
    person.sayName();
 
    /**
     * 优点：
     * (1) 把所有信息都封装在了构造函数中.
     * (2) 只在有必要时初始化原型。
     * (3) 对原型所做的修改能够立即在所有实例中得到反映。
     * (4) 实例属性在构造函数中定义，所有实例共享的属性和方法在原型中定义。
     * 缺点：不能使用字面量重写原型。
     */
}());
```

##### 寄生构造函数模式

``` 
(function(){
    console.info("寄生构造函数模式");
    // 寄生构造函数模式
 
    function MyArray(){
        // 创建数组
        var values = new Array();
 
        // 添加值
        values.push.apply(values,  arguments);
 
        // 添加方法
        values.toPipedString = function(){
            return this.join("|");
        };
 
        // 返回数组
        return values;
    }
 
    var colors = new MyArray("col1", "col2", "col3");
    console.log(colors.toPipedString());
 
    /**
     * 说明：《javascript高级程序设计第三版》 p160
     * >除了使用new操作符并把使用的包装函数叫做构造函数之外，这个模式跟工厂模式其实已一模一
     * 样的。构造函数在不返回值的情况下，默认返回新的对象实例。而通过在构造函数的末尾添加
     * 一个return语句，重写了调用构造函数时的返回值。
     */
 
    /**
     * 优点：能在不修改原生类型的情况下，创建具有额外方法的扩展原生类型的对象。
     * 缺点：返回的对象与构造函数或者与构造函数的原型属性之间没有关系，不能依赖instanceof
     * 操作符来确定对象类型。
     */
}());
```

##### 稳妥构造函数模式

```
(function(){
    console.info("稳妥构造函数模式");
    // 稳妥构造函数模式
 
    function Person(name, age, job){
        // 创建要返回的对象
        var o = new Object();
 
        // 可以在这里定义私有变量和函数
 
        // 添加方法
        o.sayName = function(){
            console.log(name);
        };
 
        // 返回对象
        return o;
    }
 
    var person = Person("name1", 29, "job1");
    person.sayName();
 
    /**
     * 说明：《javascript高级程序设计第三版》 p161
     * >所谓的稳妥对象，指的是没有公共属性，而且其方法也不引用this的对象。稳妥对象最适合在
     * 一些安全的环境中(这些环境中会禁用this和new)，或者在防止数据被其他应用程序(如Mashup
     * 程序)改动时使用。稳妥构造函数遵循与寄生构造函数类似的模式，但有两点不同：一是新创建
     * 对象的实例方法不引用this；二是不使用new操作符调用构造函数。
     */
 
    /**
     * 优点：除了调用方法外，没有别的方式可以访问其数据成员。
     * 缺点：返回的对象与构造函数或者与构造函数的原型属性之间没有关系，不能依赖instanceof
     * 操作符来确定对象类型。
     */
}());
```

#### 2. 实现继承的各种方式对比

##### 原型链

``` 
(function(){
    console.info("原型链");
    // 原型链
 
    function SuperType(){
        this.property = true;
        this.colors = ["col1", "col2"];
    }
    SuperType.prototype.getSuperValue = function(){
        return this.property;
    };
 
    function SubType(){
        this.subproperty = true;
    }
    SubType.prototype = new SuperType();
    SubType.prototype.getSubValue = function(){
        return this.subproperty;
    };
 
    var instance1 = new SubType();
    console.log(instance1.getSuperValue());
 
    console.log(instance1 instanceof Object); // true
    console.log(instance1 instanceof SuperType); // true
    console.log(instance1 instanceof SubType); // true
 
    console.log(Object.prototype.isPrototypeOf(instance1)); // true
    console.log(SuperType.prototype.isPrototypeOf(instance1)); // true
    console.log(SubType.prototype.isPrototypeOf(instance1)); // true
 
    var instance2 = new SubType();
    instance1.colors.push("col3");
    console.log(instance1.colors); // ["col1", "col2", "col3"]
    console.log(instance2.colors); // ["col1", "col2", "col3"]
 
    /**
     * 优点：使用instanceof操作符来测试实例与原型链中出现的构造函数，都返回true。同样，只
     * 要是原型链中出现过的原型，都可以说是该原型链所派生的实例的原型，因此isPrototypeOf()
     * 方法也会返回true.
     * 缺点：
     * (1) 不能使用对象字面量创建原型方法，因为这样会重写原型链。
     * (2) 子类的原型实际上是超类的实例，超类中包含引用类型值的原型属性会被所有实例共享。
     * (3) 不能在不影响所有实例的情况下给超类的构造函数传递参数。
     */
}());
```

##### 借用构造函数

```
(function(){
    console.info("借用构造函数");
    // 借用构造函数
 
    function SuperType(name){
        this.name = name;
    }
 
    function SubType(){
        // 继承了 SuperType，同时还传递了参数
        SuperType.call(this, "name1");
 
        // 实例属性
        this.age = 29;
    }
 
    var instance = new SubType();
    console.log(instance.name);
    console.log(instance.age);
 
    /**
     * 优点：可以在子类型构造函数中向超类型构造函数传递参数。
     * 缺点：方法只能在构造函数中定义，在超类原型中定义的方法对于子类型是不可见的。
     */
}());
```

##### 组合继承 （最常用）

``` 
(function(){
    console.info("组合继承");
    // 组合继承 （最常用）
 
    function SuperType(name){
        this.name = name;
        this.colors = ["col1", "col2"];
    }
    SuperType.prototype.sayName = function(){
        console.log(this.name);
 
    };
 
    function SubType(name, age){
        // 继承属性
        SuperType.call(this, name);
 
        this.age = age;
    }
 
    // 继承方法
    SubType.prototype = new SuperType();
 
    SubType.prototype.constructor = SubType;
    SubType.prototype.sayAge = function(){
        console.log(this.age);
    };
 
    var instance1 = new SubType("name1", 29);
    instance1.colors.push("col3");
    console.log(instance1.colors); // ["col1", "col2", "col3"]
    instance1.sayName(); // name1
    instance1.sayAge(); // 29
 
    var instance2 = new SubType("name2", 27);
    console.log(instance2.colors); // ["col1", "col2"]
    instance2.sayName(); // name2
    instance2.sayAge(); // 27
 
    /**
     * 优点：
     * (1) 使用原型链实现对原型属性和方法的继承，通过借用构造函数来实现对实例属性的继承。
     * 这样，既通过在原型上定义方法实现了函数复用，又能够保证每个实例都有它自己的属性。
     * (2) instanceof 和 isPrototypeOf() 也能够用于识别基于组合继承创建的对象。
     * 缺点：无论在什么情况下，都会调用两次超类型构造函数：一次是在创建子类型原型的时候，另一次是在子类型构造函数内部。
     * 子类型最终会包含超类型对象的全部实例属性，但不得不在调用子类型构造函数时重写这些属性。
     */
}());
```

##### 寄生组合式继承

```
(function(){
    console.info("寄生组合式继承");
    // 寄生组合式继承
 
    function object(o){
        function F(){}
        F.prototype = o;
        return new F();
    }
 
    function inheritPrototype(subType, superType){
        var prototype = Object(superType.prototype); // 创建对象,和var prototype = superType.prototype;等效？
        prototype.constructor = subType; // 增强对象 bug: 这里实际上是把superType.prototype.constructor改成了subType
        subType.prototype = prototype; // 指定对象
    }
 
    function SuperType(name){
        this.name = name;
        this.colors = ["red", "blue", "green"];
    }
    SuperType.prototype.sayName = function(){
        alert(this.name);
    };
 
    var instance1 = new SuperType();
 
    console.log(instance1 instanceof Object); // true
    console.log(instance1 instanceof SuperType); // true
    console.log(instance1 instanceof SubType); // false
 
    console.log(Object.prototype.isPrototypeOf(instance1)); // true
    console.log(SuperType.prototype.isPrototypeOf(instance1)); // true
    console.log(SubType.prototype.isPrototypeOf(instance1)); // false
 
    function SubType(name, age){
        SuperType.call(this, name);
        this.age = age;
    }
    inheritPrototype(SubType, SuperType);
    SubType.prototype.sayAge = function(){
        alert(this.age);
    };
 
    console.log(instance1 instanceof Object); // true
    console.log(instance1 instanceof SuperType); // true
    console.log(instance1 instanceof SubType); // true
 
    console.log(Object.prototype.isPrototypeOf(instance1)); // true
    console.log(SuperType.prototype.isPrototypeOf(instance1)); // true
    console.log(SubType.prototype.isPrototypeOf(instance1)); // true
 
    /**
     * 优点：只调用了一次SuperType构造函数，并且因此避免了在SubType.prototype上创建不必要的、多余的属性。
     * 缺点：事实上在调用inheritPrototype方法时，会把superType.prototype.constructor改成subType，
     * 使得对一个超类的实例使用instanceof和SubType.prototype.isPrototypeOf的结果在调用inheritPrototype前后会发生变化。
     * 为了避免这个问题可以使用：
     * function object(o){
         *     function F(){}
         *     F.prototype = o;
         *     return new F();
         * }
     * 即object(superType.prototype)来代替上文的Object(superType.prototype)。
     * 或者直接使用Object.create(superType.prototype)，即创建一个原型为等于superType.prototype的实例，
     * 但值得注意的是Object.create只有IE 9+ Firefox 4+ Safari 5+ Opera 12+ 和 Chrome才支持。
     */
}());
```
