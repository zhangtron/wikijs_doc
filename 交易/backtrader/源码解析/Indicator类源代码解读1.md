前面的3篇文章我们分别介绍了Cerebro、Data相关类、Strategy类，基本上Backtrader的框架已经具备，从这篇文章开始我们介绍Backtrader的重要部件，从最重要额Indicator开始。

老规矩，从家谱图开始。

# 家谱图
![backtrade指标家谱图.png](https://cdn.nlark.com/yuque/0/2022/png/29177964/1669218926219-a2fa641c-7002-4af1-b00d-f46e43f0ecfb.png#averageHue=%23fdfdfd&clientId=u3bfcbf8a-5199-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=2816&id=uedef2987&margin=%5Bobject%20Object%5D&name=backtrade%E6%8C%87%E6%A0%87%E5%AE%B6%E8%B0%B1%E5%9B%BE.png&originHeight=2816&originWidth=1556&originalType=binary&ratio=1&rotation=0&showTitle=false&size=396759&status=done&style=none&taskId=u9724fc66-259f-4a47-b7d0-47c4a3e91ee&title=&width=1556)

是不是很眼熟，对了，和Strategy/Data继承关系基本就是一样的，所以从面向对象来说，他和Strategy/Data基本上就是亲兄弟。在文章2中就明确说了，在Backtrader中，一切都是数据。

另外，看了前面几篇文章，对Backtrader的元类套路很熟悉了吧，后面就不细讲了，有兴趣的话可以对照Strategy来读代码（元类的继承关系以及代码是一样的），后面只讲特殊的处理。
# Indicator的实例化和初始化
大家还记得Indicator什么时候实例化的？在Strategy初始化的时候，请参见咱们Strategy代码解读中的MyCustomStrategy的__init__函数：
```python
# 增加MovingAverageSimple指标
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod)
```

然后又是元类MetaBase的一整套动作（doprenew、donew、dopreinit、doinit和dopostinit），具体对照Strategy的代码解读对照，说明下大概：

- 在MetaLineSeries的donew中，初始化一个LineBuffer存储在lines里面，可以看出，Indicator本质上上也是数据源，拥有Lines（家谱图中也可以看到）。初始化的时候只有一个LineBuffer.

- 在MetaLineIterator的donew中，会将参数中输入的数据（例子中的self.datas[0]）加入datas容器（既然是复数，那么可以输入多个数据，如果有的指标需要多个数据的话）中，这个数据是后续计算Indicator的基本数据。如果参数不输入数据咋办呢？就会将这个Indicator的owner的缺省data加入。Indicator的owner是谁？是MyCustomStrategy，因为是它初始化了这个Indicator。那么MyCustomStrategy的缺省data，也就是是MyCustomStrategy的第一个数据（self.datas[0]）。那MyCustomStrategy的第一个数据是谁？就是Cerebro加入的第一个数据。所以不输入这个参数数据也可以（文章2有说明，代码原理在这里）。

以上实例化完成，就开始初始化了__init__了，首先就进入到MovingAverageSimple的__init__了：
```python
def __init__(self):
        # Before super to ensure mixins (right-hand side in subclassing)
        # can see the assignment operation and operate on the line
        self.lines[0] = Average(self.data, period=self.p.period)

        super(MovingAverageSimple, self).__init__()
```

1. 首先就是在实例化一个Average，各种均线指标是非常重要也使用广泛的指标。Average是非常基本的一个类，提供的是简单的算术平均结果，这个记住如何使用就行了，具体代码就不看了。这里记住，第一个line就是Average实例化的时候返回的一个LineBuffer实例，也就是具体的数据在Average中计算。可以看出，数据的计算是在初始化的时候。
2. 调用父类的__init__，没做啥。
- 下面就是dopostinit了，在MetaLineIterator的dopostinit中，将自己（初始化后的SimpleMovingAverage实例）传递给自己的owner（MyCustomStrategy），MyCustomStrategy就可以使用这个Indicator了。

至此初始化完成。

# Indicator的使用
## Indicator的代码流程
前面一再强调，Indicator逻辑（继承关系）上和Data是一样的，所以它的使用基本和Data类是一样的。区别在于：Data的数据通过从元素数据（比如PandaData中 获取），而Indicator的数据来自于计算（比如本例中计算的移动平均值）。有了数据之后，我们循着它代码来看，这些数据是如何使用的。

在Cerebro以及Strategy的代码解读中，我们可以看出Indicator的使用轨迹，从Cerebro的runstrategies->_runonce–>Strategy的_oncepost，代码：
```python
def _oncepost(self, dt):
        for indicator in self._lineiterators[LineIterator.IndType]:
            if len(indicator._clock) > len(indicator):
                indicator.advance()

```
首先要调用Indicator的Advance：
```python
def advance(self, size=1):
        # Need intercepting this call to support datas with
        # different lengths (timeframes)
        if len(self) < len(self._clock):
            self.lines.advance(size=size)

```
可以看出，直接就到Indicator所有line的advance了，line（LineBuffer）的advance干啥了，PandasData类详解的时候说明过：所有line索引idx后移，长度计数增加。这里只改索引，不涉及底层数据。也就是当前数据移到下一个了。这个数据不断后移，移到大于最小周期之后，就进入MyCustomStrategy的next了，在这个next中，可以直接对Indicator的指标数据应用进行处理：
```python
def next(self):
        # 记录下本次的close和sma值
        self.log('Close:%.3f' % self.data.close[0])
        self.log('sma:%.3f' % self.sma[0])
   
        # 检查是否还有订单未处理，有的话等等
        if self.order:
            return
        #Check if we are in the market
        if not self.position:
            # 大于均线就买
            if self.dataclose[0] > self.sma[0]:
                # BUY, BUY, BUY!!! (with all possible default parameters)
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

在next中，就可以直接使用sma line的值。0索引对应当前值，如果要访问历史数据或者未来数据这么办，还有如何获取批量数据（切片），请参见文章2.

总之，Indicator的数据来自于计算，这些数据和Data类的line一样使用。

# Indicator的使用经验
在Backtrader中，Indicator在两个地方使用：

1. 在Strategy中，这前面代码有明确描述，MyCustomStrategy中有一个指标SimpleMovingAverage。
2. 在其他Indicator中，这个后面讲解具体指标的时候说明。不过其实本例中也有，SimpleMovingAverage指标就有另外一个Average（它也是indicator）。

根据前述代码分析可知：

1. Indicator通常在Strategy的__init__函数中实例化，实例化的时候会预先计算指标值。
2. Indicator的值（或者基于Indicator进行计算所得结果）在Strategy的next中使用。

在__init__函数中，可以对line（不管是Data的还是Indicator的Line）进行各种操作（有哪些操作参见文章2），其结果也是Line。如下代码所示：
```python
hilo_diff = self.data.high - self.data.low
```

high和low是数据的两个Line（LineBuffer），两者进行相减的操作，返回的是LineBuffer.LinesOperatiron对象，保存的是操作的结果。这个LinesOperation继承了LineBuffer，所以实际上也是一种Line（Linebuffer），可以和Line一样处理，操作计算的值在next中可以直接使用。使用的方式也是采用符号[],例如访问当前值的话，可以使用hilo_diff[0]。这个代码的含义也很明确，hilo_diff中每一数据（bar）是对应的high和low的差值。

更复杂的，我们也可以对数据和Indicator混合操作（毕竟他们本质就是Line）：
```python
sma = bt.SimpleMovingAverage(self.data.close)
close_sma_diff = self.data.close - sma
```

close_sma_diff含义就是close和均值的差值。

而且，我们也可以使用逻辑操作：
```python
close_over_sma = self.data.close > sma

```
close_over_sma也是一个Line，保存的是一组布尔值。

如果将上述逻辑操作放到next中，会发生什么呢？比如：
```python
def next(self):
        close_over_sma = self.data.close > self.sma
```

这回返回的是什么？这次就不是返回Line了，而是一个bool值：

这个值是如何获取的，实际上返回的是close的当前值（self.data.close[0])和sma的当前值（self.sma[0])进行逻辑大于判决后得出的布尔值，就是等于如下代码：
```python
close_over_sma = self.data.close[0] > self.sma[0]
```

既然__init__和next都可以进行数据计算，那到底放到哪儿合适？

经验就是怎么简单怎么来，一切都是为了易于使用。一般来讲，在__init__函数中，完成数据（包括Indicator）的计算和操作，也就是准备好数据。而在next函数中，完成数据的使用，集中于逻辑的处理，而不是数据的繁琐计算。这样做实际还有一个好处，速度更快。因为在数据都在初始化的时候准备好（批量处理），而不是在next的时候逐一进行计算。

如下是一个完整的例子，说明了上述处理原则：
```python
class MyStrategy(bt.Strategy):

    def __init__(self):

        sma1 = btind.SimpleMovingAverage(self.data)
        ema1 = btind.ExponentialMovingAverage()

        close_over_sma = self.data.close > sma1
        close_over_ema = self.data.close > ema1
        sma_ema_diff = sma1 - ema1

        buy_sig = bt.And(close_over_sma, close_over_ema, sma_ema_diff > 0)

    def next(self):

        if buy_sig:
            self.buy()
```

可以看出，__init__中进行各种数据的批量处理。而在next中，直接使用数据结果进行策略操作。在__init__的处理极大简化了next中的数据处理，而且这样做的最大好处是就是速度更快。

# Indicator绘图的控制方法
将Indicator的数据在生成的图片上展示，有利于对策略执行的可视化理解。因此，我们这里介绍下控制Indicator汇图的方法。

首先明确两点：

1. 声明的Indicator缺省会画图（当然Cerebro必须调用plot），例如我们示例中这样定义：
```python
        sma1 = btind.SimpleMovingAverage(self.data)
        ema1 = btind.ExponentialMovingAverage()
```

2. 对于通过操作符生成的Line缺省不画图，比如实例中的如下定义不会画图。
```python
        close_over_sma = self.data.close > sma1
        close_over_ema = self.data.close > ema1
        sma_ema_diff = sma1 - ema1

        buy_sig = bt.And(close_over_sma, close_over_ema, sma_ema_diff > 0)
```
但是如果你必须要画，可以通过LinePlotterIndicator类来实现：
```python
close_over_sma = self.data.close > self.sma
LinePlotterIndicator(close_over_sma, name='Close_over_SMA')
```

参数name指示图中该line的名称。

在开发Indicator的时候，可以声明一个plotinfo用来控制话务的行为，这个plotinfo可以是元组的元组（两两成对）或者dic/OrderedDict,如下所示：
```python
class MyIndicator(bt.Indicator):

    ....
    plotinfo = dict(subplot=False)
    ....
```
定义的这个参数可以在使用的地方通过如下方法设置：
```python
myind = MyIndicator(self.data, someparam=value)
myind.plotinfo.subplot = True
```

也可以在初始化的时候设置：
```python
myind = MyIndicator(self.data, someparams=value, subplot=True)
```

两种方法都可以。

下面分别说明plotinfo支持的参数，以及这些参数如何影响绘图的行为：

| 参数 | 缺省值 | 含义 |
| --- | --- | --- |
| plot | True | 是否在图中展示 |
| subplot | True | 是否在独立的窗口（和数据图不在一个窗口）展示。对于均线类的Indicator缺省值为False，是为了和数据展示在相同的窗口以方便对比效果。 |
| plotname | "" | 指定在图中展示时的名称以方便理解。缺省情况下会显示Indicator类的名称。类名称由于编码限制，可读性不强。大家在定义Indicator的时候，可以按如下方法定义图中展示的名称：
class MyIndicator(bt.Indicator): plotinfo=dict(plotname='My Indicator') |
| plotabove | Fales | 通常情况下，在独立窗口展示的Indicator（subplot设置为True），通常在Data展示窗口的下方。如果本参数设为为True，那么这个独立窗口位于Data上面。 |
| plotlinelabels | False | 对于基于Indicator进行计算操作得出的另外一个Indicator。比如时候我先计算一个RSI，在针对这个RSI计算移动平均（SimpleMovingAverage）。那么在图中展示的名称就是SimpleMovingAverage，这样的话，你可以看到的是基于RSI画出的一条移动平均线。如果设置为True，那么SimpleMovingAverage这个类中定义的line名称（例如sma）也会显示在图中。 |
| plotymargin | 0.0 | 设定图形距离边界的空白。有时候图形离开画布的顶部或者底部太远，可以通过这个参数来调整。0.15的意思是预留15%的空白 |
| plotyticks | [] | 用来指示Y轴的刻度，通常由系统自动计算。 |
| plothlines | [] | 用来指示显示水平线。比如输入[-0.5,0.5]，会显示两条水平线（值为0.5和-0.5），通常用于显示一些特殊值作为参考。 |
| plotyhlines | [] | 用于同时指示plotyticks 和plothlines。 |
| plotforce | False | 强制绘图，如果由于各种原因没能显示出来，那么终极大招就是设置这个参数为True |

# 自定义Indicator的开发
虽然Backtrader提供了很多Indicator，但是有可能满足不了所有人的要求，如果你想要开发自己的指标（自定义Indicator），Backtrader中也很容易做到。
## 开发步骤
开发步骤如下：

- 定义一个自定义的类，继承自Indicator，或者任何已有Indicator的子类。
- 定义这个Indicator拥有的Lines。每一个Indicator至少需要有一个Line。注意，如果继承了其它Indicator，那么就会继承父类定义的lines。
- 定义参数（例如移动平均先的周期），这个是可选的。
- 定义画图的参数以控制Indicator的画图方法（参见3.3节）。
- 定义__init__函数，其中给各个Line赋值或者定义next/once（可选）函数。如果lines的值可以在初始化的时候完全确定，就直接赋值。如果不行的话，需要提供next函数，给当前值（索引为0）赋值。这个next被Strategy调用，也就是每次在next中确定具体值。如果进行性能优化的处理（还记得runonce函数？）那么可以定义once函数（提供批量数据处理）。通常我们用__init__函数就够用了。实际上一些经常被应用的底层逻辑处理会使用once处理。

特别需要注意的是：

Indicator为输入的每一个数据（bar）产生一个输出，它不管输入数据是啥，也不管输入多少次。也就是这个操作满足幂等性，就是说无论多少次操作，只要输入的数据一致，输出就不会变。

## 一个简单的示例
一个简单的例子说明如何定义Indicator：
```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def __init__(self):
        self.lines.dummyline = bt.Max(0.0, self.params.value)
```

这个Indicator的要点：

1. 声明一个lines，包含一个line:dummyline.
2. 设定一个参数value，缺省为5.
3. 初始化的时候设定具体的值，这个值只会是0，或者等于参数（如果参数大于0的话）。

不要以为dummyline是一个值，它实际上是一个line，所以要使用backtrader重写的Max函数。这个line里面值是固定的，在图形上显示就是一条直线。

同样的，我们也可以通过next函数来实现：
```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def next(self):
        self.lines.dummyline[0] = max(0.0, self.params.value)
```

这里要注意，在next中，一次就是提供一个值。由于这里只是比较一个值，直接使用python内置的max函数即可。如果用once呢？这个可以进行批量赋值，参见如下示例。这样处理更要效率，速度更快。
```python
class DummyInd(bt.Indicator):
    lines = ('dummyline',)

    params = (('value', 5),)

    def once(self,start,end):
        for i in range(start, end):
            self.lines.dummyline[i] = max(0.0, self.params.value)
```

通过这个例子，可以理解之前文章描写的runonce的作用了。

一般来说，__init__函数的实现方式是最好的，一切都在初始化的时候准备好了。next和once函数会自动提供，无需进行索引以及操作的处理。

# 最小周期的计算（手动和自动）
最小周期通常由系统自动计算，但是某些特殊情况下（你可能永远都用不到），可能系统无法识别，比如下面这种情况：
```python
class SimpleMovingAverage1(Indicator):
    lines = ('sma',)
    params = (('period', 20),)

    def next(self):
        datasum = math.fsum(self.data.get(size=self.p.period))
        self.lines.sma[0] = datasum / self.p.period
```

这个系统就没法计算最小周期，虽然有个参数叫period。那么可能在第一个输入数据的时候就进入next了，整个就错位了，计算就完全不对。这个就需要通过addminperiod函数显示增加最小周期，如下：
```python
class SimpleMovingAverage1(Indicator):
    lines = ('sma',)
    params = (('period', 20),)

    def __init__(self):
        self.addminperiod(self.params.period)

    def next(self):
        datasum = math.fsum(self.data.get(size=self.p.period))
        self.lines.sma[0] = datasum / self.p.period
```

addminperiod函数通知系统考虑额外的周期。通常这种情况完全可以避免，如果所有的计算都是基于已经传递过最小周期的对象完成。比如如下一个复杂的MACD指标：
```python
from backtrader.indicators import EMA

class MACD(Indicator):
    lines = ('macd', 'signal', 'histo',)
    params = (('period_me1', 12), ('period_me2', 26), ('period_signal', 9),)

    def __init__(self):
        me1 = EMA(self.data, period=self.p.period_me1)
        me2 = EMA(self.data, period=self.p.period_me2)
        self.l.macd = me1 - me2
        self.l.signal = EMA(self.l.macd, period=self.p.period_signal)
        self.l.histo = self.l.macd - self.l.signal

```

这种情况下：

- EMA表示的的是Exponential Moving Average （指数移动平均，ema是内置的别名），该对象声明了其小的最小周期。
- macd考虑了me1和me2的最小周期，取最大值。
- sigal再次在macd的基础上增加ems的最小周期。
- histo 再次考虑macd和signal的最小周期的最大值。

可以看出，这么复杂的最小周期都可以自动计算，要诀是保证使用可以自动计算的Indicator对象。

# 一个完整的示例
下面提供一个完整的示例，包括对画图的控制，画图相关参数参见3.3节。
```python
import backtrader as bt
import backtrader.indicators as btind

class OverUnderMovAv(bt.Indicator):
    lines = ('overunder',)#增加一个overunder的Line
    params = dict(period=20, movav=bt.ind.MovAv.Simple)#引用简单移动平均线的类。

    plotinfo = dict(
        # 增加到画布顶部和底部预留空间（0.15意思是15%）
        plotymargin=0.15,

        # 针对1.0和-1.0显示两条水平线。
        plothlines=[1.0, -1.0],

        # 设置Y走的刻度在1.0和-1.0之间，也可以添加中间值。
        plotyticks=[1.0, -1.0])

    # 指示对于overunder line线的类型为虚线。
    # 注意，这里参数直接传递给matplotlib控制线的属性，matplotlib支持的参数这里都支持。
    
    plotlines = dict(overunder=dict(ls='--'))

    def _plotlabel(self):
        # 这个方法返回一个标签列表，该标签将显示在绘图上的Indicator名称之后

        # 缺省参数周期一定要显示
        plabels = [self.p.period]

        # 非缺省的均线指标都要显示
        plabels += [self.p.movav] * self.p.notdefault('movav')

        return plabels

    def __init__(self):
        movav = self.p.movav(self.data, period=self.p.period)
        self.l.overunder = bt.Cmp(movav, self.data)

```

# 不同时间粒度的混合处理
Indicator计算的时候，有时候数据具有不同的时间粒度，比如有的是一天，有的是一个月，这个时候就需要用到耦合功能了。

比如我们有两个数据，data0时间粒度是天，data1时间粒度是月，我们要计算这样一个指标：
```python
pivotpoint = btind.PivotPoint(self.data1)
sellsignal = self.data0.close < pivotpoint.s1
```

这样直接操作的话会失败。具体原因就不详述了，这里提供具体的解决方法（文章2中也有相关描述）
```python
pivotpoint = btind.PivotPoint(self.data1)
sellsignal = self.data0.close < pivotpoint.s1()
```

Backtrader提供()操作符返回一个内部LinesCoupler对象，支持更大时间粒度，这个LinesCoupler对象会将填充S1的最新值，保证和时间粒度小的数据一致。为了开启这个功能，需要关掉Cerebro的runonce开关。

下面提供一个完整的示例：
```python
import backtrader as bt
import backtrader.feeds as btfeeds
import backtrader.indicators as btind
import backtrader.utils.flushfile


class St(bt.Strategy):
    params = dict(multi=True)

    def __init__(self):
        self.pp = pp = btind.PivotPoint(self.data1)
        pp.plotinfo.plot = False  # deactivate plotting

        if self.p.multi:
            pp1 = pp()  # couple the entire indicators
            self.sellsignal = self.data0.close < pp1.s1
        else:
            self.sellsignal = self.data0.close < pp.s1()

    def next(self):
        txt = ','.join(
            ['%04d' % len(self),
             '%04d' % len(self.data0),
             '%04d' % len(self.data1),
             self.data.datetime.date(0).isoformat(),
             '%.2f' % self.data0.close[0],
             '%.2f' % self.pp.s1[0],
             '%.2f' % self.sellsignal[0]])

        print(txt)
if __name__ == '__main__':
    cerebro = bt.Cerebro()
  
    cerebro.addstrategy(St)
   
    cerebro.broker.setcash(1000000.0)
    cerebro.broker.setcommission(commission=0.001)#设定交易费用（买卖都收）
    cerebro.addsizer(bt.sizers.FixedSize, stake=100)
    stock_hfq_df = pd.read_csv("../data/sz399006创业板指.csv",index_col='date',parse_dates=True)
    
    start_date = datetime(2021, 6,1 )  # 回测开始时间
    end_date = datetime(2021, 9, 30)  # 回测结束时间
    data=MyCustomdata(dataname=stock_hfq_df, fromdate=start_date,todate=end_date)
        cerebro.adddata(data)  # 将数据传入回测系统
    cerebro.resampledata(data,timeframe=bt.TimeFrame.Months)#resample一个更大粒度的数据。
    cerebro.run(runonce=False)
    cerebro.plot()

```
# 未完待续
由于我低估了整理内置indicator的工作量，考虑到csdn篇文章不宜过大，因此indicator分为两篇，先贴第一篇。

