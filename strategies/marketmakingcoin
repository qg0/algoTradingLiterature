
wequant
首发于
wequant
关注专栏
写文章登录
浅谈比特币期货做市策略
wequant
wequant
3 个月前
一、什么是做市策略
做市策略（market-maker strategy）是一种风险中立（risk-neutral）盘口价差套利策略。其基本原理是：在盘口的卖一和买一价之间，插入委买单和委卖单，如果插入的两个单子都成交的话，做市商就吃到了买卖单之间的价差，而整个过程结束后，做市商所持有的头寸并没有变化。如果买卖单之间的价差扣除各种交易手续费之后还有盈余，那么该做市商就获得了相应的盈利。

做市策略是一种增加交易所流动性的策略，一般来讲，成熟的交易市场为了提升自身的流动性，会用低佣金（甚至为做市商提供流动性奖励金）的办法，吸引做市商来该市场做市。

二、做市策略需要注意的事项

1. 做市时机的选择。做市商本质上是整个市场的交易对手方。如果市场呈现急剧的单边行情，做市商下达的买卖委托单会大概率出现单边成交的情况，因此做市商手中就会积累大量的风险头寸，这是做市商不想承担的风险。因此，做市商在选择是否下达做市指令之前，都会预判一下市场的趋势明显程度，如果市场短期内呈现非常明显的趋势信号，做市商就会相应地减少自己的做市单数量（甚至停止做市）

2. 净头寸的处理。做市商手中累计的净头寸，可以通过很多种办法来处理，下面列举其中两种：

（1）在下一次做市时，处理掉累积的净头寸。比如做市商目前净头寸有2BTC，下次做市时，他就可以下达一个卖3BTC的委卖单，一个买1BTC的委买单。这种做法好处是净头寸可以及时得到处理，坏处是净头寸处理的时机（价格）可能不是最优的。

（2）第二种方法是开立另外一个独立的程序，对累积的净头寸进行成本计算，然后按照成本价*（1+一定比例的手续费+一定比例的profit margin)，将该头寸反向甩出市场，甩出方法又有两种：a. 按限价单从价优到价劣依次甩出市场，超时不成交的部分则撤单，等待下次机会；b. 先按限价单从价优到价劣依次甩出市场，超时不成交的部分，则按市价单甩货。第一种方法的好处是甩货成本可控，但是甩货周期可能会拖得比较长；第二种方法则能有效地控制甩货周期，但是成本不可控，孰优孰劣，需要做市商根据自己的风险偏好慎重考虑。

3. 期货换合约（移仓）的处理

期货合约都有一个到期日，在到期日结束时刻的前几个小时，我们不建议做市商继续做市，而是利用这几个小时，将即将到期的期货合约进行移仓。移仓的意思就是平掉当前期货合约的仓位，然后再开同样仓位大小的下周合约。当然，移仓也是需要考虑成本的。如果当前持仓是多头，那么我们希望移仓的时间点是在下周期货贴水最厉害的时候；反之，我们则希望移仓的时间点是在下周期货升水最厉害的时候。等移仓做完以后，做市策略继续恢复执行。

三、一个典型的比特币期货做市策略源码分享

注意，以下的策略需要其他WeQuant基础类库的支持才能运行，这里仅给出策略核心源码，是为了让读者对做市策略本身有一个具体的认识，而不用去太纠结底层下单、统计收益、收发邮件、进程监控等技术细节。关于代码的详细解释，欢迎入QQ群讨论：519538535。

期货做市策略源码：

#!/usr/bin/env python
# -*- coding: utf-8 -*-

from signalGenerator.futureSpotArb import *
from signalGenerator.strategyConfig import changeFutureContractConfig as rollCfg
import time, threading

class FutureMarketMaker(FutureSpotArb):
     def __init__(self, startRunningTime, orderRatio, timeInterval, orderWaitingTime,
                 coinMarketType, open_diff, close_diff, heart_beat_time, depth_data, account_info, transaction_info, maximum_qty_multiplier=None,
                 dailyExitTime=None):
        super(FutureMarketMaker, self).__init__(startRunningTime, orderRatio, timeInterval, orderWaitingTime,
                 coinMarketType, open_diff, close_diff, heart_beat_time, depth_data, account_info, transaction_info, maximum_qty_multiplier=maximum_qty_multiplier,
                 dailyExitTime=dailyExitTime)
                # 显示在邮件中的策略名字
        self.strat_name = "期货做市-%s" % startRunningTime.strftime("%Y%m%d_%H%M%S")
        self.trade_threshold = 0.0003 * 1.01
        self.sell_cut = 0.6
        self.buy_cut = 0.6
        self.leverage = 5
        self.remaining_delta_cash = 0
         # 策略下单参数
        self.coin_type = helper.HUOBI_COIN_TYPE_BTC
        self.contract_type = helper.CONTRACT_TYPE_WEEK
        self.initial_acct_info = None

     # cancel all pending orders
     def cancel_pending_orders(self):
        orders = self.BitVCService.order_list(self.coin_type,self.contract_type)
        while orders is not None and len(componentExtract(orders, "week", [])) > 0:
            orders = componentExtract(orders, "week", [])
            for order in orders:
                if componentExtract(order, u"id", "") != "":
                    order_id = order[u"id"]
                    self.BitVCService.order_cancel(self.coin_type,self.contract_type, order_id)
            orders = self.BitVCService.order_list(self.coin_type,self.contract_type)

     def go(self):
        self.timeLog("日志启动于 %s" % self.getStartRunningTime().strftime(self.TimeFormatForLog))
        self.timeLog("开始cancel pending orders")
        self.cancel_pending_orders()
        self.timeLog("完成cancel pending orders")

        while True:
            # 期货移仓期间，程序一直sleep
            if self.in_time_period(datetime.datetime.now(), rollCfg.CHANGE_CONTRACT_START_WEEK_DAY_FOR_NORMAL,
                                      rollCfg.CHANGE_CONTRACT_END_WEEK_DAY_FOR_NORMAL, rollCfg.CHANGE_CONTRACT_START_TIME_FOR_NORMAL,
                                      rollCfg.CHANGE_CONTRACT_END_TIME_FOR_NORMAL):
                self.timeLog("当前处于移仓时间，程序进入睡眠状态……")
                time.sleep(60)
                continue

            if self.timeInterval > 0:
                self.timeLog("等待 %d 秒进入下一个循环..." % self.timeInterval)
                time.sleep(self.timeInterval)

            self.order_info_list = []

            # 获取账户持仓信息
            try:
                account = copy.deepcopy(self.account_info)
                acct_info = account["account_info"]
                account_update_time = account["time"]
            except Exception:
                self.timeLog("尚未取得账户信息")
                continue

            # 检查账户获取时间
            if account_update_time < self.latest_trade_time:
                self.timeLog("当前账户信息时间晚于最近交易时间，需要重新获取")
                continue

            # setup initial account info
            if self.initial_acct_info is None:
                self.initial_acct_info = acct_info

            short_pos_money_delta = acct_info["bitvc_btc_hold_money_week_short"] - self.initial_acct_info["bitvc_btc_hold_money_week_short"]
            long_pos_money_delta = acct_info["bitvc_btc_hold_money_week_long"] - self.initial_acct_info["bitvc_btc_hold_money_week_long"]
            self.remaining_delta_cash = long_pos_money_delta - short_pos_money_delta  # 代表着增加了多少开多的money，需要减去（sell）
            if self.remaining_delta_cash != 0:
                self.timeLog("剩余 %.4f 数量还没有平" % self.remaining_delta_cash)

            # 查询bitvc深度数据
            try:
                bitvcDepth = copy.deepcopy(self.depth_data)["bitvc"]
            except Exception:
                self.timeLog("尚未取得bitvc深度数据")
                continue

            # 查看行情信息时间戳是否合理
            timestamp_list = [bitvcDepth["time"]]
            if not self.check_time(timestamp_list):
                self.timeLog("获取的行情信息时间延迟过大，被舍弃，进入下一循环")
                continue

            self.timeLog("记录心跳信息...")
            self.heart_beat_time.value = time.time()

            asks = bitvcDepth["asks"]
            bids = bitvcDepth["bids"]
            bitvc_sell_1_price = float(asks[len(asks) - 1][0])
            bitvc_buy_1_price = float(bids[0][0])
            margin = bitvc_sell_1_price - bitvc_buy_1_price

            future_order_sell_price = bitvc_sell_1_price - 0.5*margin*self.sell_cut
            future_order_buy_price = bitvc_buy_1_price + 0.5*margin*self.buy_cut

            future_order_sell_money = 100
            future_order_buy_money = 100

            if self.remaining_delta_cash > 0: #bought too much
                future_order_sell_money += self.remaining_delta_cash
                future_order_sell_price -= 0.2*margin*self.sell_cut
                future_order_buy_price -= 0.1*margin*self.buy_cut

            else:
                future_order_buy_money += abs(self.remaining_delta_cash)
                future_order_buy_price += 0.2*margin*self.buy_cut
                future_order_sell_price += 0.1*margin*self.sell_cut

            diff_percentage = (future_order_sell_price - future_order_buy_price)/future_order_sell_price

            if diff_percentage < self.trade_threshold:
                self.timeLog("future_order_sell_price: %.2f, future_order_buy_price: %.2f, diff percentage: %.6f%% smaller than trade threshold: %.6f%%, so ignore and continue" % ( future_order_sell_price, future_order_buy_price, diff_percentage*100, self.trade_threshold*100))
                continue

            bitvc_btc_hold_money_week_long = acct_info["bitvc_btc_hold_money_week_long"]
            bitvc_btc_hold_money_week_short = acct_info["bitvc_btc_hold_money_week_short"]

            global sold_money
            sold_money = 0
            global bought_money
            bought_money = 0

            # 策略下单参数
            coin_type = self.coin_type
            contract_type = self.contract_type

            def loop1():
                # place sell order
                order_id_list_sell = []
                if bitvc_btc_hold_money_week_long > future_order_sell_money:
                    order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_SELL, future_order_sell_price, future_order_sell_money, leverage=self.leverage))
                else:
                    if bitvc_btc_hold_money_week_long > 0:
                        order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_SELL, future_order_sell_price, bitvc_btc_hold_money_week_long, leverage=self.leverage))
                    if future_order_sell_money-bitvc_btc_hold_money_week_long > 0:
                        order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_OPEN, helper.CONTRACT_TRADE_TYPE_SELL, future_order_sell_price, future_order_sell_money-bitvc_btc_hold_money_week_long, leverage=self.leverage))
                if self.remaining_delta_cash > 0:
                    bitvc_order_query_retry_maximum_times = 100
                    bitvc_order_cancel_query_retry_maximum_times = 10
                else:
                    bitvc_order_query_retry_maximum_times = 100
                    bitvc_order_cancel_query_retry_maximum_times = 10
                global sold_money
                for order_id in order_id_list_sell:
                    if order_id is not None:
                        tmp = self.bitvc_order_wait_and_cancel(coin_type, contract_type, order_id, returnProcessedMoney=True, bitvc_order_query_retry_maximum_times=bitvc_order_query_retry_maximum_times, bitvc_order_cancel_query_retry_maximum_times=bitvc_order_cancel_query_retry_maximum_times)
                        if tmp is not None:
                            sold_money += tmp
                order_id_list_sell = []
                if sold_money < future_order_sell_money and bought_money > 0: # buy side is partially filled or filled
                    adjusted_future_order_sell_price = future_order_buy_price * (1 + 0.0003)
                    adjusted_future_order_sell_money = future_order_sell_money - sold_money
                    if bitvc_btc_hold_money_week_long - sold_money > adjusted_future_order_sell_money:
                        order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_SELL, adjusted_future_order_sell_price, adjusted_future_order_sell_money, leverage=self.leverage))
                    else:
                        if bitvc_btc_hold_money_week_long - sold_money > 0:
                            order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_SELL, adjusted_future_order_sell_price, bitvc_btc_hold_money_week_long - sold_money, leverage=self.leverage))
                        if bitvc_btc_hold_money_week_long - sold_money < 0:
                            #already opened short
                            remaining_short = adjusted_future_order_sell_money
                        else:
                            remaining_short = adjusted_future_order_sell_money - (bitvc_btc_hold_money_week_long - sold_money)
                        if remaining_short > 0:
                            order_id_list_sell.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_OPEN, helper.CONTRACT_TRADE_TYPE_SELL, adjusted_future_order_sell_price, remaining_short, leverage=self.leverage))
                for order_id in order_id_list_sell:
                    if order_id is not None:
                        tmp = self.bitvc_order_wait_and_cancel(coin_type, contract_type, order_id, returnProcessedMoney=True, bitvc_order_query_retry_maximum_times=bitvc_order_query_retry_maximum_times, bitvc_order_cancel_query_retry_maximum_times=bitvc_order_cancel_query_retry_maximum_times)
                        if tmp is not None:
                            sold_money += tmp

            def loop2():
                # place buy order
                order_id_list_buy = []
                if bitvc_btc_hold_money_week_short > future_order_buy_money:
                    order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_BUY, future_order_buy_price, future_order_buy_money, leverage=self.leverage))
                else:
                    if bitvc_btc_hold_money_week_short > 0:
                        order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_BUY, future_order_buy_price, bitvc_btc_hold_money_week_short, leverage=self.leverage))
                    if future_order_buy_money-bitvc_btc_hold_money_week_short > 0:
                        order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_OPEN, helper.CONTRACT_TRADE_TYPE_BUY, future_order_buy_price, future_order_buy_money-bitvc_btc_hold_money_week_short, leverage=self.leverage))
                if self.remaining_delta_cash < 0:
                    bitvc_order_query_retry_maximum_times = 100
                    bitvc_order_cancel_query_retry_maximum_times = 10
                else:
                    bitvc_order_query_retry_maximum_times = 100
                    bitvc_order_cancel_query_retry_maximum_times = 10
                global bought_money
                for order_id in order_id_list_buy:
                    if order_id is not None:
                        tmp = self.bitvc_order_wait_and_cancel(coin_type, contract_type, order_id, returnProcessedMoney=True, bitvc_order_query_retry_maximum_times=bitvc_order_query_retry_maximum_times, bitvc_order_cancel_query_retry_maximum_times=bitvc_order_cancel_query_retry_maximum_times)
                        if tmp is not None:
                            bought_money += tmp
                order_id_list_buy = []
                if bought_money < future_order_buy_money and sold_money > 0: # sell side is partially filled or filled
                    adjusted_future_order_buy_price = future_order_sell_price * (1 - 0.0003)
                    adjusted_future_order_buy_money = future_order_buy_money - bought_money
                    if bitvc_btc_hold_money_week_short - bought_money > adjusted_future_order_buy_money:
                        order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_BUY, adjusted_future_order_buy_price, adjusted_future_order_buy_money, leverage=self.leverage))
                    else:
                        if bitvc_btc_hold_money_week_short - bought_money > 0:
                            order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_CLOSE, helper.CONTRACT_TRADE_TYPE_BUY, adjusted_future_order_buy_price, bitvc_btc_hold_money_week_short - bought_money, leverage=self.leverage))
                        if bitvc_btc_hold_money_week_short - bought_money < 0:
                            # already opened long
                            remaining_long = adjusted_future_order_buy_money
                        else:
                            remaining_long = adjusted_future_order_buy_money - (bitvc_btc_hold_money_week_short - bought_money)
                        if remaining_long > 0:
                            order_id_list_buy.append(self.bitvc_order(coin_type, contract_type, helper.CONTRACT_ORDER_TYPE_OPEN, helper.CONTRACT_TRADE_TYPE_BUY, adjusted_future_order_buy_price, remaining_long, leverage=self.leverage))
                for order_id in order_id_list_buy:
                    if order_id is not None:
                        tmp = self.bitvc_order_wait_and_cancel(coin_type, contract_type, order_id, returnProcessedMoney=True, bitvc_order_query_retry_maximum_times=bitvc_order_query_retry_maximum_times, bitvc_order_cancel_query_retry_maximum_times=bitvc_order_cancel_query_retry_maximum_times)
                        if tmp is not None:
                            bought_money += tmp

            t1 = threading.Thread(target=loop1, name='LoopThread1')
            t2 = threading.Thread(target=loop2, name='LoopThread2')
            t1.start()
            t2.start()
            t1.join()
            t2.join()

            if len(self.order_info_list) > 0:
                transaction_id = helper.getUUID()
                for order_info in self.order_info_list:
                    coinType = self.coinMarketType
                    marketType = order_info["marketType"]
                    order_id = order_info["order_id"]
                    self.put_order_info_in_queue(coinType, marketType, order_id, transaction_id)

            self.cancel_pending_orders()
            self.latest_trade_time = time.time()

期货移仓程序源码：

#!/usr/bin/env python
# -*- coding: utf-8 -*-

from signalGenerator.futureSpotArb import *
from signalGenerator.strategyConfig import changeFutureContractConfig as rollCfg
import time

class ChangeFutureContract(FutureSpotArb):
    def __init__(self, startRunningTime, orderRatio, timeInterval, orderWaitingTime,
                 coinMarketType, open_diff, close_diff, heart_beat_time, depth_data, account_info, transaction_info, maximum_qty_multiplier=None,
                 dailyExitTime=None):
        super(ChangeFutureContract, self).__init__(startRunningTime, orderRatio, timeInterval, orderWaitingTime,
                 coinMarketType, open_diff, close_diff, heart_beat_time, depth_data, account_info, transaction_info, maximum_qty_multiplier=maximum_qty_multiplier,
                 dailyExitTime=dailyExitTime)

        self.strat_name = "合约滚动-%s" % startRunningTime.strftime("%Y%m%d_%H%M%S")

        self.change_contract_diff = rollCfg.CHANGE_CONTRACT_DIFF_1
        self.initial_acct_info = None
        self.coin_type = rollCfg.COIN_TYPE

        self.is_short_contract = None

    # 计算当前的换合约需要满足的价差比例
    def current_change_contract_diff(self, current_time):
        if self.in_time_period(current_time, rollCfg.CHANGE_CONTRACT_START_WEEK_DAY_STAGE_1,
                               rollCfg.CHANGE_CONTRACT_END_WEEK_DAY_STAGE_1, rollCfg.CHANGE_CONTRACT_START_TIME_STAGE_1,
                               rollCfg.CHANGE_CONTRACT_END_TIME_STAGE_1):
            return rollCfg.CHANGE_CONTRACT_DIFF_1
        elif self.in_time_period(current_time, rollCfg.CHANGE_CONTRACT_START_WEEK_DAY_STAGE_2,
                               rollCfg.CHANGE_CONTRACT_END_WEEK_DAY_STAGE_2, rollCfg.CHANGE_CONTRACT_START_TIME_STAGE_2,
                               rollCfg.CHANGE_CONTRACT_END_TIME_STAGE_2):
            return rollCfg.CHANGE_CONTRACT_DIFF_2
        elif self.in_time_period(current_time, rollCfg.CHANGE_CONTRACT_START_WEEK_DAY_STAGE_3,
                               rollCfg.CHANGE_CONTRACT_END_WEEK_DAY_STAGE_3, rollCfg.CHANGE_CONTRACT_START_TIME_STAGE_3,
                               rollCfg.CHANGE_CONTRACT_END_TIME_STAGE_3):
            return rollCfg.CHANGE_CONTRACT_DIFF_3
        elif self.in_time_period(current_time, rollCfg.CHANGE_CONTRACT_START_WEEK_DAY_STAGE_4,
                               rollCfg.CHANGE_CONTRACT_END_WEEK_DAY_STAGE_4, rollCfg.CHANGE_CONTRACT_START_TIME_STAGE_4,
                               rollCfg.CHANGE_CONTRACT_END_TIME_STAGE_4):
            return rollCfg.CHANGE_CONTRACT_DIFF_4
        else:
            return None

    # 计算盘口满足价差的深度数量
    def qty_and_price(self, buy_side_data, sell_side_data, price_diff):
        max_qty = 0
        buy_current_depth = 0
        sell_current_depth = 0
        buy_limit_price = None
        sell_limit_price = None

        buy_price = float(buy_side_data[buy_current_depth][0])
        sell_price = float(sell_side_data[sell_current_depth][0])
        buy_qty = float(buy_side_data[buy_current_depth][1])
        sell_qty = float(sell_side_data[sell_current_depth][1])

        while sell_price - buy_price >= price_diff:
            buy_limit_price = buy_price
            sell_limit_price = sell_price
            # 数量少的一方，深度+1
            if buy_qty > sell_qty:
                max_qty = sell_qty
                sell_current_depth += 1
                sell_qty += float(sell_side_data[sell_current_depth][1])
            else:
                max_qty = buy_qty
                buy_current_depth += 1
                buy_qty += float(buy_side_data[buy_current_depth][1])
            if buy_current_depth >= len(buy_side_data) or sell_current_depth >= len(sell_side_data):
                break
            buy_price = float(buy_side_data[buy_current_depth][0])
            sell_price = float(sell_side_data[sell_current_depth][0])
        self.timeLog("盘口数量为:%s, buy：%s， sell：%s" % (max_qty, buy_limit_price, sell_limit_price))
        return max_qty, buy_limit_price, sell_limit_price

    # cancel all pending orders
    def cancel_pending_orders(self, contract_type):
        orders = self.BitVCService.order_list(self.coin_type, contract_type)
        while orders is not None and len(componentExtract(orders, contract_type, [])) > 0:
            orders = componentExtract(orders, contract_type, [])
            for order in orders:
                if componentExtract(order, u"id", "") != "":
                    order_id = order[u"id"]
                    self.BitVCService.order_cancel(self.coin_type, contract_type, order_id)
            orders = self.BitVCService.order_list(self.coin_type, contract_type)

    def cancel_all_pending_orders(self):
        self.cancel_pending_orders(helper.CONTRACT_TYPE_WEEK)
        self.cancel_pending_orders(helper.CONTRACT_TYPE_NEXT_WEEK)
        self.latest_trade_time = time.time()

    def go(self):
        self.timeLog("日志启动于 %s" % self.getStartRunningTime().strftime(self.TimeFormatForLog))
        self.timeLog("开始cancel pending orders")
        self.cancel_all_pending_orders()
        self.timeLog("完成cancel pending orders")

        while True:
            # 非换期货时间，程序一直sleep
            if not self.in_time_period(datetime.datetime.now(), rollCfg.CHANGE_CONTRACT_START_WEEK_DAY,
                                      rollCfg.CHANGE_CONTRACT_END_WEEK_DAY, rollCfg.CHANGE_CONTRACT_START_TIME,
                                      rollCfg.CHANGE_CONTRACT_END_TIME):
                self.timeLog("当前处于非移仓时间，程序进入睡眠状态……")
                time.sleep(60)
                continue

            if self.timeInterval > 0:
                self.timeLog("等待 %d 秒进入下一个循环..." % self.timeInterval)
                time.sleep(self.timeInterval)

            # 重置部分self级别变量
            self.order_info_list = []
            self.change_contract_diff = self.current_change_contract_diff(datetime.datetime.now())

            # 查询bitvc深度数据
            try:
                bitvc_week_depth = copy.deepcopy(self.depth_data)["bitvc"]
                bitvc_next_week_depth = copy.deepcopy(self.depth_data)["bitvc_next_week"]
            except Exception:
                self.timeLog("尚未取得bitvc深度数据")
                continue
            # 查看行情信息时间戳是否合理
            timestamp_list = [bitvc_week_depth["time"], bitvc_next_week_depth["time"]]
            if not self.check_time(timestamp_list):
                self.timeLog("获取的行情信息时间延迟过大，被舍弃，进入下一循环")
                continue

            bitvc_week_depth["asks"].reverse()
            bitvc_week_sell = bitvc_week_depth["asks"]
            bitvc_next_week_buy = bitvc_next_week_depth["bids"]
            bitvc_week_buy = bitvc_week_depth["bids"]
            bitvc_next_week_depth["asks"].reverse()
            bitvc_next_week_sell = bitvc_next_week_depth["asks"]

            # 本周合约：买入平仓(看卖1)， 下周合约：卖出开仓（看买1）
            bitvc_week_sell_1 = float(bitvc_week_sell[0][0])
            bitvc_next_week_buy_1 = float(bitvc_next_week_buy[0][0])
            bitvc_week_buy_1 = float(bitvc_week_buy[0][0])
            bitvc_next_week_sell_1 = float(bitvc_next_week_sell[0][0])
            market_price = np.mean([bitvc_week_sell_1, bitvc_next_week_buy_1, bitvc_week_buy_1, bitvc_next_week_sell_1])
            price_diff = self.change_contract_diff * market_price

            try:
                account = copy.deepcopy(self.account_info)
                accountInfo = account["account_info"]
                account_update_time = account["time"]
            except Exception:
                self.timeLog("尚未取得账户信息")
                continue
            # 检查账户获取时间
            if account_update_time < self.latest_trade_time:
                self.timeLog("当前账户信息时间晚于最近交易时间，需要重新获取")
                continue

            accountInfo = self.update_bitvc_account_info(accountInfo, market_price)

            self.timeLog("记录心跳信息...")
            self.heart_beat_time.value = time.time()

            self.timeLog("换空头合约价差：%.2f, 换多头合约价差：%.2f。 当前信号价差：%.2f" % (bitvc_next_week_buy_1-bitvc_week_sell_1, bitvc_week_buy_1-bitvc_next_week_sell_1, price_diff))

            # setup initial account info
            if self.initial_acct_info is None:
                self.initial_acct_info = accountInfo
            print(self.initial_acct_info["bitvc_btc_hold_quantity_week_short"])
            # 判断合约的方向
            if self.is_short_contract is None:
                if self.initial_acct_info["bitvc_btc_hold_quantity_week_short"] > 0:
                    self.is_short_contract = True
                elif self.initial_acct_info["bitvc_btc_hold_quantity_week_long"] > 0:
                    self.is_short_contract = False

            # 空头合约，本周买入平仓，下周开空仓
            if self.is_short_contract:
                print("short")
                buy_side = bitvc_week_sell
                sell_side = bitvc_next_week_buy
                week_decreased = self.initial_acct_info["bitvc_btc_hold_quantity_week_short"] - \
                                 accountInfo["bitvc_btc_hold_quantity_week_short"]
                next_week_increased = accountInfo["bitvc_btc_hold_quantity_next_week_short"] - \
                                      self.initial_acct_info["bitvc_btc_hold_quantity_next_week_short"]
                # 本周合约剩余的money，按市场价折算成可成交数量
                week_remaining_qty = accountInfo["bitvc_btc_hold_money_week_short"] / market_price
                week_trade_type = helper.CONTRACT_TRADE_TYPE_BUY
                next_week_trade_type = helper.CONTRACT_TRADE_TYPE_SELL
                week_contract_avg_price = accountInfo["bitvc_btc_hold_price_week_short"]
            else:
                buy_side = bitvc_next_week_sell
                sell_side = bitvc_week_buy
                week_decreased = self.initial_acct_info["bitvc_btc_hold_quantity_week_long"] - \
                                 accountInfo["bitvc_btc_hold_quantity_week_long"]
                next_week_increased = accountInfo["bitvc_btc_hold_quantity_next_week_long"] - \
                                      self.initial_acct_info["bitvc_btc_hold_quantity_next_week_long"]
                # 本周合约剩余的money，按市场价折算成可成交数量
                week_remaining_qty = accountInfo["bitvc_btc_hold_money_week_long"] / market_price
                week_trade_type = helper.CONTRACT_TRADE_TYPE_SELL
                next_week_trade_type = helper.CONTRACT_TRADE_TYPE_BUY
                week_contract_avg_price = accountInfo["bitvc_btc_hold_price_week_long"]

            week_order_type = helper.CONTRACT_ORDER_TYPE_CLOSE
            next_week_order_type = helper.CONTRACT_ORDER_TYPE_OPEN

            max_qty, buy_limit_price, sell_limit_price = self.qty_and_price(buy_side, sell_side, price_diff)
            if max_qty is None:
                continue
            if self.is_short_contract:
                week_order_price = buy_limit_price
                next_week_order_price = sell_limit_price
            else:
                week_order_price = sell_limit_price
                next_week_order_price = buy_limit_price

            # qty_delta > 0, 说明本周合约买入的比下周合约成交的多，下一次挂单，下周合约多成交一些
            qty_delta = week_decreased - next_week_increased
            if week_remaining_qty == 0 and abs(qty_delta) * next_week_order_price < self.bitvc_min_cash_amount:
                continue

            # 最多平掉本周合约的全部
            qty = min(max_qty, week_remaining_qty)

            qty_week = qty
            cash_amount_week = qty_week * week_order_price
            order_id_week = self.bitvc_order(self.coin_type, helper.CONTRACT_TYPE_WEEK,
                                                 week_order_type, week_trade_type, week_order_price,
                                                 cash_amount_week, leverage=self.lever)
            executed_qty_week = self.bitvc_order_wait_and_cancel(self.coin_type, CONTRACT_TYPE_WEEK,
                                                                     order_id_week)
            if executed_qty_week is None:
                executed_qty_week = 0
            if week_contract_avg_price != 0:
                executed_qty_week = executed_qty_week * week_order_price / week_contract_avg_price
            else:
                executed_qty_week = 0
            qty_next_week = min(executed_qty_week + qty_delta,
                                     accountInfo["bitvc_btc_available_margin"] * self.lever)
            cash_amount_next_week = qty_next_week * next_week_order_price
            order_id_next_week = self.bitvc_order(self.coin_type, helper.CONTRACT_TYPE_NEXT_WEEK,
                                                       next_week_order_type, next_week_trade_type,
                                                  next_week_order_price, cash_amount_next_week,
                                                       leverage=self.lever)
            self.bitvc_order_wait_and_cancel(self.coin_type, helper.CONTRACT_TYPE_NEXT_WEEK,
                                             order_id_next_week)
            self.cancel_all_pending_orders()

四、策略实盘运行结果展示

策略在运行了半个小时之后，成功地抓住了两次做市机会，胜率100%！在比特币基准收益为-0.03%的情况下，本策略30分钟收益达到0.12%。

比特币 (Bitcoin)
做市
策略
18
深蓝
函晓淞
Sisyphus
风男子
Eric Deng
收藏
分享
举报
文章被以下专栏收录

wequant
用科学的方法投资比特币
进入专栏
4 条评论
写下你的评论

华亮
华亮
有意思
1 赞
2 个月前
陈鑫
陈鑫
对比多平台对冲呢？
2 个月前
潇瀚
潇瀚
现在有手续费了，玩不了吧
2 个月前
奥利根神棍
奥利根神棍
请问wequant君，比特币破万你赚钱了吗？
4 天前
推荐阅读
题图
浅谈比特币现货做市策略
这篇文章是上一篇文章浅谈比特币期货做市策略的火币现货版本。现货的做市策略，跟期货做市很…查看全文
wequant1 个月前
题图
后监管时代，比特币AR指标策略表现强劲！
最近很多朋友问我，比特币开始收取交易手续费了，还能玩下去吗？先不回答，上图再说：上图还…查看全文
wequant3 个月前
题图
实力、性格、争议、沉浮——记2017LPL春季赛常规赛MVP Doinb
文：十三、魔鬼、丹尼二狗图：网络Doinb的两年沉浮2017年4月16日，LPL春季赛落下帷幕。第二…查看全文
PentaQ刺猬电竞社23 天前编辑精选发表于 PentaQ！刺猬电竞社
题图
一篇关于【怎么做】的随笔
今天想和大家聊聊一个在我心头盘旋了很久的话题：【我该怎么办】。最早对这个问题留下印象，…查看全文
小轻1 个月前编辑精选发表于 咨询师札记

选择语言
