# Hdu_-Quantification
杭电金融量化比赛（风禾杯）

# 资料

往年获奖队伍代码与策略见仓库文件

[官方建议用的量化平台](http://quant.10jqka.com.cn/platform/html/home.html)

[平台帮助文档](http://quant.10jqka.com.cn/platform/html/help-api.html#7/0)

[平台数据文档](http://quant.10jqka.com.cn/platform/html/help-api.html?t=data#3/0)


# 参考策略代码

## kdj


```python

import talib
def initialize(account):      
    g.security = ['601688.SH','000725.SZ','601118.SH','600741.SH','002304.SZ',
                '601006.SH','000898.SZ','600221.SH','000100.SZ','600309.SH',
                '002426.SZ','601607.SH','600637.SH','000625.SZ','000060.SZ',
                '600688.SH','300144.SZ','002252.SZ','600583.SH','601018.SH',
                '600795.SH','601166.SH','601225.SH','600887.SH','000423.SZ',
                '600177.SH','002294.SZ','000876.SZ','600100.SH','600415.SH',
                '601668.SH','600895.SH','601966.SH','600297.SH','600008.SH',
                '300251.SZ','000792.SZ','600089.SH','002572.SZ','600031.SH',
                '600893.SH','000839.SZ','600704.SH','002714.SZ','600485.SH',
                '002065.SZ','600153.SH','600074.SH','601766.SH','600606.SH']  
    # 设置最大持仓股数
    account.n = 10
def kdj(account,data):
    # 获取智能选股列表
    stocks = g.security
    # 若选出的股票数量大于最大持仓股数，则进行交易
    if len(stocks) >= account.n:
        for stk in stocks[:account.n]:
            #获取证券过去20日的收盘价，最高价，最低价数据
            value = data.attribute_history(stk, ['close', 'high', 'low'], 20, '1d', True, None)
            #利用talib库中STOCH函数进行K,D值
            K, D = talib.STOCH(value['high'].values, value['low'].values, value['close'].values, 9, 3, 0, 3, 0)
            if max(K[-1], D[-1]) < 20 and K[-2] < D[-2] and K[-1] > D[-1]:
                # K, D线均在20以内，且K线向上突破D线即金叉，买入
                order_value(stk, account.cash/account.n)
    else:
        pass

    # 若持仓股当前出现卖出信号，则卖出
    for stk2 in list(account.positions):
        #获取证券过去100日的收盘价，最高价，最低价数据
            value = data.attribute_history(stk2, ['close', 'high', 'low'], 100, '1d', True, None)
            #利用talib库中STOCH函数进行K,D值
            K_value, D_value = talib.STOCH(value['high'].values, value['low'].values, value['close'].values, 9, 3, 0, 3, 0)
            if min(K_value[-1], D_value[-1]) > 80 and K_value[-2] > D_value[-2] and K_value[-1] < D_value[-1]:
                order_target(stk2,0) 
#设置买卖条件，每个交易频率（日/分钟/tick）调用一次   
def handle_data(account, data):
    kdj(account,data)

```

## MACD

```python

# MACD策略
# 选出一只股票，若MACD由负变正，则买入；若MACD由正变负，则卖出。

# --- 1. 导入所需库包 ------------------------------------------------------
import talib

# --- 2. 初始化账户 --------------------------------------------------------
#初始化账户   
def initialize(account):   
    account.security = ['601688.SH','000725.SZ','601118.SH','600741.SH','002304.SZ',
                '601006.SH','000898.SZ','600221.SH','000100.SZ','600309.SH',
                '002426.SZ','601607.SH','600637.SH','000625.SZ','000060.SZ',
                '600688.SH','300144.SZ','002252.SZ','600583.SH','601018.SH',
                '600795.SH','601166.SH','601225.SH','600887.SH','000423.SZ',
                '600177.SH','002294.SZ','000876.SZ','600100.SH','600415.SH',
                '601668.SH','600895.SH','601966.SH','600297.SH','600008.SH',
                '300251.SZ','000792.SZ','600089.SH','002572.SZ','600031.SH',
                '600893.SH','000839.SZ','600704.SH','002714.SZ','600485.SH',
                '002065.SZ','600153.SH','600074.SH','601766.SH','600606.SH']   
    
def macd(account,data):
    for i in range(len(account.security[:10])):
        # 获取股票过去40天的行情数据
        values = data.attribute_history(account.security[i], ['close'], 40, '1d', False, None)
        # 计算MACD值
        DIFF, DEA, MACD = talib.MACD(values['close'].values,
                    fastperiod=12, slowperiod=26, signalperiod=9)
        # 将MACD值赋值为全局变量
        account.macd = MACD
        # 若MACD由正转负
        if account.macd[-1] < 0:
            # 卖出所有股票
            order_target(account.security[i], 0)
        
        # 若MACD由负转正
        if account.macd[-1] > 0 and account.macd[-2] < 0:
            order_value(account.security[i],20000)
# --- 4. 盘中设置买卖条件，每个交易频率（日/分钟）调用一次-------------
def handle_data(account, data):
    macd(account,data)

```

## 波动

```python

# 波动率与趋势

# 波动率是用来反映市场波动幅度的大小，大家通常也用来观察市场情绪或预测市场趋势。

# 由于A 股市场做空机制相对欠缺，波动率分布并不对称。故有必要加以区分，我们把波动率区分为上行波动率与下行波动率。对于按其振幅来计算波动率是以开盘价为基准，开盘价以上的波动定义为上行波动率，反之为下行波动率。通常情况下市场处于上涨阶段时上行波动率要大于下行波动率，下跌阶段则反之，同时二者的差值我们定义为波动率剪刀差。

# 因此我们可以基于上面这个逻辑构建针对指数的择时策略，当前一天的上行波动率减去下行波动率的差值趋势（为增加稳定性，采用60 日移动均值）为正时就看多，反之则看空。

import talib as tl
import numpy as np
'''
波动率:
按其振幅来计算波动率是以开盘价为基准,开盘价以上的波动定义为上行波动率,反之为下行波动率。
策略步骤:
1计算相应指数相对强弱RPS；
2计算相应指数上行波动率、下行波动率，并计算二者差值；
3计算当天波动率差值的移动均值；
4观察前一天的趋势(波动率差值的移动均值)，如为正就持有或卖入、否则就空仓或卖出。
'''
def initialize(account):
    #设置设置基准指数为沪深300
    set_benchmark('000300.SH')
    g.security = ['601688.SH','000725.SZ','601118.SH','600741.SH','002304.SZ',
                '601006.SH','000898.SZ','600221.SH','000100.SZ','600309.SH',
                '002426.SZ','601607.SH','600637.SH','000625.SZ','000060.SZ',
                '600688.SH','300144.SZ','002252.SZ','600583.SH','601018.SH',
                '600795.SH','601166.SH','601225.SH','600887.SH','000423.SZ',
                '600177.SH','002294.SZ','000876.SZ','600100.SH','600415.SH',
                '601668.SH','600895.SH','601966.SH','600297.SH','600008.SH',
                '300251.SZ','000792.SZ','600089.SH','002572.SZ','600031.SH',
                '600893.SH','000839.SZ','600704.SH','002714.SZ','600485.SH',
                '002065.SZ','600153.SH','600074.SH','601766.SH','600606.SH'] 
def get_Volatility_avg(stock,n,data):
    '''
    输入：股票,n
    输出: 前一日波动差值移动平均值
    '''
    Count = max([300,n*5])
    value = data.attribute_history(stock, ['open','high','low'], Count, '1d', True, 'pre')
    #清理缺失值
    value = value.dropna()
    #数据太少则跳过
    if len(value)<n*5:
        return 0
    
    value['up_open'] = value.high-value.open
    value['open_low'] = value.open-value.low
    value['Volatility'] = value['up_open']-value['open_low']
    

    Volatility_avg = tl.EMA(value.Volatility.values,n)
    #log.info(Volatility_avg)
    return Volatility_avg[-1]
    
def handle_data(account,data):
    for i in range(len(g.security[:10])):
        stock = g.security[i]
        #采用60 日移动均值
        n=60
        
        Volatility_avg = get_Volatility_avg(stock,n,data)
    
        if Volatility_avg>0:
            order_value(stock,20000)  
        else:
            order_target_percent(stock,0)

```



## 双均线策略

```python

# 双均线策略
# 策略逻辑：当五日均线与二十日均线金叉时买入，当五日均线与二十日均线死叉时卖出。

#初始化账户   
def initialize(account):   
    #设置要交易的证券(600519.SH 贵州茅台)   
    get_iwencai('沪深300','hushen')   
    
#设置买卖条件，每个交易频率（日/分钟/tick）调用一次   
def handle_data(account,data):
    for i in account.hushen[:100]:
        #获取证券过去20日的收盘价数据   
        close = history(i, ['close'], 20, '1d', False, 'pre', is_panel=1)
        #计算五日均线价格   
        MA5 = close.values[-5:].mean()   
        #计算二十日均线价格   
        MA20 = close.values.mean()   
        #如果五日均线大于二十日均线   
        if MA5 > MA20:   
            #使用所有现金买入证券   
            order_value(i,100000)   
            #记录这次买入   
            log.info("买入 %s" % (i))   
        #如果五日均线小于二十日均线，并且目前有头寸   
        if MA5 < MA20 and account.positions_value > 0:   
            #卖出所有证券   
            order_target(i,0)   
            #记录这次卖出   
            log.info("卖出 %s" % (i))
			
```



        
    




# 更多外部数据


[平台数据文档](http://tushare.org/)

