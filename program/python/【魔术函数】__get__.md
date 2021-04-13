##### `__get*__ __set*__ __del*__` 魔术函数

注意，这几个函数只针对新式类有效，经典类是无效的
```python
object.__getattr__(self, name)
object.__setattr__(self, name, value)
object.__delattr__(self, name)
object.__getattribute__(self, name)
object.__get__(self, instance, owner)
object.__set__(self, instance, value)
object.__delete__(self, instance)
```

参考: [customizing-attribute-access](https://docs.python.org/3/reference/datamodel.html#customizing-attribute-access)  
这几个函数必须和`AttributeError`这个异常一起说明

##### `__dict__`变量

这个变量存储了模块，类等对象的可访问属性，是一个字典，key是属性名，value是属性值，我们通过`x.a`访问x对象的a属性的时候，其实就是`x.__dict__['a']`的方式在访问

##### `__getattr__`和`__getattribute__`

1. 对象(注意这里说的是对象，不是说的类，意思是所有的python对象)如果定义了`__getattribute__`这个函数，那么通过`x.a`的方式访问的时候，会无条件的走到这个函数里面，也就是不管`__dict__`是否有这个属性，或者这个属性是什么，都不管，一定会调用`__getattribute__`这个函数，其实当定义了这个函数后`__dict__`已经不存在了，当用`a.__dict__`的方式访问`__dict__`属性的时候，已经无条件的走到了这个函数内部，但是注意，它只是覆盖了a这个实例的`__dict__`属性，类本身的`__dict__`还是仍然存在的，因为类也是一个对象，它有自己独立的创建者，而它的创建者有可能没有实现`__getattribute__`函数，也有可能在里面对`__dict__`重新定义了

```python
def fun2():
    print 'fun2'

class A(object):
    def fun1(self):
        print 'fun1'
    def __getattribute__(self, name):
        return fun2

a = A()
print a.__dict__ #调用了__getattribute__
a.fun1() #调用了__getattribute__
a.fun3() #调用了__getattribute__
<function fun2 at 0x0254B330>
fun2
fun2
```

2. 如果`__getattribute__`抛出了`AttributeError`，此时如果还定义了`__getattr__`，这个时候才会走到`__getattr__`这个函数里面，当然如果`__getattribute__`显示的调用了`__getattr__`也是可以的

```python
class A(object):
    def fun1(self):
        print 'fun1'

    def __getattribute__(self, name):
        print '__getattribute__'
        raise AttributeError()

    def __getattr__(self, name):
        print '__getattr__'

a = A()
a.fun1 #__getattribute__抛出AttributeError异常，两个都会被调用
a.fun3 #__getattribute__抛出AttributeError异常，两个都会被调用
```

3. 如果没有定义`__getattribute__`，`__getattr__`会在`__dict__`找不到的时候被调用

```python
class A(object):
    def fun1(self):
        print 'fun1'

    # def __getattribute__(self, name):
    #     print '__getattribute__'

    def __getattr__(self, name):
        print '__getattr__'

a = A()
a.fun1
a.fun3 # 没有定义__getattribute__，走到了__getattr__里面
```

4. 使用`__getattr__`的时候注意循环调用的情况，比如:

```python
class A(object):
    def fun1(self):
        print 'fun1'

    def __getattr__(self, name):
        print '__getattr__'
        return self.b

a = A()
a.fun1 # 正常调用fun1
a.fun3 # 会走到__getattr__，此时里面又有一个self.b，还是找不到，再次进入了__getattr__，如此循环，直到python抛出异常如下
# RuntimeError: maximum recursion depth exceeded while calling a Python object
```

##### 描述符

```python
class Des(object):
    def __get__(self, instance, owner=None):
        print '__get__'

    def __getattribute__(self, name):
        print 'Des::__getattribute__'
        raise AttributeError()

    def __getattr__(self, name):
        print 'Des::__getattr__'

class Owner(object):
    des = Des()

    def __getattribute__(self, name):
        print 'Owner::__getattribute__'
        raise AttributeError()

    def __getattr__(self, name):
        print 'Owner::__getattr__'

ins = Owner()
print ins.__dict__
print Owner.__dict__
print '*' * 50
print ins.des
print Owner.des
print '*' * 50
Owner::__getattribute__ #ins.__dict__ 无条件的调用__getattribute__
Owner::__getattr__ # __getattribute__抛出异常，走到了__getattr__
None # 最后__getattr__返回None ---> 这里才是ins.__dict__调用的结束，后面是Owner.__dict__的开始
{'__module__': '__main__', 'des': <__main__.Des object at 0x02513290>, '__getatt
ribute__': <function __getattribute__ at 0x0250B870>, '__getattr__': <function _
_getattr__ at 0x0250B8B0>, '__dict__': <attribute '__dict__' of 'Owner' objects>
, '__weakref__': <attribute '__weakref__' of 'Owner' objects>, '__doc__': None} #Owner.__dict__ 本身是存在的
**************************************************
Owner::__getattribute__ #ins.des类似于上的ins.__dict__
Owner::__getattr__ #ins.des类似于上的ins.__dict__
None #ins.des类似于上的ins.__dict__
__get__ #Owner.des就会访问描述符
None
**************************************************
```

实例访问属性的访问顺序:
1. 有`__getattribute__`会无条件的走到这个函数
2. 有`__getattr__`并且在找不到属性或者`__getattribute__`抛出`AttributeError`异常的时候，会走到这个函数(当然如果找不到属性，又没有定义`__getattribute__`函数，也会走到此函数内)
3. 当上面的函数都未定义，访问属性的时候，会先找自己的`__dict__`，如果找不到会找所属类的`__dict__`，如果还是找不到，会找父类的`__dict__`，直到找到或者抛出异常(实例的属性会覆盖同名的类的属性)
4. 描述符只能属于类，如果属于实例，实例获得是的描述符这个类的一个实例，而不会去访问`__get__`等函数
5. 当类和实例有同名的属性，这个属性实现了描述符的协议(`__get__、__set__、__del__`)，则会区分出资料描述符(实现了`__get__`和至少`__set__、__del__`中的一个)和非资料描述符(只实现了`__get__`)
6. 根据上面的点，如果是资料描述符，那么类的属性覆盖实例的属性，如果是非资料描述符，那么实例的属性会覆盖类的属性

```python
class Des(object):
    def __get__(self, instance, owner=None):
        print '__get__'

    def __set__(self, instance, value):
        print '__set__'

class Owner(object):
    des = Des()

    def __init__(self):
        self.des = 1 #这里会访问到Des的__set__函数

ins = Owner()
print ins.des # 由于Des是资料描述符，则这里访问会走到__get__函数里面，如果把Des的__set__函数注释掉，这里访问得到的是1(Owner的__init__里面定义的)
```

换句话说：资料描述符优先于dict，而dict查找优先于非资料描述符
instance就是具体的实例对象，也就是ins，owner是具体的类，就是Owner

实例访问属性的等价格式:

```python
class Owner(object):
    def fun(self):
        print 'fun'

ins = Owner()
ins.fun()
# 等同于下面
Owner.__dict__['fun'](ins)
```

##### 使用场景

###### 实现赖加载

###### 初入

```python
# -*- coding:utf-8 -*-
class deco(object):
    def __init__(self, func):
        self._func = func
        
class A(object):
    @deco
    def func(self):
        pass
        
a = A()
print(a.func) # <__main__.deco object at 0x000000000399BE88> 也就是func相当于是一个deco的实例，类推到描述符
```

###### get引入

```python
# -*- coding:utf-8 -*-
class deco(object):
    def __init__(self, func):
        self._func = func
        
    def __get__(self, instance, owner):
        print('get')
        
class A(object):
    @deco
    def func(self):
        pass
        
a = A()
print(a.func) # get, 走到deco的__get__方法
```

###### 举例

假设我们有一个工具类MongoUtil，它的作用是封装一些数据库操作。例如：

```python
import pymongo
 
class MongoUtil:
    def __init__(self):
        connect = pymongo.MongoClient()
        db = connect.tieba
        self.post = db.post
        self.user = db.user
        
    def write_post(self, post):
        # 处理post信息
        self.post.insert_one(post)
    
    def read_user_info(self):
        rows = self.user.find()
        # 读取user信息并处理
        # ...
```

我们发现这样写有一个问题——类在初始化的时候，就会创建数据库的链接。但我们并不是在类刚刚初始化时就读写数据库。
为了让数据库在第一次使用时再创建连接，我们就要实现懒加载机制：

```python
import pymongo
 
class MongoUtil:
    def __init__(self):
        connect = pymongo.MongoClient()
        self.db = connect.tieba
        self.post = None
        self.user = None
        
    def write_post(self, post):
        # 处理post信息
        if not self.post:
            self.post = self.db.post
        self.post.insert_one(post)
    
    def read_user_info(self):
        if not self.user:
            self.user = self.db.user
        rows = self.user.find()
        # 读取user信息并处理
        # ...
```

这样写确实实现了懒加载，但每一个操作都需要判断当前是否联系到了对应的集合中。这样就会出现大量的重复代码。
为了解决这个问题，我们可以使用装饰器实现一个懒加载机制：

```python
import pymongo
 
class lazy:
    def __init__(self, func):
        self.func = func
 
    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            value = self.func(instance)
            setattr(instance, self.func.__name__, value)
            return value
 
class MongoUtil:
    def __init__(self):
        connect = pymongo.MongoClient()
        self.db = connect.tieba
 
    @lazy
    def post(self):
        return self.db.post
    
    @lazy
    def user(self):
        return self.db.user
    
    def write_post(self, post):
        # 处理post信息
        self.post.insert_one(post)
    
    def read_user_info(self):
        rows = self.user.find()
        # 读取user信息并处理
        # ...
```

> 我们实现了一个装饰器类lazy来装饰两个类属性post和user。当self.post第一次被调用时，它会正常连接结合，当第二次或以上访问self.post时，就会直接使用第一次返回的对象，不会再次连接MongoDB的集合。self.user同理
使用MongoDB举例只是为了说明基于装饰器的类属性懒加载的代码写法。而实际上，pymongo已经自动实现了懒加载机制，当我们直接db.tieba.post时，它并不会真的去连接MongoDB，只有当我们要增删改查集合里面的数据时，pymongo才会创建连接