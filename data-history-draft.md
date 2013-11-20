# Data History

In many strategies, it is useful to compare the most recent bar data to previous bars.
Finding signals in recent pricing data is one of the core building blocks of algorithmic trading.
Accordingly, the Quantopian platform provides utilities to easily access and perform calculations on recent history.

## What's Changing

Previous iterations on this concept were the `batch_transform` decorator, and the `mavg`, `stddev`
"rolling transform" wrappers.

The new utility of `history` will be used to replace most, and hopefully all, of the uses of `batch_transform` calculations.
`batch_transform` had some complexity and ensuing bugs around `window_length` and `refresh_frequency`, especially
some impedance mismatches around using minute data and specifying the parameters in day units.

Also, there were some memory and CPU performance concerns around storing 390 minutes per day, when most
algorithms that use many days of pricing history are usually expecting the daily close prices over that time span.

`history` can also be used to replace the `mavg`, `stddev` utilities, as well as opening up more custom calculations.
The use of `mavg` and other transforms, e.g. `data[sid(24)].mavg(10)` also had some inefficiencies with the CPU and memory.

## Bar Units

So that the unit of measurement is explicit, we have add the `frequency` field, which defines the how often the data is sampled.
For daily backtests, the function `data.history(bar_count=7, frequency='1d', field='price')` will return seven days worth of daily bars.
For minute backtests, the function `data.history(bar_count=15, frequency='1m', field='price')` will return 15 minutes of minute bars.

## Data Shape

Each data point data returned by the history API will have the following values:

- `dt`, the timestamp of the last bar in the sample
- `open`, the open of the first bar in the sample
- `high`, the max of the highs in the sample
- `low`, the min of the lows in the sample
- `close`, the close of the last bar in the sample
- `volume`, the total volume of the bars in the sample

One way to conceptualize this is that the data returned by history returns "boxes" of the underlying data.

### Specifying bar_count and frequency

The frequency parameters above, i.e. "7d", "15m", and "3d", specify how often the data is sampled.
The bar_count specifies the number of samples to include in the dataframe returned by the history function.
Below are code for example scenarios, along with explanations of the data returned.

- Monthly Units
    - "M" suffix to the frequency parameter.
        - Months are always calendar months.
        - data.history(1, "1M", "price") will return a dataframe with 1 row corresponding to the current calendar
          month of algorithm time, up until the current bar.
        - The current monthly bar will have:
             - open as the open of the first business day of the month
             - close as the close of the most recent data bar delivered to the algorithm
        - All other months will have the open as the open of the first business day of the month, and the
          close as the close of the last business day of the month.

- Weekly Units
    - "W" suffix to the frequency parameter.
        - Weeks are always trading weeks, running Monday-Friday. For weeks where one or more days is a holiday,
          the weekly bar for that week is included as a complete week.
        - data.history(1, "1W", "price") returns the current week of the simulation, up until the current bar,
          even if the week is partial (i.e. current day is Wednesday, you'll receive one bar comprised of data from
          Monday through Wednesday).
        - The current week bar will have:
              - open as the open of the first business day of the week
              - close as the close of the most recent data bar delivered to the algorithm
        - All prior weeks will have the open as the open of the first business day in the week, and the
          close as the close of the last business day of the week.

- Day Units
    - "d" suffix to the frequency parameter
    - returns are always in daily bars and bars never span more than one trading day
    - examples:
        - data.history(1, "1d", "price") returns the data since the current day's open as a bar, even if it is partial.
        - data.history(2, "1d", "price") returns the previos day as a bar, as well as data since the open for the current day.
        - data.history(3, "1d", "price") returns the previos two days as bars, as well as data since the open for the current day.
    - frequency specified in days will treat any trading day as a single day unit. Scheduled half day sessions,
      early closures for emergencies such as Hurrican Sandy, and other truncated sessions are all treated as a 
      single trading day.

- Minute Units
    - returns are always in minutes. *Beware* that the returning dataframe may span more than one trading day.
    - specifying a frequency in minutes for a daily simulation will raise and exception.
    - "m" suffix to the frequency parameter
        - data.history(1, "1m", "price") returns a dataframe with one rwo corresponding to the current minute
        - data.history(2, "1m", "price") returns a dataframe with two rows, corresponding to the current minute 
          plus the previous minute
        - data.history(391, "1m", "price") returns a dataframe with 391 rows corresponding to the current minute plus
          the previous 390.
              - The purpose of this example is to illustrate that minute frequency history requests can span multiple
                trading days.
              - A full trading day is comprised of just 390 minutes, so the dataframe will always include all the
                minutes in the current day. In the Nth minute of the day, the dataframe will have N minutes from today
                plus 391-N minutes from the prior day. In the case of a half day, there are just 195 minutes of
                trading. If the prior day were a half day, at the open the dataframe returned would have the first bar
                from today, 195 bars for the half day, and 195 bars from two days prior.

For day, week, and month units, the values of the most recent bar in the history will differ when used in daily
simulation versus minutely simulation. The reason for the difference is that in minute simulation (as well as paper
trading and live trading) the current day can be partial.
Only at the last trading minute of each day will `date.history(bar_count=1, frequency='1d', field='price')` return
identical values for both minutely and daily simulations.

### Constraints

Frequency of one day and above (i.e., '1d', '2d', '1W','1M', etc.) have a limit of being within the date range of the backtest.
e.g. if the simulation date range was from 2011-01-01 to 2011-12-31, the limit for bar_count at a frequency of '1D' would be 252, the limit for a frequency of '2D' would be 176, for '1M' would be 12, etc.

In minute simulations, the number of data points in a history is limited to 2000, so that an algorithm does not consume too much memory.
The data points are counted by taking the product of the frequency and the count.
i.e. both `data.history(bar_count=2000, frequency='1m', field='price')` and `data.history(bar_count=1000, frequency='2m', field='price')`
are at the max.

As we improve memory capabilities, the minute limit may change.

#### More on Daily History in Minute-Bar Backtests

In minute-bar backtests, we provide an easier way to access daily pricing information.

To retrieve the daily data, when in minute mode, use the unit suffix `d`, when specifying the window length.
e.g., ``data.history(bar_count=2, frequency='1d', field='price')`

The daily history in relation to the minute bars, for each day, is:

- close_price, is the close of the last minute bar for that day
- open_price, is the first open of the first minute bar
- volume, is the sum of all minute `volume`s
- high, is the max of the minute `high`s
- low, is the min of the minute bar `low`s

For a window of `data.history` specified in days, e.g. `data.history(bar_count=2, frequency='1d', field='price')`, the returned data also includes today's values up to the current minute, labeled with the date.


If the data source looks like:
```
                        XYZ
2013-09-06 14:31:00     22.0
2013-09-06 14:32:00     21.5
...
2013-09-05 21:00:00     19.0
2013-09-06 14:31:00     20.0
2013-09-06 14:32:00     18.0
```

The following example code:

```
def handle_data(context, data):
    log.info(data.history(bar_count=2, frequency='1d', field='price'))
```

The output at 2013-09-06 14:31 UTC:
```
               XYZ
2013-09-05     19.0
2013-09-06     20.0
```


One minute later; at 2013-09-06 14:32 UTC:
```
               XYZ
2013-09-05     19.0
2013-09-06     18.0
```

## Illiquidity and Forward Filling

By default, `data.history` methods will forward fill missing bars.
The index that will be filled for daily will be the U.S. exchanges' trading calendar days.
For minutes, the index will be those trading days from 9:31 AM to 4:00 PM in New York time,
represented in UTC. Minutes will account for half trading days, and will fill to 9:31 AM to 1:00 PM on those days.

However, it can be useful to have visibility into whether a trade happened on a given day or minute.
We provide an option for the `data.history` methods, which disables forward filling.
e.g., `data.history(bar_count=15, frequency='1d', field='price', ffill=False)`
Will return a dataframe that has `nan` value for price for days on which a given sid did not trade.

## Specifying Field and Return Type

The `data.history` functions return a pandas DataFrame populated with the values for the field
specified in the second parameter.

e.g.

```
data.history(bar_count=2, frequency='1d', field='price')
```

The above returns a DataFrame populated with the price information for each stock in the universe.

The available options for the field parameter are:

- open_price
- high
- low
- close_price
- price
- volume

Also, if fetcher is used, the fetcher field will be available as an option.

## Current and Previous Bar

One use case is to get an OHLC value for the previous bar, to do some comparison
against the current minute.
This algo probably won't make any money, but it illustrates comparing the previous
bar to the current bar.

```
def handle_data(context, data):
    price_history = data.history(bar_count=2, frequency='1m', field='price')
    for s in data:
        prev_bar = price_history[s][-2]
        curr_bar = price_history[s][-1]
        if curr_bar > prev_bar:
            order(s, 20)
```

## Looking Back X Bars

It can also be useful to look further back into history for a comparison.
The percent change over a number of bars, needs just two data points which can
be accessed as such.

The following example operates over all stocks available in `data`, in pandas Series format.

```
def handle_data(context, data):
    prices = data.history(bar_count=15, frequency='1m', field='price')
    pct_change = (prices.ix[-1] - prices.ix[0]) / prices.ix[0]
    log.info(pct_change)
```

The pandas DataFrame provides many useful calculation tools, which turn what previously required
hand crafted code in the algo into one-line function calls:
http://pandas.pydata.org/pandas-docs/dev/api.html#api-dataframe-stats

Let's redo percent change example, leveraging some helpful pandas syntax.

We can use the `iloc` pandas DataFrame function to get the first and last values as a pair:

```
price_history.iloc[[0, -1]]
```

The percent change example can be re-written as:

```
def handle_data(context, data):
    price_history = data.history(bar_count=15, frequency="1d", field='price')
    pct_change = price_history.iloc[[0, -1]].pct_change()
    log.info(pct_change)
```

The difference code example in the Current and Previous Bar section can also be written as:

```
def handle_data(context, data):
    price_history = data.history(bar_count=15, frequency="1d", field='price')
    diff = price_history.iloc[[0, -1]].diff()
    log.info(diff)
```

## Panel Calculation, née Batch Transform

Where previously batch transform would be used, a panel is now passed
to a function. To show this difference, here is an OLS transform.

### Using Older Batch Transform

```
import statsmodels.api as sm

@batch_transform(window_length=30)
def ols_transform(panel, sid1, sid2):
    """
    Computes regression coefficient (slope and intercept)
    via Ordinary Least Squares between two SIDs.
    """
    p0 = panel.price[sid1]
    p1 = sm.add_constant(panel.price[sid2], prepend=True)
    return sm.OLS(p0, p1).fit().params

def initialize(context):
    context.sid1 = sid(42)
    context.sid2 = sid(123)

def handle_data(context, data):
    slope, intercept = ols_transform(data, sid1, sid2)
```

### Using New History Method

With the new history available within data, the above code will be:

```
import statsmodels.api as sm

def ols_transform(prices, sid1, sid2):
    """
    Computes regression coefficient (slope and intercept)
    via Ordinary Least Squares between two SIDs.
    """
    p0 = prices[sid1]
    p1 = sm.add_constant(prices[sid2], prepend=True)
    return sm.OLS(p0, p1).fit().params
    
def initialize(context):
    context.sid1 = sid(42)
    context.sid2 = sid(123)

def handle_data(context, data):
    price_history = data.history(bar_count=30, frequency='1d', field='price')
    slope, intercept = ols_transform(price_history, context.sid1, context.sid2)
```



### History and Back Test Start

The data available to history is only limited by our historical database. The full history is available on the first day of the backtest.

Data history does not require the algorithm to run for bar_count bars to fill the trailing window before returning a value.

The data is backfilled so that calculations can be done immediately.

### TALib Port

```
macd = ta.MACD(window_length=30)

def initialize(context):
    set_universe(DollarVolumeUniverse(95, 95.1))

def handle_data(context, data):
    macd_result = macd(data)
```

is now:

```
def initialize(context):
    set_universe(DollarVolumeUniverse(95, 95.1))

def handle_data(context, data):
    price_history = data.history(bar_count=30, frequency='1d', field='price')
    macd_result = talib.MACD(price_history)
```

### Data Strucure

The history's data structure is a pandas DataFrame with the shape of:
- items, OHLCV data
- minor_axis = sids
- major_axis = timestamps


### Multiple Time Frequencies

It is now easy to use multiple time ranges and frequencies as signal inputs.

```
# An algorithm that uses the moving average both over days and minutes

def handle_data(context, data):
    daily_prices = data.history(bar_count=30, frequency='1d', field='price')
    minute_prices = data.history(bar_count=15, frequency='1m', field='price')
    for s in data:
        if daily_prices[s].mean() < minute_prices[s].mean():
            order(s, 50)
```

### Rolling Transforms

The rolling transforms of mavg, stddev, etc. will now be replaced by their pandas counterparts.

- mavg -> DataFrame.mean
- stddev -> DataFrame.std
- vwap -> DataFrame.sum, for volume and price

#### Standard Deviation

```
def handle_data(context, data):
    price_history = data.history(bar_count=5, frequency='1d', field='price')
    log.info(price_history.std())
```

#### Moving Average

```
def handle_data(context, data):
    price_history = data.history(bar_count=5, frequency='1d', field='price')
    log.info(price_history.mean())
```

#### VWAP

```
# vwap

def vwap(prices, volumes):
    return (prices * volumes).sum() / volumes.sum()

def handle_data(context, data):
    prices_a = data.history(15, '1m, field='price')
    volumes_a = data.history(15, '1m', field='volume')

    prices_b = data.history(30, '1m', field='price')
    volumes_b = data.history(30, '1m', field='volume')

    vwap_a = vwap(prices_a, volumes_a)
    vwap_b = vwap(prices_b, volumes_b)

    for s in data:
        if vwap_a[s] > vwap_b[s]:
            order(s, 50)

    log.info(vwap)
```

##### Transform Wrappers

Since the implementation of transforms like VWAP can be cumbersome,
we will provide some wrappers which provide the data extraction and application
of the transform in one step.

e.g.

```
vwap_a = data.transforms.vwap(bar_count=15, frequency='1m')
vwap_b = data.transforms.vwap(bar_count=15, frequency='30m')

for s in data:
    if vwap_a[s] > vwap_b[s]:
        order(s, 50)
```

These will replace the older style of `vwap`, i.e., `data[sid(24)].vwap(10)`.
The older style will be considered deprecated and unsupported.

### Resampling

Through history we will provide data at minute and daily frequency, however
there are use cases for using data at other frequencies.

To start, `data.history` will not provide those frequencies, but the DataFrames returned
by history can be resampled down to meet those needs.

#### Hour

The following code examples are recipes for sampling different values down to hourly
chunks.

##### Price

```
def handle_data(context, data):
    prices = data.history(300, '1m', field='price')
    weekly_prices = prices.resample('H', how='last')
```

##### Volume

```
def handle_data(context, data):
    prices = data.history.minutes(300, '1m', field='volume')
    weekly_prices = prices.resample('H', how='sum')
```

##### Open

```
def handle_data(context, data):
    prices = data.history(300, '1m', field='open')
    weekly_prices = prices.resample('H', how='first')
```

### Extract Minutes For Day

If the current days of minutes is desired, pandas built-in indexing can be used
to extract those values.
This example uses 390 minutes, which is a normal trading day's worth of minutes,
but the indexing allows the algorithm to use just today's minutes, even on half day's.

```
def handle_data(context, data)
    prices = data.history(bar_count=390, frequency='1m', field='open')
    todays_prices = prices.ix[get_datelabel()]

    log.info(today_prices)
```

### Trading At Market Close

In minute mode `handle_data` is called every minute which is okay when checking for intraday signals, but what if you wanted to execute your code only once per day? `endofday_check` is a method that checks if it’s currently at the market close and returns True if it is and False otherwise

An example use of the method would be if you wanted to order a security at the close price, you would first call `endofday_check`: e.g. 

```
def handle_data(context,data):
    if (endofday_check()):
        order(5000, sid)
```

This would execute orders at 3:45 PM (the last time that the NYSE accepts orders). It also takes into account early closes such as the Friday after Thanksgiving where the market closes at 1 PM. 

`endofday_check` takes in a `minutes_early` integer, which will return True at 3:45 PM - `minutes_early`. 

So the following would return True at 3:40PM

```
endofday_check(5)
```
