### 周期自适应的区间突破策略
== 
思路概括如下：
（1）核心策略为布林带突破策略，即当收盘价向上突破布林带上轨时做多，当布林带向下突破中轨时平仓
（2）布林带只有一个参数，即时间周期，面对不同的行情应采取不同的周期参数，灵活配置
（3）本策略使用ATR（平均价格真实波动幅度）来衡量行情的波动率，当ATR变大时，使用更大的时间周期，当ATR变小时，反之
（4）设定周期参数的变化范围，即[最小周期， 最大周期]

``` python
import string
import time
import pandas as pd
import math
import numpy as np
from CAL.PyCAL import *
from datetime import datetime

#回测区间
start = "2006-01-01"
end = "2016-03-22"

#手续费
commission = Commission(buycost=0.001, sellcost=0.002)
#初始资金
capital_base = 1000000.0
refresh_rate = 1
#初始周期
init_period = 110
#最大周期
max_period = 100
#最小周期
min_period = 60
#atr周期
atr_period = 19

#设置回测股票池
benchmark = '510050.XSHG'
benchmark_code = '000016.ZICN'
universe = ["510050.XSHG"]

#起算前k个交易日的日期
def get_date_delta_from_current(current_date, delta):
    date = current_date
    cal = Calendar('China.SSE')
    delta = "%dB"%(delta)
    delta_ago_day =  cal.advanceDate(date, delta, BizDayConvention.Following)
    delta_ago_day = datetime(delta_ago_day.year(), delta_ago_day.month(), delta_ago_day.dayOfMonth()).strftime('%Y%m%d')
    return delta_ago_day

#读取大盘信息
def get_market_index(the_index, date_current, period):
    end_day = get_date_delta_from_current(date_current, -1)
    start_day = get_date_delta_from_current(date_current, -period)
    mkt_idx = DataAPI.MktIdxdGet(indexID=the_index, beginDate=start_day, endDate=end_day, field=['openIndex', 'closeIndex', 'highestIndex', 'lowestIndex'], pandas="1")
    mkt_idx = mkt_idx.dropna()
    return mkt_idx

#计算TR
def get_tr(mkt_idx, day_id):
    open_price = (mkt_idx["openIndex"][day_id])
    close_price = (mkt_idx["closeIndex"][day_id])
    high_price = (mkt_idx["highestIndex"][day_id])
    low_price = (mkt_idx["lowestIndex"][day_id])
    pre_close_price = (mkt_idx["closeIndex"][day_id-1])
    TR_pt1 = high_price - low_price
    TR_pt2 = high_price - pre_close_price
    TR_pt3 = pre_close_price - low_price
    TR = max(TR_pt1, TR_pt2, TR_pt3)
    return TR

#计算ATR,指数加权版
def get_eatr(mkt_idx, period, now_day_id):
    alpha = 2.0/(1.0+period)
    atr = get_tr(mkt_idx, now_day_id-2*period+1)
    for day_id in range(now_day_id-2*period+2, now_day_id+1):
        atr = alpha*(get_tr(mkt_idx, day_id)) + (1.0-alpha)*atr
    return atr

#计算布林带
def get_bollinger_bands(mkt_idx, period, std_cof, now_day_id):
    M = mkt_idx["closeIndex"][now_day_id-period+1 : now_day_id+1].mean()
    std = mkt_idx["closeIndex"][now_day_id-period+1 : now_day_id+1].std()
    UB = M + std_cof*std
    LB = M - std_cof*std
    return [UB, M, LB]

#初始化
def initialize(account):
    account.period = init_period
    

#信号
def handle_data(account):
    
    #取数据长度
    lookback_window = max(max_period, min_period, 2*atr_period)+2
    
    #获取量价数据
    hist_prices = {}
    hist_prices["openPrice"] = account.get_attribute_history('openPrice', lookback_window)
    hist_prices["closePrice"] = account.get_attribute_history('closePrice', lookback_window)
    hist_prices["highPrice"] = account.get_attribute_history('highPrice', lookback_window)
    hist_prices["lowPrice"] = account.get_attribute_history('lowPrice', lookback_window)
    hist_turnovers = {}
    hist_turnovers["turnoverVol"] = account.get_attribute_history('turnoverVol', lookback_window)
    hist_turnovers["turnoverValue"] = account.get_attribute_history('turnoverValue', lookback_window)
    
    #读取市场信息
    mkt_idx = get_market_index(benchmark_code, account.current_date, lookback_window)
    
    #计算布林带
    now_day_id = len(mkt_idx["closeIndex"]) - 1
    [UB, M, LB] = get_bollinger_bands(mkt_idx, account.period, 2.0, now_day_id)
    
    if (mkt_idx["closeIndex"][now_day_id] > UB):
        cash = account.referencePortfolioValue
        for stk in account.universe:
            buy_amount_shou = int((cash/len(account.universe))/(account.referencePrice[stk]*100.0))
            order(stk, buy_amount_shou*100)
    
    if (mkt_idx["closeIndex"][now_day_id] < M):
        for stk in account.valid_secpos:
            order_to(stk, 0)
    
    #计算atr
    now_day_atr = get_eatr(mkt_idx, atr_period, now_day_id)
    pre_day_id = now_day_id - 1
    pre_day_atr = get_eatr(mkt_idx, atr_period, pre_day_id)
   
    #调整period
    account.period = account.period * (now_day_atr / pre_day_atr)
    account.period = max(account.period, min_period)
    account.period = min(account.period, max_period)
    account.period = int(account.period+0.5)
    
    ```
    
    
  
