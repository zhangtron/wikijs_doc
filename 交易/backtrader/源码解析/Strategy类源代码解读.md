上一篇文章详细介绍数据相关类，本文开始 介绍和Cerebro一样重要的Strategy。如果Cerebro是大脑，那Strategy就是心脏，所有血液（数据）流经心脏（Strategy）处理。

老规矩，先上Strategies的家族图谱。
![backtrade策略家谱图.png](https://cdn.nlark.com/yuque/0/2022/png/29177964/1669175236148-0435ba10-371b-41f7-9c72-9871f4eb66b9.png#averageHue=%23f9f9f1&clientId=uf2eefe74-7ee2-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=2806&id=u6c8cc68a&margin=%5Bobject%20Object%5D&name=backtrade%E7%AD%96%E7%95%A5%E5%AE%B6%E8%B0%B1%E5%9B%BE.png&originHeight=2806&originWidth=1168&originalType=binary&ratio=1&rotation=0&showTitle=false&size=435636&status=done&style=none&taskId=u37cf34c9-b817-40f3-8020-97d25e62dda&title=&width=1168)
Strategy家族图谱


同样的，牢牢记住这个图，这是咱们的家谱。

Note1：图中类名后面加标号直接和未加标号同名类定义完全相同，比如LineRoot1和Lineroot是相同的。主要是为了图形清爽，不然太多交叉，看不清楚。

Note2：由于Strategy类通常要自定义，所以增加了一个自定义类继承自Strategy。

# 一个简单的均线Strategy
为了更好地进行代码解读，我们提供了一个简单的定制MyCustomStrategy类，继承Strategy，完成均线策略。
```python
class MyCustomStrategy(bt.Strategy):
    params = (
        ('maperiod', 5),
    )

    def log(self, txt, dt=None):
        ''' Logging function fot this strategy'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # Keep a reference to the "close" line in the data[0] dataseries
        self.dataclose = self.datas[0].close

        # To keep track of pending orders and buy price/commission
        self.order = None
        self.buyprice = None
        self.buycomm = None

        # Add a MovingAverageSimple indicator
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Buy/Sell order submitted/accepted to/by broker - Nothing to do
            return

        # Check if an order has been completed
        # Attention: broker could reject order if not enough cash
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:  # Sell
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        # Simply log the closing price of the series from the reference
        self.log('Close:%.3f' % self.data.close[0])
        self.log('turnover, %.8f' % self.data.turnover[0])
        # Check if an order is pending ... if yes, we cannot send a 2nd one
        if self.order:
            return

        #Check if we are in the market
        if not self.position:

            
            if self.dataclose[0] > self.sma[0]:

                # 大于均线就买
                self.log('BUY CREATE, %.2f' % self.dataclose[0])

                # Keep track of the created order to avoid a 2nd order
                self.order = self.buy()

        else:

            if self.dataclose[0] < self.sma[0]:
                # 小于均线卖卖卖！
                self.log('SELL CREATE, %.2f' % self.dataclose[0])

                # Keep track of the created order to avoid a 2nd order
                self.order = self.sell()

```

此段代码具体解释，请参见系列文章1.

下面以人的一生来描述Strategy的发展过程。

# 孕育阶段
在Cerebro代码详解一文中，我们说明了如何使用strategy：

- 首先通过Cerebro中添加Strategy类（本文以添加定制类MyCustomStrategy为例），这个只是添加类，还没有实例化，和Strategy实现没啥关系。
- 然后Cerebro在runstrategies函数中实例化和初始化，代码如下：
```python
for stratcls, sargs, skwargs in iterstrat:
    sargs = self.datas + list(sargs)
    try:
        strat = stratcls(*sargs, **skwargs)
    except bt.errors.StrategySkipError:
        continue  # do not add strategy to the mix    
```
下面我们看看Strategy的实例化和初始化。

# Strategy的实例化
从家谱图中我们可以看出，MyCustomStrategy的父类中有元类，所以实例化会受MetaBase元类的控制。首先到MetaBase的__call__走一圈：

- 首先是doprenew，没人重写，啥也没做。

- 然后donew，顺着家谱找，MetaStrategy重写了donew，但是其中第一句话，就先调用父类的donew，父类是MetaLineIterator，它的donew又调用到自己父类的donew，继续MetaLineSeries–>MetaLineRoot->MetaParams->MetaBase.太复杂了，俄罗斯套娃啊。熟悉的味道，又到MetaParams，这个之前介绍过，就是调用MetaBase的donew实例化MyCustomStrategy，并且完成参数到属性的映射。

- MyCustomStrategy实例化完成之后，就到MetaLineRoot的donew，这个系列文章4也讲过，主要是查找自己的owner是谁？MyCustomStrategy是初始化的发起者，所以它没有owner。

- 继续到MetaLineSeries的donew，这个系列文章4也讲过（为啥都讲过？这也是面向对象的好处啊，代码大量复用）。 首先初始化一个AutoInfoClass保存plotinfo，并根据参数设置属性，画图使用，暂时忽略，后续专题再讲。然后就是最重要的也实例化了和初始化了lines以及LineBuffer（这一块在文章4中有详细描述），这里注意下，LineBuffer的owner就是MyCustomStrategy。这样MyCustomStrategy和数据源一样也拥有lines了，请参见家谱图。

- 下一步就是MetaLineIterator，这个就是strategy特有的，看代码：
```python
def donew(cls, *args, **kwargs):
        _obj, args, kwargs = \
            super(MetaLineIterator, cls).donew(*args, **kwargs)

        # Prepare to hold children that need to be calculated and
        # influence minperiod - Moved here to support LineNum below
        _obj._lineiterators = collections.defaultdict(list)

        # Scan args for datas ... if none are found,
        # use the _owner (to have a clock)
        mindatas = _obj._mindatas
        lastarg = 0
        _obj.datas = []
        for arg in args:
            if isinstance(arg, LineRoot):
                _obj.datas.append(LineSeriesMaker(arg))

            elif not mindatas:
                break  # found not data and must not be collected
            else:
                try:
                    _obj.datas.append(LineSeriesMaker(LineNum(arg)))
                except:
                    # Not a LineNum and is not a LineSeries - bail out
                    break

            mindatas = max(0, mindatas - 1)
            lastarg += 1

        newargs = args[lastarg:]

        # If no datas have been passed to an indicator ... use the
        # main datas of the owner, easing up adding "self.data" ...
        if not _obj.datas and isinstance(_obj, (IndicatorBase, ObserverBase)):
            _obj.datas = _obj._owner.datas[0:mindatas]

        # Create a dictionary to be able to check for presence
        # lists in python use "==" operator when testing for presence with "in"
        # which doesn't really check for presence but for equality
        _obj.ddatas = {x: None for x in _obj.datas}

        # For each found data add access member -
        # for the first data 2 (data and data0)
        if _obj.datas:
            _obj.data = data = _obj.datas[0]

            for l, line in enumerate(data.lines):
                linealias = data._getlinealias(l)
                if linealias:
                    setattr(_obj, 'data_%s' % linealias, line)
                setattr(_obj, 'data_%d' % l, line)

            for d, data in enumerate(_obj.datas):
                setattr(_obj, 'data%d' % d, data)

                for l, line in enumerate(data.lines):
                    linealias = data._getlinealias(l)
                    if linealias:
                        setattr(_obj, 'data%d_%s' % (d, linealias), line)
                    setattr(_obj, 'data%d_%d' % (d, l), line)

        # Parameter values have now been set before __init__
        _obj.dnames = DotDict([(d._name, d)
                               for d in _obj.datas if getattr(d, '_name', '')])

        return _obj, newargs, kwargs

```

1. 首先会参见一个特殊的字典_lineiterators用来保存line的遍历，主要用于计算最小周期（啥是最小周期？请参考系列文章2）。这里使用的collections.defaultdict类，继承python内置的dic。具体用法可以网上搜索下。
2. 下一步就是将参数中携带的datas记录到MyCustomStrategy的datas中。这个参数谁输入的？请看本文的第一段代码，Cerebro实例化的时候输入的，datas就是Cerebro加载的数据源。这样Strategy就可以和加载的原始数据建立关联了。这一段代码还有点技巧，只添加确实是数据类的参数。
3. 在下面的处理是针对indicator类的，忽略，有专题讲。
4. 然后就是对数据的line名字进行处理，方便访问。各种访问方法在文章2详细描述过。
- 这个处理完成之后，就到MetaStrategy的donew了：
```python
def donew(cls, *args, **kwargs):
        _obj, args, kwargs = super(MetaStrategy, cls).donew(*args, **kwargs)

        # Find the owner and store it
        _obj.env = _obj.cerebro = cerebro = findowner(_obj, bt.Cerebro)
        _obj._id = cerebro._next_stid()

        return _obj, args, kwargs
```

这里就是记住策略的运行环境env也就是咱们的大脑Cerebro，并记住自己在Cerebro中的标识（由Cerebro分配）。

至此，MyCustomStrategy的实例化完成。看看一个实例化做了多少事情。

# Strategy的初始化
继续MetaBase的__call__函数，前面已经完成了实例化，现在开始初始化了。首先调用dopreinit函数，看家谱图，又是MetaStrategy定义了dopreinit，在dopreinint中，又开始调用父类的dopreinit，俄罗斯套娃开始，这次是MetaStrategy->MetaLineIterator->Metabase,还好只有三层套娃，从上往下看：

- MetaBase没做啥，直接到MetaLineIterator：
```python
def dopreinit(cls, _obj, *args, **kwargs):
        _obj, args, kwargs = \
            super(MetaLineIterator, cls).dopreinit(_obj, *args, **kwargs)

        # if no datas were found use, use the _owner (to have a clock)
        _obj.datas = _obj.datas or [_obj._owner]

        # 1st data source is our ticking clock
        _obj._clock = _obj.datas[0]

        # To automatically set the period Start by scanning the found datas
        # No calculation can take place until all datas have yielded "data"
        # A data could be an indicator and it could take x bars until
        # something is produced
        _obj._minperiod = \
            max([x._minperiod for x in _obj.datas] or [_obj._minperiod])

        # The lines carry at least the same minperiod as
        # that provided by the datas
        for line in _obj.lines:
            line.addminperiod(_obj._minperiod)

        return _obj, args, kwargs


```

这一段代码就是根据基于所有数据初始化最小周期，并将最小周期同步到所有line。最小周期的概念和详细解释参见文章2.

- 下一个就是MetaStrategy的dopreinit：
```python
def dopreinit(cls, _obj, *args, **kwargs):
        _obj, args, kwargs = \
            super(MetaStrategy, cls).dopreinit(_obj, *args, **kwargs)
        _obj.broker = _obj.env.broker
        _obj._sizer = bt.sizers.FixedSize()
        _obj._orders = list()
        _obj._orderspending = list()
        _obj._trades = collections.defaultdict(AutoDictList)
        _obj._tradespending = list()

        _obj.stats = _obj.observers = ItemCollection()
        _obj.analyzers = ItemCollection()
        _obj._alnames = collections.defaultdict(itertools.count)
        _obj.writers = list()

        _obj._slave_analyzers = list()

        _obj._tradehistoryon = False

        return _obj, args, kwargs

```

这里主要就是和初始化自己相关部件（broker来自Cerebro）或者提供相应的容器。

dopreinit完成之后，就进行doinit了。还是一样，查家谱，看看有没有俄罗斯套娃，居然没有，直接就到MetaBase的doinit了，其中直接调用MyCustomStrategy的__init__了，也就是我们定制类的初始化，在定制类的初始化过程中，通常会引用策略中需要使用的数据，尤其是增加Indicator的。具体可以看看文章1相关描述。在本例中，我们初始化了一个移动平均指标，并引用需要使用的close数据：
```python
def __init__(self):
        # 引用需要使用的数据line。
        self.dataclose = self.datas[0].close

        # 跟踪委托单、买价以及佣金等
        self.order = None
        self.buyprice = None
        self.buycomm = None

        # 增加简单移动平均指标
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)
```

初始化这就完成了？No，还有初始化后处理，看dopostinit函数，看套娃，是MetaStrategy->MetaLineIterator->MetaBase,还好只有三层套娃，从上往下看：

- 首先到MetaBase，啥也没干。
- 再到MetaLineIterator，代码如下：
```python
def dopostinit(cls, _obj, *args, **kwargs):
        _obj, args, kwargs = \
            super(MetaLineIterator, cls).dopostinit(_obj, *args, **kwargs)

        # my minperiod is as large as the minperiod of my lines
        _obj._minperiod = max([x._minperiod for x in _obj.lines])

        # Recalc the period
        _obj._periodrecalc()

        # Register (my)self as indicator to owner once
        # _minperiod has been calculated
        if _obj._owner is not None:
            _obj._owner.addindicator(_obj)

        return _obj, args, kwargs
```
首先记录最小周期，最小周期取所有Line的最小周期之最大值，同上为1.
然后调用_periodrecalc重新计算下最小周期，这个函数代码如下：
```python
def _periodrecalc(self):
        # last check in case not all lineiterators were assigned to
        # lines (directly or indirectly after some operations)
        # An example is Kaufman's Adaptive Moving Average
        indicators = self._lineiterators[LineIterator.IndType]
        indperiods = [ind._minperiod for ind in indicators]
        indminperiod = max(indperiods or [self._minperiod])
        self.updateminperiod(indminperiod)
```
 这个函数主要就是从Line迭代器中取出Indicator对象，获取所有Indicator的最小周期，我们的MyCustomStrategy中，增加了一个移动平均线，周期是5.所以这里最小周期是5

- 再到MetaStrategy，主要就是初始化sizer，比较简单，就不贴代码。

至此，咱们的MyCustomStrategy孕育成功，马上出生了。

# 出生阶段
Strategy实例化和初始化之后，就是startup了。Startup在Cerebro的runstrategies函数中调用（参见文章3）：

```python
strat._start()    
```
这里只是提供片段，Cerebro对Strategy的完整驱动过程在文章3有描述。

调用_start，一样需要查家谱，找到Strategy：
```python
def _start(self):
        self._periodset()

        for analyzer in itertools.chain(self.analyzers, self._slave_analyzers):
            analyzer._start()

        for obs in self.observers:
            if not isinstance(obs, list):
                obs = [obs]  # support of multi-data observers

            for o in obs:
                o._start()

        # change operators to stage 2
        self._stage2()

        self._dlens = [len(data) for data in self.datas]

        self._minperstatus = MAXINT  # start in prenext

        self.start()
```

第一步就是调用_peroidset进行最小周期设定，这个函数主要处理就是依据各line数据进行计算，最小周期的原理前面文章详述过，这个函数代码就不贴了。

然后就是启动analyzer和observer，具体在对应类中再详述。

下面就是调用_stage2进入阶段2.这个又需要看套娃了（看家谱），LineIterator->LineMultiple->Lineroot。LineRoot中完成设置操作标识（_opstage）为2。然后继续到LineMultiple，该函数直接遍历所有Line，调用LineBuffer的_stage2函数，在LineBuffer的函数中也是设置各自的标识为2（实际也是通过LineRoot完成，还记得LineBuffer也继承了LineRoot）。再看LineIterator代码：
```python
def _stage2(self):
        super(LineIterator, self)._stage2()

        for data in self.datas:
            data._stage2()

        for lineiterators in self._lineiterators.values():
            for lineiterator in lineiterators:
                lineiterator._stage2()
```

1. 首先是遍历所有数据调用_stage2，之后实际又到LineRoot（因为数据类也继承自LineRoot），将所有数据相关Line（"close"等）进入stage2。
2. 然后是将Line迭代器里面的Line（移动平均线、Observer）进入stage2.

这一段，就是将所有U相关的Line进入stage2（_opstage设置为2）。

下面记录数据个数（_dlens）初始化最小周期状态（_minperstatus设置为最大值）。

然后就调用Strategy的start函数，啥也没干。那有啥作用？如果你想在策略运行前进行一些初始处理，可以重写这个函数。

# 儿童阶段
start完成之后，就该prenext了。文章2讲过最小周期，对于Indicator中，需要多个数据进行计算（比如移动平均），这样的话，前面部分数据就无效，因次需要在最小周期之后才正式进行进行数据处理。在此之前，我们称之为成长过程（儿童阶段）。

我们在MyCustomStrategy策略中初始化中，使用了一个周期为5的SimpleMovingAverage。当处理的数据小于5的时候，prenext会被调用。代码中，prenext没有任何操作。我们在实现MyCustomStrategy的时候也没有重写，但是我们还是需要从代码角度看看prenext是如何使用的。

Cerebro代码解读的时候描述过，_runonce的时候会按照针对所有strategies调用_oncepost函数（_runonce的代码请回到文章3再看看，数据的不断后移由该函数推动），如下：
```python
for strat in runstrats:
                strat._oncepost(dt0)
                if self._event_stop:  # stop if requested
                    return
```
而在_runonce代码中，根据最小周期的记录，调用prenext：
```python
   minperstatus = self._getminperstatus()
        if minperstatus < 0:
            self.next()
        elif minperstatus == 0:
            self.nextstart()  # only called for the 1st value
        else:
            self.prenext()
```

- 首先获取最小周期的状态，注意minperstatus为4，因为咱们要计算周期为5的移动平均，前4个数据无效。最小周期的实现还是比较复杂，这里牢牢记住最小周期就是所有line的有效数据起始位的最大值即可。也就是在真正进行逻辑处理之前，所有line的数据必须从有效位开始。而在有效位之前，可以通过prenext进行一些定制处理。
- 从minperstatus为4开始，首先会走到prenext。每调用一次，这个数字就减1.所以prenext会被调用4次。
- 然后第5次，这个时候所有line（包括sma）已经都有数据了，但是这次会调用一次nextstart。nextstart通常也是空操作，有需求可以定义这个函数。
- 从第6次开始，也就是第二个有效数据开始，进入真正的逻辑处理。

还记得文章2介绍最小周期的时候，提供了一个复杂情况最小周期的计算，当时说未经验证，现在经过代码分析，确认数据是正确的。

这里总结下最小周期：

1. 取所有line中最小周期中的最大值。
2. 在未取得有效数据的时候，每一次输入数据会调用prenext。（最小周期-1次）
3. 取得第一个有有效数据，调用一次nextstart。（1次）
4. 然后每一次数据都会调用next。

至此，所有line的数据进入有效阶段，Strategy成熟了，开始进入成年。
# 成年阶段
在所有Line的数据有效之后（prenext之后），下一步就是调用next。在next逐步对所有line的加载的数据进行自定义处理。

在策略的逻辑处理过程中，我们可能需要进行各种操作，也可能会收到各种通知或者发送通知和周边部件协作。下面我们逐一讨论。

# 策略操作
策略中主要涉及买（创建多单）、卖（创建空单）、清仓（或者叫平仓，也就是将所有头寸，不管是多单还是空单，都关闭）和取消未成交委托单（是不是和你在股市上的操作一样？）。对应4个函数：buy，sell，close和cancel。这几个函数就不细讲了，他们的做法就是直接调用broker类（实例）的对应方法完成对应的操作，因此后续在将broker类的时候详解。

sell/buy操作关键参数如下表所示：

| 参数 | 缺省值 | 含义 |
| --- | --- | --- |
| data | None | 指定本次操作归属的data。每个data记录的是每个标的（股票、期货等等）的数据（open/close…），买卖操作就是基于这些数据。在多个资产（或者证券，包括股票期货等等）的情况下，你可能需要针对不同的数据创建委托单。缺省情况就是针对第一个数据（data0）。 |
| size | None | 本单买卖的数量。比如说股票，本次你要买卖多少股。有些地方可能有最小限制，比如国内最小一手100股。这个可以通过addsizer的stake指定。 |
| price | None | 指定价格。这个参数在市价委托单单（Market，通常是下一个开市价格）或者收市委托单（close价格）的时候，不需要设置（也就是None）。因为价格由市场来决定，在Backtrader中使用的开市委托单
对于限价委托（Limit）单、止损委托单（Stop）和止损限价委托单（StopLimit），这个price就是委托单的触发价格。几种单子的情况下文还要详细描述。 |
| plimit | None | 止损限价。这个只有止损限价委托单的有效。因为这种类型的委托单需要两个价格，具体参见下文描述。 |
| exectype | None | 委托单成交类型：
None:这个就是市价委托，在backtrader中，采取的下一个bar的开市（open）价格创建委托单。
Limit:限价委托单。这种在向broker发出买卖某种股票的指令时，对买卖的价格作出限定，对于多单（买），限定一个最高价，只允许broker按其规定的最高价或低于最高价的价格成交，对于空单（卖），限定一个最低价。限价委托的最大特点是，股票的买卖可按照投资人希望的价格或者更好的价格成交，有利于投资人实现预期投资计划。
Stop: 止损委托单。对于多单：低于指定价格卖出，防止亏损扩大。对于空单，高于指定价格卖出。这个价格采用的是市价（也就是下一个开市价open），也成为止损市价委托单。还有一种止盈委托单，和上述策略相反
StopLimit:止损限价委托单，就是以限价委托的止损单。止损限价指令避免了止损指令成交价格不确定的不足，在止损价委托中，投资者要注明两个价格：止损价（对应参数price）和限价（对应参数plimit)，一旦市场价格达到或超过止损价格，止损限价委托自动形成一个限价委托。
国内后两种券商都不支持，据说期货支持，没玩过。不过现在很多券商会提供一些条件单功能，基本上也可以达成相同的效果。因此，我们在做好策略回测之后，对于验证好的策略，可以通过券商的条件单设置自动完成交易。
还有跟踪止损、跟踪止损限价等，委托单的成交方式是策略的重要手段，以后专题研讨。 |
| valid | None | 有效期。有如下取值：
None:无有限期，这种情况下，改委托单一致存在直到委托单满足条件被执行或者被取消。现实中，通常会有时间限制，但是我们这里还是当做无期限。
datetime.datetime 或者datetime.date 实例:也就是指定时间或者日期。也就是订单截止时间。
Order.DAY 或者0 或者 timedelta():也就是指定订单的持续时间。
数值：使用数值指定的截止时间。 |
| tradeid | 0 | 这是一个内部标识。如果多个交易（trade）使用的相同的资产，那么通过整个标识区分不同的交易。在后续通知的处理中，tradeid会返回给Strategy进行区分处理 |
| **kwargs |  | 还要一些broker的实现会支持更多的参数，那么通过**kwargs传递。 |

从这里可以看出，我们在设计一些复杂策略的时候，可以在broker中自定义。具体实现机制我们在broker类中再讲。

# 通知
我们在市场上进行各种操作的时候，也会经常收到不同的通知，Strategy也模拟这些场景，可以针对通知进行特定的处理，这些通知包括：

- notify_order(order)：委托单的通知，当委托单的状态发生改变的话，Strategy就会收到这个通知。状态包括：

1. 提交（Submitted）：委托单发送给broker。
2. 接受（Accepted）：委托单已被broker接受。
3. 部分成交（Partial）：委托单只有部分被执行，比如你要买100股，实际成交50股。
4. 完全成交（Completed）：委托单全部成功执行。
5. 取消（Canceled）：委托单被用户取消。
6. 超时（Expired)：委托单因为超时被取消。
7. 金额不足（Margin)：委托单因为现金金额不足被取消。
8. 拒绝（Rejected）：委托单被broker拒绝。

策略中可以针对不同的状态进行不同的处理，通常我们在成交的时候记录成交的价格、佣金等信息。

- notify_trade(trade) ：交易通知。任何仓位变化都会通知到Strategy。比如开仓、平仓，增仓，减仓等等（这些名词请百度，不一一解释了）。trade有三种状态：created（开仓）、open（开放状态）和close（平仓）。我们可以在平仓之后记录本次交易的盈利情况，如下：
```python
def notify_trade(self, trade):
        if not trade.isclosed:#    如果没有平仓，就返回。
            return
 
        self.log('操作利润, 毛利 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))
```

- notify_fund(cash, value, fundvalue, shares)：资金通知。broker中的现金以及资产信息。每次数据输入的时候均会调用（prenext和next之前）

- notify_cashvalue(self, cash, value)：现金通知。同上，是个子集。

此外，还可以接受来自store（参见Cerebro的store）和数据的通知，可以针对性处理，这里不一一介绍了。

# 繁殖阶段
没有对应的操作。这里指的是优化策略（参见文章1和文章2）的时候，输入参数范围，可以生成（繁殖）多个Strategy实例。

# 死亡
在运行next完成之后，Cerebro通过_stop函数通知Strategy恢复初始设置。在runstrategies函数中：
```python
for strat in runstrats:
                strat._stop()
```
Strategy的_stop函数：
```python
def _stop(self):
    self.stop()

    for analyzer in itertools.chain(self.analyzers, self._slave_analyzers):
        analyzer._stop()

    # change operators back to stage 1 - allows reuse of datas
    self._stage1()
```
首先直接调用stop函数，这个stop函数没有任何操作。如果我们需要在stop做一些定制化的处理，可以在MyCustomStrategy类中重写这个函数。

# Strategy使用方法
## Strategy重写步骤
前面描述了Strategy的运行机制以及各关键过程，通过了解这些关键信息，我们可以通过自定义类来完成自己自己的策略。在文章1有手把手的说明，这里再总结下：

- 首先定义类继承自Strategy并定义相关参数（参数都会转化为属性直接访问，元类中有详细介绍）：
```python
class MyCustomStrategy(bt.Strategy):
    params = (
        ('maperiod', 5),
    )
```

- 在__init__函数中引用需要使用的数据和指标：
```python
def __init__(self):
        # 引用第一个数据源的收盘价（close)
        self.dataclose = self.datas[0].close

        # 增加移动平均指标
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)
```

- 在next根据数据和指标来决定买卖操作
```python
def next(self):
        ...
        if not self.position:

            # 大于均线就买
            if self.dataclose[0] > self.sma[0]:
          
                self.order = self.buy()
         else:

            if self.dataclose[0] < self.sma[0]:
                # 小于均线卖卖卖！
                self.order = self.sell()
```
以上非关键代码省略。

实际上，做到以上3点就已经可以了。当然为了了解更多的信息，可以重写各notify消息以跟踪委托单、交易、资金、持仓等信息。

下面以Backtrader自带的一个双均线策略的例子来看看使用backtrader开发一个策略是多么的简单！

# 双均线策略示例
啥叫双均线策略呢？就是提供两条均线：快速均线（周期短）和慢速均线（周期长），快速均线向上跨越慢速均线买入，快速均线向下跨越慢速均线卖出。
```python
class MA_CrossOver(bt.Strategy):
    '''This is a long-only strategy which operates on a moving average cross

    Note:
      - Although the default

    Buy Logic:
      - No position is open on the data

      - The ``fast`` moving averagecrosses over the ``slow`` strategy to the
        upside.

    Sell Logic:
      - A position exists on the data

      - The ``fast`` moving average crosses over the ``slow`` strategy to the
        downside

    Order Execution Type:
      - Market

    '''
    alias = ('SMA_CrossOver',)

    params = (
        # period for the fast Moving Average
        ('fast', 10),
        # period for the slow moving average
        ('slow', 30),
        # moving average to use
        ('_movav', btind.MovAv.SMA)
    )

    def __init__(self):
        sma_fast = self.p._movav(period=self.p.fast)
        sma_slow = self.p._movav(period=self.p.slow)

        self.buysig = btind.CrossOver(sma_fast, sma_slow)

    def next(self):
        if self.position.size:
            if self.buysig < 0:
                self.sell()

        elif self.buysig > 0:
            self.buy()


```

这里可以看出也是采取三步法：

- 首先继承Strategy，并且定义别名和参数。别名和参数均在元类中进行对应处理。别名和类名可以等同看待和使用。参数定义了快速均线周期和慢速均线周期，同时还定义了要使用的指标类，嗯不错，类也可以做参数。
- 在__init__函数中获取了快速均线和慢速均线，并通过CrossOver类初始化一个购买信号，这个CrossOver在第一个参数（快速均线）向上穿越第二个参数（慢速均线）的时候，设定值为1.当向下穿越的时候，设定值为-1，缺省为0.特别注意的是，快速均线、慢速局向以及购买信号（bugsig）都是Line，包含一组数据。至于CrossOver类，后续我们在Indicator代码解读时再说明。
- 在next函数中，针对bugsig Line，如果当前数据为大于0（也就是快速均线向上穿越慢速均线）买入，反之卖出。

以上只是提供一个均线策略，后续我们专题提供各种策略的实现方式。

# 信号策略类（SigStrategy）
除了普通的Strategy之外，Backtrader还提供一个特殊的信号策略类。信号策略不用重写写Strategy类，直接使用Indicator定义多空信号来触发买卖操作，主要用于一些简单的策略实现。由于这种策略使用的用途不广，另外实现方式比较单一，没有普通Strategy类灵活，后续我们主要用普通Strategy类实现各种策略，因此源代码就不详细解读了，只介绍如何使用。

第一步是定义信号指标。
## 信号指标定义
信号指标也是一种指标，直接承载Indicator。和普通指标不同的是加载方式。普通指标在Strategy中通过addstrategy函数加载，信号指标通过Cerebro的add_signal函数完成。具体请查看文章3 Cerebro代码详解中关于Signal部分。
我们以第9章中双均线策略为例，看看如何改造成SigStrategy。
```python
class MyCrossSignal(bt.Indicator):
    lines = ('MySignal',)
    params = (
        # 快速均线周期
        ('fast', 10),
        # 慢速均线周期
        ('slow', 30),
        # 需要使用的移动平均指标
        ('_movav', btind.MovAv.SMA)
    )
    def __init__(self):
        sma_fast = self.p._movav(period=self.p.fast)
        sma_slow = self.p._movav(period=self.p.slow)
        self.lines.MySignal = btind.CrossOver(sma_fast, sma_slow)
```
代码在9.2节详细描述，关键点在于这里会保存信号指标的值到MySignal Line中，这个Line中信号值只有3种取值：1,0和-1.

在Backtrader中，信号值的含义如下：

1. 大于0：发出多头信号
2. 小于0：发出空头信号
3. 等于0：不发送信号。

在本例中，快速均线向上穿越慢速均线，信号指标值为1，发出多头（long）信号。快速均线向下穿越慢速均线，信号指标值为-1，发出空头（short）信号。
## 信号加载方法
信号定义好之后，通过Cerebro的函数add_signal(sigtype, sigcls, *sigargs, **sigkwargs)加载。

其中第一个参数是信号类型，第二个参数就是对应的信号类（这里是类，不是实例，在Cerebro run过程中实例化）。后面就是传递给信号类的参数。代码示例如下：
```python
cerebro.add_signal(bt.SIGNAL_SHORT, MyCrossSignal,slow=30,fast=10)
```
信号加载之后，Cerebro会完成实例化并根据信号类型决定如何处理。注意，这里示例加上了参数，如果不加，就是使用缺省值。

注意，Cerebro中，信号触发的买卖采用的市价委托。
## 信号类型
在信号类加载的时候，需要指定信号类型来决定如何进行买卖操作（开仓、平仓），在Backtrader中，分为两大类五种类型的信号，包括LONGSHORT，LONG，SHORT、LONGEXIT和SHORTEXIT，对应的代码定义为bt.SIGNAL_LONGSHORT、bt.SIGNAL_LONG、bt.SIGNAL_SHORT、bt.SIGNAL_LONGEXIT和bt.SIGNAL_SHORTEXIT。
## 开仓信号类型
开仓信号类型包含如下：

- LONGSHORT：这种信号类型下，收到多空信号都会开仓，收到多头信号，建立多单。收到空头信号，建立空单。这句话比较难以理解，下面以实例来说明。我们实例中，在2020年11月20日，2021年1月15日，2021年6月4日和2021年9月18日收到多头信号，在2020年12月21日、2021年3月30日、2021年6月21日和9月27日收到空头信号。系统委托单和交易信息如下：
> 2020-11-20, 买单成交 成交价格: 129.86, 成交金额: 129.86, 佣金 0.13
> 2020-12-21, 卖单成交 成交价格: 130.13, 成交金额: 129.86, 佣金 0.13
> 2020-12-21, 卖单成交 成交价格: 130.13, 成交金额: -130.13, 佣金 0.13
> 2020-12-21, 操作收益, 毛利润 0.27, 净利润 0.01
> 2021-01-15, 买单成交 成交价格: 133.89, 成交金额: -130.13, 佣金 0.13
> 2021-01-15, 买单成交 成交价格: 133.89, 成交金额: 133.89, 佣金 0.13
> 2021-01-15, 操作收益, 毛利润 -3.76, 净利润 -4.02
> 2021-03-30, 卖单成交 成交价格: 142.87, 成交金额: 133.89, 佣金 0.14
> 2021-03-30, 卖单成交 成交价格: 142.87, 成交金额: -142.87, 佣金 0.14
> 2021-03-30, 操作收益, 毛利润 8.98, 净利润 8.70
> 2021-06-04, 买单成交 成交价格: 137.51, 成交金额: -142.87, 佣金 0.14
> 2021-06-04, 买单成交 成交价格: 137.51, 成交金额: 137.51, 佣金 0.14
> 2021-06-04, 操作收益, 毛利润 5.36, 净利润 5.08
> 2021-06-21, 卖单成交 成交价格: 134.96, 成交金额: 137.51, 佣金 0.13
> 2021-06-21, 卖单成交 成交价格: 134.96, 成交金额: -134.96, 佣金 0.13
> 2021-06-21, 操作收益, 毛利润 -2.55, 净利润 -2.82
> 2021-09-08, 买单成交 成交价格: 131.62, 成交金额: -134.96, 佣金 0.13
> 2021-09-08, 买单成交 成交价格: 131.62, 成交金额: 131.62, 佣金 0.13
> 2021-09-08, 操作收益, 毛利润 3.34, 净利润 3.07
> 2021-09-27, 卖单成交 成交价格: 127.11, 成交金额: 131.62, 佣金 0.13
> 2021-09-27, 卖单成交 成交价格: 127.11, 成交金额: -127.11, 佣金 0.13
> 2021-09-27, 操作收益, 毛利润 -4.51, 净利润 -4.77



 我们可以看到，11月20日收到多头信号，建立多头仓位。12月21日收到空头信号，首先清掉多头仓位，然后建立空头仓位。同样1月15日收到多头信号，首先平掉空头仓位，然后建立多头仓位。后面都一样。总结就是：不管收到多空信号，都会开仓。开仓之前，如果有持仓，先平仓。

- Long：这种信号类型下，只有收到多头信号才会开仓（多单）。那么什么时候平仓呢？
1. 如果存在LongExit类型的信号实例（后文描述），那么，会使用这个信号实例的空头信号来平仓。
2. 如果存在Short类型的信号实例（后文描述），那么，会使用这个信号实例的空头信号建立空仓之前平掉这个多头仓位。
3. 如果都没有，就使用本信号实例的空头信号平仓。

 同样，实例如下：
> 2020-11-20, 买单成交 成交价格: 129.86, 成交金额: 129.86, 佣金 0.13
> 2020-12-21, 卖单成交 成交价格: 130.13, 成交金额: 129.86, 佣金 0.13
> 2020-12-21, 操作收益, 毛利润 0.27, 净利润 0.01
> 2021-01-15, 买单成交 成交价格: 133.89, 成交金额: 133.89, 佣金 0.13
> 2021-03-30, 卖单成交 成交价格: 142.87, 成交金额: 133.89, 佣金 0.14
> 2021-03-30, 操作收益, 毛利润 8.98, 净利润 8.70
> 2021-06-04, 买单成交 成交价格: 137.51, 成交金额: 137.51, 佣金 0.14
> 2021-06-21, 卖单成交 成交价格: 134.96, 成交金额: 137.51, 佣金 0.13
> 2021-06-21, 操作收益, 毛利润 -2.55, 净利润 -2.82
> 2021-09-08, 买单成交 成交价格: 131.62, 成交金额: 131.62, 佣金 0.13
> 2021-09-27, 卖单成交 成交价格: 127.11, 成交金额: 131.62, 佣金 0.13
> 2021-09-27, 操作收益, 毛利润 -4.51, 净利润 -4.77


可以看出，只有在收到多头信号才开仓，收到空头信号平仓。本例中Short和LongExit并没有定义。

- Short：这种信号类型下，只有收到空头信号才会开仓（空单）。那么什么时候平仓呢？
1. 如果存在ShortExit类型的信号实例（后文描述），那么，会使用这个信号实例的多头信号来平仓。
2. 如果存在Long类型的信号实例（前文描述），那么，会使用这个信号实例的多头信号建立多仓之前平掉这个空头仓位。
3. 如果都没有，就使用本信号实例的多头信号平仓。

 同样，实例如下：
> 2020-12-21, 卖单成交 成交价格: 130.13, 成交金额: -130.13, 佣金 0.13
> 2021-01-15, 买单成交 成交价格: 133.89, 成交金额: -130.13, 佣金 0.13
> 2021-01-15, 操作收益, 毛利润 -3.76, 净利润 -4.02
> 2021-03-30, 卖单成交 成交价格: 142.87, 成交金额: -142.87, 佣金 0.14
> 2021-06-04, 买单成交 成交价格: 137.51, 成交金额: -142.87, 佣金 0.14
> 2021-06-04, 操作收益, 毛利润 5.36, 净利润 5.08
> 2021-06-21, 卖单成交 成交价格: 134.96, 成交金额: -134.96, 佣金 0.13
> 2021-09-08, 买单成交 成交价格: 131.62, 成交金额: -134.96, 佣金 0.13
> 2021-09-08, 操作收益, 毛利润 3.34, 净利润 3.07
> 2021-09-27, 卖单成交 成交价格: 127.11, 成交金额: -127.11, 佣金 0.13



可以看出，只有在收到空头信号才开仓，收到多头信号平仓。本例中Long和ShortExit并没有定义。

以上示例中成交金额好像有点问题，但是利润好像是正确的。请先忽略，专注与开仓平仓操作。

# 平仓信号类型
平仓信号类型只用于平仓，这类信号作为最高优先级的平仓信号（参见前文平仓描述），平仓优先使用这个信号：

- LONGEXIT: 接收空头信号平掉多头仓位。
- SHORTEXIT: 接收多头信号平掉空头仓位。

特别注意的是，以上各种类型的信号实例可以同时存在，信号类也可以不一样，比如LONG使用双均线信号类，LONGEXIT使用单均线策略信号类。单均线策略信号类的代码示例如下：
```python
class MySignal(bt.Indicator):
    lines = ('signal',)
    params = (('period', 30),)

    def __init__(self):
        self.lines.signal = self.data - bt.indicators.SMA(period=self.p.period)
```

就是输入数据的close价格大于移动平均线，为多头信号。小于close移动平均，为空头信号。有人说，这里没有close啊？移动平均也没输入数据啊？这个是简写，具体请参见文章2有详细说明。

如果我们在Cerebro加上两个信号类：
```python
cerebro.add_signal(bt.SIGNAL_LONG, MyCrossSignal,slow=30,fast=10)
cerebro.add_signal(bt.SIGNAL_LONGEXIT, MySignal,period=30)
```

这样的话，收到双均线多头信号建立多头仓位，收到单均线的空头信号平掉这个多头仓位。

# 累积和并发订单的处理
在上述多空信号的情况下，可能会不断发起多个委托单，可能造成：

累积：已在市场（有仓位）的情况下，还会发起加仓委托单。

并发：在已有一个委托单未完成的情况下，还会发起新的委托单。

系统缺省情况下是不允许累积和并发的。如果你想支持这两种情况，可以通过如下命令打开开关。
```python
cerebro.signal_accumulate(True) #也可以设置为False再次关闭
cerebro.signal_concurrency(True)# #也可以设置为False再次关闭
```
# 自定义信号策略类
前面说过，使用信号类（实际上是一种特殊的指标）可以不用重写Strategy类。但是有一个问题，如何了解这个操作的中间过程？或者说前面你的委托单以及交易情况的打印从哪儿来的？我们可以自定义策略类，用来跟踪相关的消息通知，代码实例如下：
```python
class MyCustomSigStrategy(bt.SignalStrategy):


    def log(self, txt, dt=None):
        ''' 策略记录功能'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # 提交和接受委托单不做任何处理
            return

            # 订单成效，记录。
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    '买单成交 成交价格: %.2f, 成交金额: %.2f, 佣金 %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:  # Sell
                self.log('卖单成交 成交价格: %.2f, 成交金额: %.2f, 佣金 %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('委托单取消/金额不足/拒绝')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('操作收益, 毛利润 %.2f, 净利润 %.2f' %
                 (trade.pnl, trade.pnlcomm))


```

这个类继承自SignalStrategy，和普通的strategy类一样，可以通过notify_order/notify_trade接受相关信息（毕竟SignalStrategy也是Strategy的子类，爸爸会的他都会）。next就不用重写了，因为相关操作是通过信号触发的。

# 总结
本文详细介绍了Strategy类的源代码，并提供使用了Strategy的详细方法。同时，还介绍了信号策略类的使用方法和技巧，通过本文，我们应该可以上手进行策略的编码了。

但是，为了编写更复杂的策略，我们需要更详细地了解更多的指标，下一篇文章，我们开始介绍指标类的代码和使用技巧。

