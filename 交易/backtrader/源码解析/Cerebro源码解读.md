---
title: Cerebro源码解读
description: 来源：https://blog.csdn.net/h00cker?type=blog
published: true
date: 2023-01-24T08:09:22.545Z
tags: 交易, backtrader, 源码解读
editor: markdown
dateCreated: 2023-01-24T07:41:09.950Z
---

前面两篇文章已经一步一步展示了如何使用backtrader以及使用backtrader的一些重要概念和注意事项。但是你要真正灵活地使用backtrader实现自己的策略，还需要了解backtrader各个组成部分。本文开始，对backtrader的类进行详细的说明。为了让大家更能深入了解backtrader的运行机制，咱们基于源代码进行解读。

代码架构中，会用到元类，我们先了解元类在backtrader中的应用。

# 元类

在backtrader的类定义中，经常出现如下定义：
```python
class Cerebro(with_metaclass(MetaParams, object)):
    
class MetaLineRoot(metabase.MetaParams):

class WriterBase(with_metaclass(bt.MetaParams, object))

```

里面经常出现的metaclass，就是元类的意思。

在backtrader中，基本上所有类从元类继承。我们从顶往下看，先看metabase类。
```python
class MetaBase(type):
    def doprenew(cls, *args, **kwargs):
        return cls, args, kwargs

    def donew(cls, *args, **kwargs):
        _obj = cls.__new__(cls, *args, **kwargs)s
        return _obj, args, kwargs

    def dopreinit(cls, _obj, *args, **kwargs):
        return _obj, args, kwargs

    def doinit(cls, _obj, *args, **kwargs):
        _obj.__init__(*args, **kwargs)
        return _obj, args, kwargs

    def dopostinit(cls, _obj, *args, **kwargs):
        return _obj, args, kwargs

    def __call__(cls, *args, **kwargs):
        cls, args, kwargs = cls.doprenew(*args, **kwargs)
        _obj, args, kwargs = cls.donew(*args, **kwargs)
        _obj, args, kwargs = cls.dopreinit(_obj, *args, **kwargs)
        _obj, args, kwargs = cls.doinit(_obj, *args, **kwargs)
        _obj, args, kwargs = cls.dopostinit(_obj, *args, **kwargs)
        return _obj

```

这个元类类继承type，前面实例中还有object。这些是啥区别？这个就得从python的特性说起。在python中，一切皆为对象。甚至数据类型，以及数值都是对象。不信看看：
> print(type(2))
> <class ‘int’>
> print(isinstance(2, object))
> True
> print(isinstance(2, int))
> True

- [x] 翻译：是示例吗？


数值2是int类的实例，也是int超类object的实例。记住：所有类的最顶层基类是object。
> print(isinstance(object, type))
> True
> print(isinstance(int, type))
> True

这个看出来啥呢？object类是type的实例，所有的类都继承自object，也就是所有类都是type生成的。type就是Python在背后用来创建所有类的元类。是总结一句话：实例由类生成，类由type生成。
那元类又是啥？元类就是用来创建这些类（对象）的，元类就是类的类。
这些关系如下代码即可明了：
```python
name = 'bob'
print("name.__class__ is %s"%name.__class__)
print("name.__class__.__class__ is %s"%name.__class__.__class__)

class Bar(object): pass

print("Bar.__class__ is %s"%Bar.__class__)
print("Bar.__class__.__class__ is %s"%Bar.__class__.__class__)

mybar=Bar()
print("mybar.__class__ is %s"%mybar.__class__)
print("mybar.__class__.__class__ is %s"%mybar.__class__.__class__)

```
结果如下：
> name.__class__ is <class ‘str’>
> name.__class__ .__class__is <class ‘type’>
> Bar.__class__ is <class ‘type’>
> Bar.__class__ .__class__.class is <class ‘type’>
> mybar.__class__ is <class ‘main.Bar’>
> mybar.__class__ .__class__.class is <class ‘type’>



总之一句话：实例的类是类（创建它的类），类的类是type。

普通类和元类创建类啥区别？比较复杂，我们这里关注影响backtrader架构的的特点。

普通类和元类创建的类的一个重要区别就是：
普通类实例化的时候，先用__new__构造新的空对象，然后调用__init__方法,去初始化这个对象。如果要调用__call__,需要使用实例后加一个括号显式调用。
元类创建类实例化的时候，首先调用元类的__call__，然后才是__new__和__init__，这样有啥好处？由于要先调用__call__，我们可以控制类的生成，比如说，我们可以控制生成类的参数（普通类代码中固定死了）。实际上99%的没啥用。就是构造架构的时候简化下代码，代价是代码太难读了。

回到咱们的MetaBase类，该类定义了doprenew/donew/dopreinit/doinit/dopostinit,其中donew中调用类的__new__创建对象和doinit函数对__init__对对象进行初始化。特别注意__call__函数，直接完成预处理、实例化、初始化，后处理一系列动作，而且这个处理不需要显式调用，在类进行实例化的时候就会完成。所有MetaBase（或者其子类）创建的类都会先调用这个函数，包括Cerebro、lineseries、strategies、broker、indicator等等。

这些功能类并没有继承MetaBase，而是继承MetaBase的子类MetaParams：
```python
class MetaParams(MetaBase):
    def __new__(meta, name, bases, dct):
        # Remove params from class definition to avoid inheritance
        # (and hence "repetition")
        newparams = dct.pop('params', ())

        packs = 'packages'
        newpackages = tuple(dct.pop(packs, ()))  # remove before creation

        fpacks = 'frompackages'
        fnewpackages = tuple(dct.pop(fpacks, ()))  # remove before creation

        # Create the new class - this pulls predefined "params"
        cls = super(MetaParams, meta).__new__(meta, name, bases, dct)

        # Pulls the param class out of it - default is the empty class
        params = getattr(cls, 'params', AutoInfoClass)

        # Pulls the packages class out of it - default is the empty class
        packages = tuple(getattr(cls, packs, ()))
        fpackages = tuple(getattr(cls, fpacks, ()))

        # get extra (to the right) base classes which have a param attribute
        morebasesparams = [x.params for x in bases[1:] if hasattr(x, 'params')]

        # Get extra packages, add them to the packages and put all in the class
        for y in [x.packages for x in bases[1:] if hasattr(x, packs)]:
            packages += tuple(y)

        for y in [x.frompackages for x in bases[1:] if hasattr(x, fpacks)]:
            fpackages += tuple(y)

        cls.packages = packages + newpackages
        cls.frompackages = fpackages + fnewpackages

        # Subclass and store the newly derived params class
        cls.params = params._derive(name, newparams, morebasesparams)

        return cls

    def donew(cls, *args, **kwargs):
        clsmod = sys.modules[cls.__module__]
        # import specified packages
        for p in cls.packages:
            if isinstance(p, (tuple, list)):
                p, palias = p
            else:
                palias = p

            pmod = __import__(p)

            plevels = p.split('.')
            if p == palias and len(plevels) > 1:  # 'os.path' not aliased
                setattr(clsmod, pmod.__name__, pmod)  # set 'os' in module

            else:  # aliased and/or dots
                for plevel in plevels[1:]:  # recurse down the mod
                    pmod = getattr(pmod, plevel)

                setattr(clsmod, palias, pmod)

        # import from specified packages - the 2nd part is a string or iterable
        for p, frompackage in cls.frompackages:
            if isinstance(frompackage, string_types):
                frompackage = (frompackage,)  # make it a tuple

            for fp in frompackage:
                if isinstance(fp, (tuple, list)):
                    fp, falias = fp
                else:
                    fp, falias = fp, fp  # assumed is string

                # complain "not string" without fp (unicode vs bytes)
                pmod = __import__(p, fromlist=[str(fp)])
                pattr = getattr(pmod, fp)
                setattr(clsmod, falias, pattr)
                for basecls in cls.__bases__:
                    setattr(sys.modules[basecls.__module__], falias, pattr)

        # Create params and set the values from the kwargs
        params = cls.params()
        for pname, pdef in cls.params._getitems():
            setattr(params, pname, kwargs.pop(pname, pdef))

        # Create the object and set the params in place
        _obj, args, kwargs = super(MetaParams, cls).donew(*args, **kwargs)
        _obj.params = params
        _obj.p = params  # shorter alias

        # Parameter values have now been set before __init__
        return _obj, args, kwargs

```

需要重点关注的是donew函数，在这个函数里面分别提取package，frompackage和params并赋值，前面两个对我们影响不大，关键是params（最后几行代码）。

如何提取参数了？比如在Cerebro类中，参数定义为元组：
```python
params = (
        ('preload', True),
        ('runonce', True),
        ('maxcpus', None),
        ('stdstats', True),
        ('oldbuysell', False),
        ('oldtrades', False),
        ('lookahead', 0),
        ('exactbars', False),
        ('optdatas', True),
        ('optreturn', True),
        ('objcache', False),
        ('live', False),
        ('writer', False),
        ('tradehistory', False),
        ('oldsync', False),
        ('tz', None),
        ('cheat_on_open', False),
        ('broker_coo', True),
        ('quicknotify', False),
    )

```

用起来就不方便了，这里通过setattr直接赋值到具体的独立参数里，方便进行访问。
总结，元组在backtrader中的应用（目前看出来的）：

1. 所有类实例化的时候通过原来的__call__完成，不同的类可以进行一些特殊化的处理。
2. 参数通过donew完成映射。

总之，元类在backtrader中就是做繁琐无聊的工作，让具体功能类专注于功能的实现。

下面开始具体类的分析，先从backtrader中最重要的Cerebro开始。

# Cerebro
cerebro是backtrader系统的中心控制系统，主要的工作包括：

1. 收集所有的输入（data feeds），执行者（strategies），观测者（Observers）、评价者（Analyzers）以及文档（Writers），保证系统在任何时候都正常运行。
2. 执行回测或者实时数据输入以及交易。
3. 返回结果。
4. 画图。

下面我们一步一步地结合源代码来分析Cerebro运行机制。

特别说明下：

- Cerebro只是搭建了运行的架构，很多细节都是在组件中实现，这里重点关注Cerebro的运行机制。

另外，还有一部分代码过于非常琐碎，对我们使用影响不大，也会省略。

- 当然还有一些代码我现在也没看到应用场景，或者看错了，先记录于此，后续随着深入学习后再更正和补充。
# Cerebro的初始化
Cerebro通过如下代码进行初始化：
```python
cerebro = bt.Cerebro(**kwargs)

```

这里实例化了一个Cerebro，首先调用是Cerebro类的__init__函数（再此之前还会调用元类的一些处理，例如参数的统一处理，参见前述元类的介绍）：
```python
def __init__(self):
        self._dolive = False
        self._doreplay = False
        self._dooptimize = False
        self.stores = list()
        self.feeds = list()
        self.datas = list()
        self.datasbyname = collections.OrderedDict()
        self.strats = list()
        self.optcbs = list()  # holds a list of callbacks for opt strategies
        self.observers = list()
        self.analyzers = list()
        self.indicators = list()
        self.sizers = dict()
        self.writers = list()
        self.storecbs = list()
        self.datacbs = list()
        self.signals = list()
        self._signal_strat = (None, None, None)
        self._signal_concurrent = False
        self._signal_accumulate = False
        self._dataid = itertools.count(1)
        self._broker = BackBroker()#初始化一个缺省的broker实例。
        self._broker.cerebro = self
        self._tradingcal = None  # TradingCalendar()
        self._pretimers = list()
        self._ohistory = list()
        self._fhistory = None

```

可以看出，初始化函数只是设置了类属性（只描述公共属性，私有属性主要用于类函数实现，具体用到的时候在讨论），这些属性如下表所示，具体如何使用我们在。

| 名称 | 定义 | 说明 |
| --- | --- | --- |
| stores | list | 用于存储接收到的其他组件（例如data）消息（通知）。 |
| feeds | list | 用于存储数据源（data Feeds）的基类。 |
| datas | list | 用于存储数据源。那这个和feeds啥区别？区别就是feeds是data的基类。前面说了，整个框架用了大量类的方法，很复杂，但是简化了应用。我们主要关注应用方法。两者之间的的关系，我们后续在data类中再详述。 |
| datasbyname | OrderedDict | 就是存储数据源的名称。存储的方式是有序的字典（python中，字典dict是没有顺序的）。 |
| strats | list | 用于存储策略（strategies）类 |
| optcbs | list | 存储优化策略的回调。对执行完的优化策略进行处理的回调。啥是优化策略？本系列文章1有过介绍。 |
| observers | list | 保存observers类，具体信息在介绍该类的时候再说明。由于Cerebro是整个系统的中心，需要控制多个部件协调行动，所以需要保存这些部件的类（或实例）。 |
| analyzers | list | 保存analyzers类，具体信息在介绍该类的时候再说明。 |
| indicators | list | 保存indicators类，具体信息在介绍该类的时候再说明。 |
| writers | list | 保存writers类，具体信息在介绍该类的时候再说明。 |
| storecbs | list | 保存对notify_store消息处理的回调.对于收到存储在store消息中可以定制处理过程。 |
| datacbs | list | 保存对data类消息处理的回调。同上类似。 |
| signals | list | 保存信号。这种主要用于通过signal strategy进行回测的机制。 |
| sizers | dict | 保存sizers类，具体信息在介绍该类的时候再说明。 |

Cerebro类还支持一系列的参数（这些参数在元类中统一进行处理），实例化的时候通过**kwargs传递，参数以元组的元组形式定义，如下：
```python
params = (
        ('preload', True),
        ('runonce', True),
        ('maxcpus', None),
        ('stdstats', True),
        ('oldbuysell', False),
        ('oldtrades', False),
        ('lookahead', 0),
        ('exactbars', False),
        ('optdatas', True),
        ('optreturn', True),
        ('objcache', False),
        ('live', False),
        ('writer', False),
        ('tradehistory', False),
        ('oldsync', False),
        ('tz', None),
        ('cheat_on_open', False),
        ('broker_coo', True),
        ('quicknotify', False),
    )
```
具体信息如下表所示：

| 参数 | 取值范围 | 缺省值 | 含义 |
| --- | --- | --- | --- |
| preload | TrueFalse | TRUE | 是否为Strategies预加载传递给Cerebro的不同数据源。通常我们选择为TRUE。 |
| runonce | TrueFalse | TRUE | 前面描述过，runonce是indicators类对数据访问方法的优化以加快速度，使用的是矢量化模式来提高算法的运行速度，后续如果有大量数据需要进行回测的时候将有极大的优势。strategies和Observers通常是基于事件对数据进行处理。 |
| maxcpus | None->运行系统可用的核 | None | 指定可以用于优化的CPU核。 |
| stdstats | TrueFalse | TRUE | 如果为True的话为创建已缺省的Observers，包括Broker，Buysell和Trades。 |
| oldbuysell | TrueFalse | FALSE | 创建Observers的时候，使用新的bugsell还是之前实现的。没有特殊需求，我们使用新代码。 |
| oldtrades | TrueFalse | FALSE | 创建Observers的时候，使用新的bugsell还是之前实现的。没有特殊需求，我们使用新代码。 |
| lookahead | 数值 | 0 | 这个主要用于扩展数据的时候，扩大数据的缓存的大小。通常设置缺省值即可。 |
| exactbars | False，数值 | FALSE | 这里用于指示如何缓存数据以节约内存。False就是Lines对象数据都会加入到内存以方便计算。如果采用其他值，会有一些特殊处理减少内存，可能对画图，runonce等机制造成影响。咱们现在机器这么强，内存这么大，而且白菜价，就不要节约了，缺省False就好。 |
| optdatas | True False | TRUE | 这个也是速度处理的优化机制，主要是数据预装载、runonce等，可以提高效率。设置成缺省值就行了 |
| optreturn | True False | TRUE | 同上，也是对不同类的优化机制，可以提高效率，设置为缺省值即可。 |
| objcache | True False | FALSE | 一种实验性质的方法，没啥用，缺省值。 |
| live | True False | FALSE | 是否实时输入数据，这个咱们用不上，缺省值。 |
| writer | True False | FALSE | 设置为True的话，为自动创建一个writer，缺省输出日志到stdout。 |
| tradehistory | True False | FALSE | 设置为True的话，会记录所有策略的每一次交易信息。当然也可以在strategies里面设置 |
| oldsync | True False | FALSE | 新版本之后（1.9.0.99之后）提供新来的datas同步机制。如果你要用老的同步机制，可以设置为True。谁这么无聊呢？大家都喜新厌旧。 |
| tz | None，String | None | 记录时区信息，缺省None，使用的就是UTC，对于中国，就是“UTC+8”。对于回测，时区并不重要。 |
| cheat_on_open | True False | FALSE | 这个设置为True的话，会调用strategies的next_open方法，这个方法在next之前调用，主要用于在对订单的评估，可以基于前一天的open价发起一个订单。具体有啥用，还没想到有啥场景，咱们记住这个茬，说不定哪个策略能用到 |
| broker_coo | True False | TRUE | 和上面差不多，设置为True，broker调用set_coo开启‘cheat_on_open’。开启这一个参数，上一个参数cheat_on_open也必须打开。 |
| quicknotify | True False | FALSE | 就是在next之前发起broker的通知。回测没啥意义，只有实时数据可以快速通知。 |


可以看出，Cerebro基本没做啥实质性的工作，只是初始化了一堆容器，真正的处理还是在run之后。

# 增加数据源（Data Feeds）

增加数据源最常见的方式就是cerebro.adddata(data)，data就是必须是已经实例化的data feed。

例如：
```python
data = bt.feeds.PandasData(dataname=stock_hfq_df, fromdate=start_date, todate=end_date)  # 加载数据
cerebro.adddata(data)
```

也可以是resample和replay
```python
cerebro.resampledata(data,timeframe=bt.TimeFrame.Weeks) 
cerebro.replaydatadata(data, timeframe=bt.TimeFrame.Days)

```

系统可以接受任何数量的data feeds，包括普通的数据以及resample/replay（这俩啥意思？后面Data Feeds再细说）的数据。

下面看adddata代码：
```python
    def adddata(self, data, name=None):
        '''
        Adds a ``Data Feed`` instance to the mix.
        If ``name`` is not None it will be put into ``data._name`` which is
        meant for decoration/plotting purposes.
        '''
        if name is not None:
            data._name = name

        data._id = next(self._dataid)
        data.setenvironment(self)

        self.datas.append(data)
        self.datasbyname[data._name] = data
        feed = data.getfeed()
        if feed and feed not in self.feeds:
            self.feeds.append(feed)

        if data.islive():
            self._dolive = True

        return data

```
    
整个流程简述如下：

1. 记录下data的名称
2. 分配data的ID，注意，这里从1开始。
3. 本Cerebro和data建立关联关系：data调用setenvironment记录自己现在归当前Cerebro管，同时Cerebro将data加入到datas列表中，两人建立关系了。
4. 获取data的feed，如果feed有效，加入到feeds列表中。feed是data的基类。
5. 最后记录是否实时数据。

再看看resampdata的代码：
```python
def resampledata(self, dataname, name=None, **kwargs):
        '''
        Adds a ``Data Feed`` to be resample by the system

        If ``name`` is not None it will be put into ``data._name`` which is
        meant for decoration/plotting purposes.

        Any other kwargs like ``timeframe``, ``compression``, ``todate`` which
        are supported by the resample filter will be passed transparently
        '''
        if any(dataname is x for x in self.datas):
            dataname = dataname.clone()

        dataname.resample(**kwargs)
        self.adddata(dataname, name=name)
        self._doreplay = True

        return dataname
```

整个函数流程关键点简述如下：

1. 首先要判断resample目的dataname（data对象实例）是不是已经保存在datas列表中。如果是已经保存的，那么不能直接resample了，需要clone一个新的data对象。
2. 然后就调用data feeds的resample函数对数据进行处理，具体处理方法后续在data feeds类中再讨论。
3. resample后的data就直接调用adddata函数来处理了，同时记录._doreplay标记。

replaydata的代码：
```python
def replaydata(self, dataname, name=None, **kwargs):
        '''
        Adds a ``Data Feed`` to be replayed by the system

        If ``name`` is not None it will be put into ``data._name`` which is
        meant for decoration/plotting purposes.

        Any other kwargs like ``timeframe``, ``compression``, ``todate`` which
        are supported by the replay filter will be passed transparently
        '''
        if any(dataname is x for x in self.datas):
            dataname = dataname.clone()

        dataname.replay(**kwargs)
        self.adddata(dataname, name=name)
        self._doreplay = True

        return dataname

```

和resample处理差不多，就是需要使用data的replay函数进行处理后然后加入到Cerebro的datas列表中。

# 加入策略

Cerebro加入策略就更简单了：
```python
cerebro.addstrategy(TestStrategy)
```

看看源代码做了啥？也很简单：
```python
    def addstrategy(self, strategy, *args, **kwargs):
        '''
        Adds a ``Strategy`` class to the mix for a single pass run.
        Instantiation will happen during ``run`` time.

        args and kwargs will be passed to the strategy as they are during
        instantiation.

        Returns the index with which addition of other objects (like sizers)
        can be referenced
        '''
        self.strats.append([(strategy, args, kwargs)])
        return len(self.strats) - 1
```
  
函数注释写得很清楚，就是将strategies类加入到Cerebro用于存储的容器中（strats），返回索引，这个索引可以供其他对象引用。特别注意的是，这里只是存储了strategies类以及传递给它的参数，strategies只有在run的时候才会实例化。

# 加入其他部件

我们可以理解Cerebro是军队的的指挥中心，要打仗了，先把部队准备好。除了主力部队strategies之外，咱们还得一些配合主力行动的部队，主要包括writer（记录过程），analyzer（分析结果）以及observer(观察交易过程），提供的函数分别是：

- addwriter
- addanalyzer
- addobserver (或者 addobservermulti)

几个函数源代码如下：
```python
    def addwriter(self, wrtcls, *args, **kwargs):
        '''Adds an ``Writer`` class to the mix. Instantiation will be done at
        ``run`` time in cerebro
        '''
        self.writers.append((wrtcls, args, kwargs))

    def addsizer(self, sizercls, *args, **kwargs):
        '''Adds a ``Sizer`` class (and args) which is the default sizer for any
        strategy added to cerebro
        '''
        self.sizers[None] = (sizercls, args, kwargs)
    
    def addanalyzer(self, ancls, *args, **kwargs):
        '''
        Adds an ``Analyzer`` class to the mix. Instantiation will be done at
        ``run`` time
        '''
        self.analyzers.append((ancls, args, kwargs))

    def addobserver(self, obscls, *args, **kwargs):
        '''
        Adds an ``Observer`` class to the mix. Instantiation will be done at
        ``run`` time
        '''
        self.observers.append((False, obscls, args, kwargs))

    def addobservermulti(self, obscls, *args, **kwargs):
        '''
        Adds an ``Observer`` class to the mix. Instantiation will be done at
        ``run`` time

        It will be added once per "data" in the system. A use case is a
        buy/sell observer which observes individual datas.

        A counter-example is the CashValue, which observes system-wide values
        '''
        self.observers.append((True, obscls, args, kwargs))

```
    
代码都简单，就是将各个部队装入对应的容器，注意，这里装入的都是类以及参数，在run的时候实例化。这些部队各自功能，后续在各自类中详述。

# 更改broker

大家应该记得初始化的时候实例化了一个缺省的broker，如果你希望提供自己的broker，可以通过如下方法重写：
```python
broker = MyBroker()
cerebro.broker = broker
```

有人说，broker我记得是私有属性啊，咋直接赋值操作了？这个用的是property（装饰器），等同通过setbroker/getbroker设置/读取。

好了，部队ready，开始run了。

# 开始run

用户使用如下代码开始run：
```python
result = cerebro.run(**kwargs)
```
注意，run函数是带参数的。

## run函数代码解读

run的源代码如下（比较长，分几段进行解析）
```python
def run(self, **kwargs):
        '''The core method to perform backtesting. Any ``kwargs`` passed to it
        will affect the value of the standard parameters ``Cerebro`` was
        instantiated with.

        If ``cerebro`` has not datas the method will immediately bail out.

        It has different return values:

          - For No Optimization: a list contanining instances of the Strategy
            classes added with ``addstrategy``

          - For Optimization: a list of lists which contain instances of the
            Strategy classes added with ``addstrategy``
        '''
        self._event_stop = False  # Stop is requested

        if not self.datas:
            return []  # nothing can be run

        pkeys = self.params._getkeys()
        for key, val in kwargs.items():
            if key in pkeys:
                setattr(self.params, key, val)

        # Manage activate/deactivate object cache
        linebuffer.LineActions.cleancache()  # clean cache
        indicator.Indicator.cleancache()  # clean cache

        linebuffer.LineActions.usecache(self.p.objcache)
        indicator.Indicator.usecache(self.p.objcache)

        self._dorunonce = self.p.runonce
        self._dopreload = self.p.preload
        self._exactbars = int(self.p.exactbars)

        if self._exactbars:
            self._dorunonce = False  # something is saving memory, no runonce
            self._dopreload = self._dopreload and self._exactbars < 1

        self._doreplay = self._doreplay or any(x.replaying for x in self.datas)
        if self._doreplay:
            # preloading is not supported with replay. full timeframe bars
            # are constructed in realtime
            self._dopreload = False

        if self._dolive or self.p.live:
            # in this case both preload and runonce must be off
            self._dorunonce = False
            self._dopreload = False

        self.runwriters = list()

        # Add the system default writer if requested
        if self.p.writer is True:
            wr = WriterFile()
            self.runwriters.append(wr)

        # Instantiate any other writers
        for wrcls, wrargs, wrkwargs in self.writers:
            wr = wrcls(*wrargs, **wrkwargs)
            self.runwriters.append(wr)

        # Write down if any writer wants the full csv output
        self.writers_csv = any(map(lambda x: x.p.csv, self.runwriters))

        self.runstrats = list()

```

要点：

1. 记录_event_stop为False，说明开始启动了，需要通过stop来停止。
2. 检查是否有数据源（datas），没有的话，没法跑，退出。
3. 检查下run函数携带的参数在不在Cerebro的参数表里面，有的话，更新Cerebro实例的参数，说明Cerebro参数在初始化的时候可以设置，在run的时候也可以设置。
4. 紧接着就是参数的的处理，根据参数设置一些标记（参数的含义参见前述的表格，绝大部分用不上），清空缓存，为系统运行做好准备。
5. 如果参数指示要求提供writers（记录日志），那么就实例化一个缺省的writers。同时看看还有没有其他的writers（通过addwriter添加的），有的话，就实例化。两者都加入到runwriters容器，然后看看这些writers有没有参数要求输出csv，任意一个有这个参数，记录到writers_csv标志中。注意这里Any的用法。
6. 初始化运行策略容器（runstrats）以供后用。

下面继续：
```python
if self.signals:  # allow processing of signals
            signalst, sargs, skwargs = self._signal_strat
            if signalst is None:
                # Try to see if the 1st regular strategy is a signal strategy
                try:
                    signalst, sargs, skwargs = self.strats.pop(0)
                except IndexError:
                    pass  # Nothing there
                else:
                    if not isinstance(signalst, SignalStrategy):
                        # no signal ... reinsert at the beginning
                        self.strats.insert(0, (signalst, sargs, skwargs))
                        signalst = None  # flag as not presetn

            if signalst is None:  # recheck
                # Still None, create a default one
                signalst, sargs, skwargs = SignalStrategy, tuple(), dict()

            # Add the signal strategy
            self.addstrategy(signalst,
                             _accumulate=self._signal_accumulate,
                             _concurrent=self._signal_concurrent,
                             signals=self.signals,
                             *sargs,
                             **skwargs)

        if not self.strats:  # Datas are present, add a strategy
            self.addstrategy(Strategy)

        iterstrats = itertools.product(*self.strats)
        if not self._dooptimize or self.p.maxcpus == 1:
            # If no optimmization is wished ... or 1 core is to be used
            # let's skip process "spawning"
            for iterstrat in iterstrats:
                runstrat = self.runstrategies(iterstrat)
                self.runstrats.append(runstrat)
                if self._dooptimize:
                    for cb in self.optcbs:
                        cb(runstrat)  # callback receives finished strategy
        else:
            if self.p.optdatas and self._dopreload and self._dorunonce:
                for data in self.datas:
                    data.reset()
                    if self._exactbars < 1:  # datas can be full length
                        data.extend(size=self.params.lookahead)
                    data._start()
                    if self._dopreload:
                        data.preload()

            pool = multiprocessing.Pool(self.p.maxcpus or None)
            for r in pool.imap(self, iterstrats):
                self.runstrats.append(r)
                for cb in self.optcbs:
                    cb(r)  # callback receives finished strategy

            pool.close()

            if self.p.optdatas and self._dopreload and self._dorunonce:
                for data in self.datas:
                    data.stop()

        if not self._dooptimize:
            # avoid a list of list for regular cases
            return self.runstrats[0]

        return self.runstrats

```

要点如下：

1. 最前面是处理信号相关功能。如果要处理信号，首先看_signal_strat容器中有没有记录信号策略类（SignalStrategy，通过signal_strategy加入的，至于这个东西的用途，咱们在strategies再介绍）。如果没有的话，看看普通strategies容器strats第一个策略是不是SignalStrategy类（用到了isinstance这个函数），还不是的话（记得把普通策略类返回），就创建一个空的signal_strategy。听起来有点绕吧，总之就是用户自己通过signal_strategy函数添加的signalStrategy，普通strategy容器中的signalStrategy以及缺省创建的signalstrategy，三个按优先顺序，必须插入一个信号策略类，注意还是保存在普通strategies容器strats中，只是参数设置为signal。这里关键要记住的是，如果要使用信号回测方式，那么一定要加一个信号策略类。

2. 如果用户没有设定策略，Cerebro局加一个缺省的strategies类。大家还记得系列文章（1）中最简单程序没有加策略，系统还是一样可以运行。

3. 下面一行代码就比较重要了，使用了product将所有的strategies类存放到iterstrats迭代器中。

4. 紧接着的处理就运行strategy，分为两类处理：

> ①如果不进行性能优化，或者参数中maxcpus为1：那么直接按循序调用runstrategies（下一步详述）运行进行策略的运行。如果是优化策略（就是一个策略输入多个参数进行优化评估）的处理。执行完的策略还可以调用回调（回调可通过optcallback增加）进行处理。
> ②对于需要进行性能优化的处理：首先对数据进行reset/start/preload的处理（具体处理，后续在data feeds类详述），然后根据maxcpus参数启用多线程，在多线程中调用runstrategies进行策略运行。同样对于优化策略进行回调处理。执行完毕，还得调用data的stop方法。


# runstrategies函数代码解读

下面看runstrategies:
```python
def runstrategies(self, iterstrat, predata=False):
    '''
    Internal method invoked by ``run```to run a set of strategies
    '''
    self._init_stcount()

    self.runningstrats = runstrats = list()
    for store in self.stores:
        store.start()

    if self.p.cheat_on_open and self.p.broker_coo:
        # try to activate in broker
        if hasattr(self._broker, 'set_coo'):
            self._broker.set_coo(True)

    if self._fhistory is not None:
        self._broker.set_fund_history(self._fhistory)

    for orders, onotify in self._ohistory:
        self._broker.add_order_history(orders, onotify)

    self._broker.start()

    for feed in self.feeds:
        feed.start()

    if self.writers_csv:
        wheaders = list()
        for data in self.datas:
            if data.csv:
                wheaders.extend(data.getwriterheaders())

        for writer in self.runwriters:
            if writer.p.csv:
                writer.addheaders(wheaders)

    # self._plotfillers = [list() for d in self.datas]
    # self._plotfillers2 = [list() for d in self.datas]

    if not predata:
        for data in self.datas:
            data.reset()
            if self._exactbars < 1:  # datas can be full length
                data.extend(size=self.params.lookahead)
            data._start()
            if self._dopreload:
                data.preload()

    for stratcls, sargs, skwargs in iterstrat:
        sargs = self.datas + list(sargs)
        try:
            strat = stratcls(*sargs, **skwargs)
        except bt.errors.StrategySkipError:
            continue  # do not add strategy to the mix

        if self.p.oldsync:
            strat._oldsync = True  # tell strategy to use old clock update
        if self.p.tradehistory:
            strat.set_tradehistory()
        runstrats.append(strat)

    tz = self.p.tz
    if isinstance(tz, integer_types):
        tz = self.datas[tz]._tz
    else:
        tz = tzparse(tz)

```

要点如下：

1. 策略运行计数。

2. 初始化一个容器runningstrats用于存储正在运行的策略。

3. 然后是对部件（Cerebro控制的部队）的参数设置和初始化：信号存储（store）的启动（start）；broker参数的设置和启动。feed（feed哪来的？adddata的时候获取的）的启动。Writers的参数设置。

4. 如果predata为否的话，那么现在就开始调用preload预加载数据。部分性能优化情况下，在run的时候就preload了。

5. 下面重点来了：从iterstrat迭代器（run调用本函数带进来的，携带的是加入的strategies及其参数）提取每一个strategy和参数，进行strategy的实例化和初始化，并返回正在执行的strategy的实例，记录到runstrats容器中。如下代码非常重要，使用了元类的方法进行处理（这就是元类的好处，极大的简化处理，代价是难懂）：
```python
strat = stratcls(*sargs, **skwargs)
```

6. 记录下时区，用datas类存储的时区或者参数输入时区。

继续代码：
```python
     if runstrats:
#loop separated for clarity
             defaultsizer = self.sizers.get(None, (None, None, None))
             for idx, strat in enumerate(runstrats):
                 if self.p.stdstats:
                     strat._addobserver(False, observers.Broker)
                     if self.p.oldbuysell:
                         strat._addobserver(True, observers.BuySell)
                     else:
                         strat._addobserver(True, observers.BuySell,
                                           barplot=True)

                    if self.p.oldtrades or len(self.datas) == 1:
                        strat._addobserver(False, observers.Trades)
                    else:
                        strat._addobserver(False, observers.DataTrades)
    
                for multi, obscls, obsargs, obskwargs in self.observers:
                    strat._addobserver(multi, obscls, *obsargs, **obskwargs)
    
                for indcls, indargs, indkwargs in self.indicators:
                    strat._addindicator(indcls, *indargs, **indkwargs)
    
                for ancls, anargs, ankwargs in self.analyzers:
                    strat._addanalyzer(ancls, *anargs, **ankwargs)
    
                sizer, sargs, skwargs = self.sizers.get(idx, defaultsizer)
                if sizer is not None:
                    strat._addsizer(sizer, *sargs, **skwargs)
    
                strat._settz(tz)
                strat._start()
    
                for writer in self.runwriters:
                    if writer.p.csv:
                        writer.addheaders(strat.getwriterheaders())
    
            if not predata:
                for strat in runstrats:
                    strat.qbuffer(self._exactbars, replaying=self._doreplay)
    
            for writer in self.runwriters:
                writer.start()
    
            # Prepare timers
            self._timers = []
            self._timerscheat = []
            for timer in self._pretimers:
                # preprocess tzdata if needed
                timer.start(self.datas[0])
    
                if timer.params.cheat:
                    self._timerscheat.append(timer)
                else:
                    self._timers.append(timer)
    
            if self._dopreload and self._dorunonce:
                if self.p.oldsync:
                    self._runonce_old(runstrats)
                else:
                    self._runonce(runstrats)
            else:
                if self.p.oldsync:
                    self._runnext_old(runstrats)
                else:
                    self._runnext(runstrats)
    
            for strat in runstrats:
                strat._stop()
```
    
要点如下：

1. 开始针对运行的strategy实例，分别加入Observers、indicators、analyzer以及sizer类，并调用start函数开始启动，主要进行一些初始的处理，后续在strategies类中说明。
2. 对于运行的writers（存储在runwriters容器中，run的时候实例化），设置表头，并启动。
3. 设定定时器，并开始对strategies执行_runonce/_runonce_old/_runnext_old/_runnexth函数，这些后面单独描述。
4. 后面一段代码就不贴了，主要就是停止各部件。如果是优化策略运行的话，还要对运行结果进行分析。

下面开始runnext和runonce代码的解读，先看最常用的runnext：

# _runnext函数代码解读
```python
def _runnext(self, runstrats):
        '''
        Actual implementation of run in full next mode. All objects have its
        ``next`` method invoke on each data arrival
        '''
        datas = sorted(self.datas,
                       key=lambda x: (x._timeframe, x._compression))
        datas1 = datas[1:]
        data0 = datas[0]
        d0ret = True

        rs = [i for i, x in enumerate(datas) if x.resampling]
        rp = [i for i, x in enumerate(datas) if x.replaying]
        rsonly = [i for i, x in enumerate(datas)
                  if x.resampling and not x.replaying]
        onlyresample = len(datas) == len(rsonly)
        noresample = not rsonly

        clonecount = sum(d._clone for d in datas)
        ldatas = len(datas)
        ldatas_noclones = ldatas - clonecount
        lastqcheck = False
        dt0 = date2num(datetime.datetime.max) - 2  # default at max
        while d0ret or d0ret is None:
            # if any has live data in the buffer, no data will wait anything
            newqcheck = not any(d.haslivedata() for d in datas)
            if not newqcheck:
                # If no data has reached the live status or all, wait for
                # the next incoming data
                livecount = sum(d._laststatus == d.LIVE for d in datas)
                newqcheck = not livecount or livecount == ldatas_noclones

            lastret = False
            # Notify anything from the store even before moving datas
            # because datas may not move due to an error reported by the store
            self._storenotify()
            if self._event_stop:  # stop if requested
                return
            self._datanotify()
            if self._event_stop:  # stop if requested
                return

            # record starting time and tell feeds to discount the elapsed time
            # from the qcheck value
            drets = []
            qstart = datetime.datetime.utcnow()
            for d in datas:
                qlapse = datetime.datetime.utcnow() - qstart
                d.do_qcheck(newqcheck, qlapse.total_seconds())
                drets.append(d.next(ticks=False))

            d0ret = any((dret for dret in drets))
            if not d0ret and any((dret is None for dret in drets)):
                d0ret = None

            if d0ret:
                dts = []
                for i, ret in enumerate(drets):
                    dts.append(datas[i].datetime[0] if ret else None)

                # Get index to minimum datetime
                if onlyresample or noresample:
                    dt0 = min((d for d in dts if d is not None))
                else:
                    dt0 = min((d for i, d in enumerate(dts)
                               if d is not None and i not in rsonly))

                dmaster = datas[dts.index(dt0)]  # and timemaster
                self._dtmaster = dmaster.num2date(dt0)
                self._udtmaster = num2date(dt0)

                # slen = len(runstrats[0])
                # Try to get something for those that didn't return
                for i, ret in enumerate(drets):
                    if ret:  # dts already contains a valid datetime for this i
                        continue

                    # try to get a data by checking with a master
                    d = datas[i]
                    d._check(forcedata=dmaster)  # check to force output
                    if d.next(datamaster=dmaster, ticks=False):  # retry
                        dts[i] = d.datetime[0]  # good -> store
                        # self._plotfillers2[i].append(slen)  # mark as fill
                    else:
                        # self._plotfillers[i].append(slen)  # mark as empty
                        pass

                # make sure only those at dmaster level end up delivering
                for i, dti in enumerate(dts):
                    if dti is not None:
                        di = datas[i]
                        rpi = False and di.replaying   # to check behavior
                        if dti > dt0:
                            if not rpi:  # must see all ticks ...
                                di.rewind()  # cannot deliver yet
                            # self._plotfillers[i].append(slen)
                        elif not di.replaying:
                            # Replay forces tick fill, else force here
                            di._tick_fill(force=True)

                        # self._plotfillers2[i].append(slen)  # mark as fill

            elif d0ret is None:
                # meant for things like live feeds which may not produce a bar
                # at the moment but need the loop to run for notifications and
                # getting resample and others to produce timely bars
                for data in datas:
                    data._check()
            else:
                lastret = data0._last()
                for data in datas1:
                    lastret += data._last(datamaster=data0)

                if not lastret:
                    # Only go extra round if something was changed by "lasts"
                    break

            # Datas may have generated a new notification after next
            self._datanotify()
            if self._event_stop:  # stop if requested
                return

            if d0ret or lastret:  # if any bar, check timers before broker
                self._check_timers(runstrats, dt0, cheat=True)
                if self.p.cheat_on_open:
                    for strat in runstrats:
                        strat._next_open()
                        if self._event_stop:  # stop if requested
                            return

            self._brokernotify()
            if self._event_stop:  # stop if requested
                return

            if d0ret or lastret:  # bars produced by data or filters
                self._check_timers(runstrats, dt0, cheat=False)
                for strat in runstrats:
                    strat._next()
                    if self._event_stop:  # stop if requested
                        return

                    self._next_writers(runstrats)

        # Last notification chance before stopping
        self._datanotify()
        if self._event_stop:  # stop if requested
            return
        self._storenotify()
        if self._event_stop:  # stop if requested
            return

```

要点如下：

1. 首先按照周期大小排序datas，将具有最小周期的data作为data0，其他的放到datas1容器中。
2. 将resample和replay的数据索引单独保存，做好几个标记。计算clone数据（啥时候clone？加入resample和replay数据的时候都会clone）的个数、非clone数据的个数以及数据总的数量。
3. dt0初始化最大值为最大日期（9999年12月31日）的多边格里高利度序数（每一日期对应一个独一无二的数，方便程序处理）。
4. 如果所有数据是live或者没有新的数据（数据如何更新，后续讲data类的时候再详述），那么就需要等待（newqcheck这一块的处理）。
5. 识别有没有收到stop消息（从store容器中读取），在等待数据的时候，可能因为失败没有进一步的数据。
6. 所有的data都会调用do_qcheck（输入等待的时间，至于干啥用的，以后在data类的时候再详述）以及next。
7. 后面一段确实比较难以理解，其总体思路是寻找主data（dmaster），通常应该是周期最短的数据。然后其他数据以此数据作为基准。（目前理解有限，后续在解读Data相关类的时候针对此场景再补充分析）
8. 在此过程中，需要调用_brokernotify和_datanotify检测是否有停止信号。
9. 数据完备（bar成功产生之后），就调用策略的next方法，这样就进入到我们定制策略next中进行处理了。
# _runonce函数代码解读

runonce之前专门讲过，其特点是采取矢量模式，直接对数据进行处理，以提高效率。系统缺省采用runonce的方式。
```python
def _runonce(self, runstrats):
        '''
        Actual implementation of run in vector mode.

        Strategies are still invoked on a pseudo-event mode in which ``next``
        is called for each data arrival
        '''
        for strat in runstrats:
            strat._once()
            strat.reset()  # strat called next by next - reset lines

        # The default once for strategies does nothing and therefore
        # has not moved forward all datas/indicators/observers that
        # were homed before calling once, Hence no "need" to do it
        # here again, because pointers are at 0
        datas = sorted(self.datas,
                       key=lambda x: (x._timeframe, x._compression))

        while True:
            # Check next incoming date in the datas
            dts = [d.advance_peek() for d in datas]
            dt0 = min(dts)
            if dt0 == float('inf'):
                break  # no data delivers anything

            # Timemaster if needed be
            # dmaster = datas[dts.index(dt0)]  # and timemaster
            slen = len(runstrats[0])
            for i, dti in enumerate(dts):
                if dti <= dt0:
                    datas[i].advance()
                    # self._plotfillers2[i].append(slen)  # mark as fill
                else:
                    # self._plotfillers[i].append(slen)
                    pass

            self._check_timers(runstrats, dt0, cheat=True)

            if self.p.cheat_on_open:
                for strat in runstrats:
                    strat._oncepost_open()
                    if self._event_stop:  # stop if requested
                        return

            self._brokernotify()
            if self._event_stop:  # stop if requested
                return

            self._check_timers(runstrats, dt0, cheat=False)

            for strat in runstrats:
                strat._oncepost(dt0)
                if self._event_stop:  # stop if requested
                    return

                self._next_writers(runstrats)

```

要点如下：

1. 开始三行代码是关键，针对每个strategy，运行_once方法并进行reset。实际上就是将数据索引指向0.
2. 仍然将数据按照周期（timeframe）大小排序。
3. 调用advance_peek看下一个时间（索引为1，还记得索引为1的含义？），最近时间记为dt0.
4. 对于最近时间的的data调用advance（嗯，又是数据的处理，后面再具体讲吧）
5. 如果设置了cheat_on_open，那么在策略的next之前调用next_open方法。
6. 检测是否收到broker的stop消息。
7. 下面开始调用策略的、_oncepost方法，这个方法中会调用到strategy的next方法，怎么玩的？必须要到数据类中探索了。

总的来看，Cerebro的run过程，实际上按照时间驱动数据不断运行，在此过程中协调其他部件协同工作。

# 画图

最后可以调用plot方法进行可视化输出：
```python
    def plot(self, plotter=None, numfigs=1, iplot=True, start=None, end=None,
             width=16, height=9, dpi=300, tight=True, use=None,
             **kwargs):
        '''
        Plots the strategies inside cerebro

        If ``plotter`` is None a default ``Plot`` instance is created and
        ``kwargs`` are passed to it during instantiation.

        ``numfigs`` split the plot in the indicated number of charts reducing
        chart density if wished

        ``iplot``: if ``True`` and running in a ``notebook`` the charts will be
        displayed inline

        ``use``: set it to the name of the desired matplotlib backend. It will
        take precedence over ``iplot``

        ``start``: An index to the datetime line array of the strategy or a
        ``datetime.date``, ``datetime.datetime`` instance indicating the start
        of the plot

        ``end``: An index to the datetime line array of the strategy or a
        ``datetime.date``, ``datetime.datetime`` instance indicating the end
        of the plot

        ``width``: in inches of the saved figure

        ``height``: in inches of the saved figure

        ``dpi``: quality in dots per inches of the saved figure

        ``tight``: only save actual content and not the frame of the figure
        '''
        if self._exactbars > 0:
            return

        if not plotter:
            from . import plot
            if self.p.oldsync:
                plotter = plot.Plot_OldSync(**kwargs)
            else:
                plotter = plot.Plot(**kwargs)

        # pfillers = {self.datas[i]: self._plotfillers[i]
        # for i, x in enumerate(self._plotfillers)}

        # pfillers2 = {self.datas[i]: self._plotfillers2[i]
        # for i, x in enumerate(self._plotfillers2)}

        figs = []
        for stratlist in self.runstrats:
            for si, strat in enumerate(stratlist):
                rfig = plotter.plot(strat, figid=si * 100,
                                    numfigs=numfigs, iplot=iplot,
                                    start=start, end=end, use=use)
                # pfillers=pfillers2)

                figs.append(rfig)

            plotter.show()

        return figs

```
   
这个过程比较简单，就是引入plotter模块，针对每一个运行的strategy画图。贴字段代码的目的，就是告诉大家，Cerebro没干啥，都是驱动别人干活。

# 总结

Cerebro解读完了，读完好像还是一头雾水，没看懂系统到底咋玩的啊！这就对了，因为Cerebor就没干啥，我们需要深入了解Cerebro驱动的部件是如何工作的，反过来你就会理解Cerebor。

关于这个文档的顺序，本来我想从底层部件代码解读，但是想到没有一个整体的框架，看部件的代码也很难理解。所以先从Cerebro开始，从整体上看Backtrader是怎么运行的，再看看每一个部件工作机制以及如何在整个系统中起作用，最后在整体上完全掌握。所以，现在看的不太明白也没啥，只要知道Cerebro的运行顺序，以及在哪里用到什么部件就行了。

下一篇文章我们开始解读data相关类，看看数据是如何保存以及如何驱动的。
