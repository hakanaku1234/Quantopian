# Implementation of Quantitative Trading Algorithm on Quantopian

## Overview

### What is 'Quantitative Trading'

Quantitative trading consists of trading strategies based on quantitative analysis, which rely on mathematical computations and number crunching to identify trading opportunities. As quantitative trading is generally used by financial institutions and hedge funds, the transactions are usually large in size and may involve the purchase and sale of hundreds of thousands of shares and other securities. However, quantitative trading is becoming more commonly used by individual investors.

Quantitative traders take advantage of modern technology, mathematics and the availability of comprehensive databases for making rational trading decisions.

Quantitative traders take a trading technique and create a model of it using mathematics, and then they develop a computer program that applies the model to historical market data. The model is then backtested and optimized. If favorable results are achieved, the system is then implemented in real-time markets with real capital.

The way quantitative trading models function can best be described using an analogy. Consider a weather report in which the meteorologist forecasts a 90% chance of rain while the sun is shining. The meteorologist derives this counterintuitive conclusion by collecting and analyzing climate data from sensors throughout the area. A computerized quantitative analysis reveals specific patterns in the data. When these patterns are compared to the same patterns revealed in historical climate data (backtesting), and 90 out of 100 times the result is rain, then the meteorologist can draw the conclusion with confidence, hence the 90% forecast. Quantitative traders apply this same process to financial market to make trading decisions.

### About Quantopian Platform

[Quantopian](https://www.quantopian.com/) is a SaaS algorithmic trading platform for retail investors to hack up their own strategies, backtest it and let it run on either paper trading or real money mode (through interactive brokers).  It is exclusively within the US equity universe (as of now) with plenty of features but also shortcomings.  Quantopian is still early in its roll out phase so we can expect more to come within the near future.  The typical process for the trader is as follows:

* User comes up with a trade idea
* Type up the code in Quantopian
* Set-up trade parameters including fees/commission/slippage and datafeed granularity
* Backtest over a variety of time periods and scenarios
* Paper trade, real trade or enter Q open (a contest to see who has an amazing algo)

## An Simple Example of Trend Trading Algorithm

Trend trading means when the stock goes up quickly, we're going to buy; when it goes down we're going to sell.  Hopefully we'll ride the waves.

[![20period_simple_moving_average_buy_sell.jpg](https://s27.postimg.org/8kr18yzo3/20period_simple_moving_average_buy_sell.jpg)](https://postimg.org/image/oizqz3tvz/)

The algorithm we are going to use here is the `Fast Fourier Transform`. 

A [fast Fourier transform (FFT)](https://en.wikipedia.org/wiki/Fast_Fourier_transform) algorithm computes the discrete Fourier transform (DFT) of a sequence, or its inverse. Fourier analysis converts a signal from its original domain (often time or space) to a representation in the frequency domain and vice versa. An FFT rapidly computes such transformations by factorizing the DFT matrix into a product of sparse (mostly zero) factors. As a result, it manages to reduce the complexity of computing the DFT from O(n^2), which arises if one simply applies the definition of DFT, to O(n * log n), where n is the data size.

To implement the algorithm on Quantopian, first step is to import Python package. 

```
import numpy
from scipy.fftpack import fft
from scipy.fftpack import ifft
```

To run an algorithm in Quantopian, you need two functions: initialize and handle_data.

The initialize function sets any data or variables that you'll use in your algorithm. For instance, you'll want to define the security (or securities) you want to backtest.  You'll also want to define any parameters or values you're going to use later. It's only called once at the beginning of your algorithm.

```
def initialize(context):
    context.security = sid(24) # Apple stock
```
The handle_data function is where the real work is done.  This function is run either every minute (in live trading and minutely backtesting mode) or every day (in daily backtesting mode).

```
def handle_data(context, data):
```
    
To make market decisions, we will need to know the stock's average price for the last 5 days, the stock's current price, and the cash available in our portfolio.

```
    N = 20
    prices = numpy.asarray(history(N, '1d', 'price')) 
    
    #print numpy.transpose(prices)[0]
        
    y = fft(numpy.transpose(prices)[0])
    y[3:7] = 0
  
    #y1=y; 
    #for i in range(3,N):
    #    y1[i] = 0
    x1rec = ifft(y).real
   
    current_rec = x1rec[-1]
    average_rec = numpy.mean(x1rec)
   
    current_price = data[context.security].price
    cash = context.portfolio.cash
```
   
Here is the meat of our algorithm. If the current price is 1% above the 5-day average price and we have enough cash, then we will order. If the current price is below the average price, then we want to close our position to 0 shares.

```   
    print x1rec
    # print current_rec,average_rec
    if current_rec > 1.01*average_rec and cash > current_price:
        
        # Need to know how many shares we can buy
        number_of_shares = int(cash/current_price)
        
        # Place the buy order (positive means buy, negative means sell)
        order(context.security, +number_of_shares)
        
    elif current_rec < average_rec:
        
        # Sell all of our shares by setting the target position to zero
        order_target(context.security, 0)
    
    # Plot the stock's price
    record(stock_price=data[context.security].price)
```

## Implementation of Pairs Trading Strategy

### What is Pairs Trading?

[![pair-trade-example.jpg](https://s11.postimg.org/vqf2pb14z/pair_trade_example.jpg)](https://postimg.org/image/9r8o23kan/)

[Pairs Trading](http://www.investopedia.com/university/guide-pairs-trading/) is a market-neutral trading strategy that matches a long position with a short position in a pair of highly correlated instruments such as two stocks, exchange-traded funds (ETFs), currencies, commodities or options. Pairs traders wait for weakness in the correlation, and then go long on the under-performer while simultaneously going short on the over-performer, closing the positions as the relationship returns to its statistical norm. The strategy’s profit is derived from the difference in price change between the two instruments, rather than from the direction in which each moves. Therefore, a profit can be realized if the long position goes up more than the short, or the short position goes down more than the long (in a perfect situation, the long position will rise and the short position will fall, but this is not a requirement for making a profit). It is possible for pairs traders to profit during a variety of market conditions, including periods when the market goes up, down or sideways, and during periods of either low or high volatility.

### An Example of Pairs Trading of SPY and ETFs

Here we use Pairs Trading strategy to arbitrage between SPY and US stock market ten sectors ETFs with PCA.

First step is importing packages we need.

```
import pandas as pd
import numpy as np
from sklearn.decomposition import PCA
from sklearn.preprocessing import scale
from  datetime import datetime, timedelta
import statsmodels.api as sm
import pytz
```

Then we will initialize our portfolio. The build the portfolio with Index: ‘SPY’ and US ten sectors ETFs     'IYJ','IYW','IYH','IYZ','IDU','IYF','IYE','IYK','IYM','IYC'

```
def initialize(context):

    context.stocks=symbols('IYJ','IYW','IYH','IYZ','IDU','IYF',
                                  'IYE','IYK','IYM','IYC')
    context.index=symbol('SPY')
    
    set_commission(commission.PerTrade(cost=0))
    context.spreads=[]

    context.window_length=181
    context.zscore_length=181
    context.zscore_invested=[]
    
    context.invested=False
    
    schedule_function(my_rebalance, date_rules.every_day(), time_rules.market_open(hours=1))

```
The model we use is simple multiple linear regression. Then we are going to improve the model with PCA. The model is shown below:

[![QQ图片20170321155035.png](https://s2.postimg.org/vtblno2xl/QQ_20170321155035.png)](https://postimg.org/image/8rv0hx39x/)

We first trained our model using historical data.

```
def before_trading_start(context, data):
    
    stocks_his=data.history(context.stocks,'price',           
                            context.window_length,
                            '1d').iloc[0:context.window_length-1].dropna()
    
    index_his=data.history(context.index,'price',           
                            context.window_length,
                            '1d').iloc[0:context.window_length-1].dropna()
    
    context.stocks_his=stocks_his
    context.index_his=index_his
    
    beta=PCRegression(context,context.index_his,context.stocks_his)
    zscore=compute_zscore(context,context.spreads)
    
    context.beta=beta
    context.zscore=zscore
         
def handle_data(context,data):
    
    record(zscore=context.zscore,position=context.portfolio.positions_value)
    
```
We reduce the number of variables using [Principal component analysis (PCA)](https://en.wikipedia.org/wiki/Principal_component_analysis)   

```
def PCRegression(context,Y, X):

    pca = PCA()
    A = scale(X)
    B = scale(Y)
    
    a1 = np.ones((A.shape[0],A.shape[1] + 1))
    a1[:, 1:] = A
    A = a1
    
    newA = pca.fit_transform(A)[:,:-1]
    a2 = np.ones((newA.shape[0], newA.shape[1] + 1))
    a2[:, 1:] = newA
    newA = a2
    
    regr1 = sm.OLS(B, A).fit()
    beta=regr1.params   
    
    regr2 = sm.OLS(B, newA).fit()
    pc_beta=regr2.params
    
    
    spread=B[-1]-np.dot(newA[-1,0:7],pc_beta[0:7])
    context.spreads.append(spread)
    
    return beta

```

We calculate z score. 
If Zscore >= 2, we sell SPY and buy ETFs. 

If Zscore <= -2, we buy SPY and sell ETFs. 

Else, if Abs(Zscore) <= Abs(Zscore_invested) -1, we close out our positions.

```

def compute_zscore(context,spreads): 

    if len(spreads)<context.zscore_length:
        return

    spread_wind=spreads[-(context.zscore_length-1):]
    zscore=(spreads[-1]-np.mean(spread_wind))/np.std(spread_wind)
    context.zscore_length+=1
    
    return zscore

def my_rebalance(context,data):
    
    place_order(context,data,context.zscore,context.beta)
    context.window_length+=1
```    

Then we can start trading.

```
def place_order(context,data,zscore,beta):

    if zscore== None:
        return
    
    stocks_price=data.current(context.stocks,'price')
    index_price=data.current(context.index,'price') 
    a = np.ones((1,stocks_price.count() + 1))
    a[:,1:] = np.array(stocks_price)
    stocks_price_ = a
    x_=np.dot(stocks_price_,beta)
    
    
    y_value=context.portfolio.portfolio_value*index_price\
           /(index_price+x_)
    x_value=context.portfolio.portfolio_value-y_value
    x_weight=x_value*beta[1:]*stocks_price/np.dot(beta[1:],stocks_price)
    
    if zscore>=1.2 and not context.invested:
        for i in range(len(context.stocks)):
                order_target_value(context.stocks[i],4*x_weight[i])   
        order_target_value(context.index,-4*y_value)
        context.invested=True
        context.zscore_invested=zscore
        
    elif zscore<=-1.2 and not context.invested:
        for i in range(len(context.stocks)):
                order_target_value(context.stocks[i],-4*x_weight[i])
        order_target_value(context.index,4*y_value)
        context.invested=True
        context.zscore_invested=zscore        
        
    elif np.abs(zscore)<(np.abs(context.zscore_invested)-1)and context.invested:    
         for stock in context.stocks:
            order_target_value(stock,0)
         order_target_value(context.index,0)
         context.invested=False
   
```

Here is the evaluation result of our pairs trading strategy.

[![图片1.png](https://s29.postimg.org/c4l1vkllj/image.png)](https://postimg.org/image/lcdac9snn/) 
