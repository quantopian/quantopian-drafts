# Data History

In many strategies, it is useful to compare the most recent bar data to previous bars.
Finding signals in recent pricing data is one of the core building blocks of algorithmic trading.
Accordingly, the Quantopian platform provides utilities to easily access and perform calculations on recent history.

## What's Changing

Previous iterations on this concept were the `batch_transform` decorator, and the `mavg`, `stddev`
"rolling transform" wrappers.

The new utility of `history` will be used to replace most, and hopefully all, of the uses of `batch_transform` calculations.
`batch_transform` had some complexity and ensuing bugs around `window_length` and `refresh_period`, especially
some impedance mismatches around using minute data and specifying the parameters in day units.

Also, there were some memory and CPU performance concerns around storing 390 minutes per day, when most
algorithms that use many days of pricing history are usually expecting the daily close prices over that time span.

`history` can also be used to replace the `mavg`, `stddev` utilities, as well as opening up more custom calculations.
The use of `mavg` and other transforms, e.g. `data[sid(24)].mavg(10)` also had some inefficiencies with the CPU and memory.

## Bar Units

So that the unit of measurement is explicit, we have add the `period` field, which defines the how often the data is sampled.
For daily backtests, the function `data.history(bar_count=7, period='1d', field='price')` will return seven days worth of daily bars.
For minute backtests, the function `data.history(bar_count=15, period='1m', field='price')` will return 15 minutes of minute bars.

## Data Shape

Each data point data returned by the history API will have the following values:

- `dt`, the timestamp of the last bar in the sample
- `open`, the open of the first bar in the sample
- `high`, the max of the highs in the sample
- `low`, the min of the lows in the sample
- `close`, the close of the last bar in the sample
- `volume`, the total volume of the bars in the sample

One way to conceptualize this is that the data returned by history returns "boxes" of the underlying data.

### Specifying Period

The parameters above, i.e. "7d", "15m", and "3d" specify the length of the window returned.

- days
    - "d"
        - "1d", returns the current bar.
        - "2d", returns a sampling of the current and previous bar.
        - "3d", returns a sampling of the current bar and the two previous
        - and so on

- minutes
    - "m"
    The 'm' suffix returns an absolute number of minute bars.
        - "1m", returns the current minute
        - "2m", returns the a sampling of the current minute plus the previous minute
        - "391m", returns a sampling of all the minutes in the current day, plus the difference between how many minutes
          have occurred so far in the current and the minutes in the previous day. In the case of a half-day,
          the 391 minute window would extend over three calendar days.
    - "d"
    Often when dealing with minute data, it is useful to specify the window length in days. The day window
    always starts at the beginning of a trading day, it is not a rolling start within the day.
        - "1d", returns a sampling of the minutes so far in the current day.
        - "2d", returns a sampling the minutes so far in the current day, plus all the minutes from the previous day.
        - and so on...

- days and minutes
    - "W"
    The 'W' suffix returns weeks.
        - "1W", The -1 indexed week will be the current week of the algorithm time, up until the current bar.
        All other weeks will have the open as the open of the first business day in the week, and the
        close as the close of the last available business day of the week.
    - "M"
    The 'M' suffix returns months.
        - "1M", The -1 indexed month will be the current month of algorithm time, up until the current bar.
        All other months will have the open as the open of the first business day of the monht, and the
        close as the close of the last available business day of the month.
        

Note with "d" in minute mode, a half day counts as a total day, so the total number of minutes sampled
when specifying the window using days can change depending on holidays.

To further cement the relationship of using "d" in minute mode and using "d" in daily mode.
On the last trading minute of each day, the statement of `date.history(bar_count=1, period='1d', 'price')` should
be the same for both.

### Constraints

Period of one day and above (i.e., '1d', '2d', '1W','1M', etc.) have a limit of being with in the date
range of the backtest.
e.g. if the algo range was from 2011-01-01 to 2011-12-01, the limit for a period of '1D' would be 252,
the limit for a period of '2D' would be 176, for '1M' would be 12, etc.

In minute mode, so that an algorithm does not consume too much memory,
minute mode is limited to 2000 minutes.
The minutes are counted by the product of the period and the count.
i.e. both `data.history(bar_count=2000, period='1m', 'price')` and `data.history(bar_count=1000, period='2m', 'price')`
are at the max.

As we improve memory capabilities, the minute limit may change.

#### More on Daily History in Minute-Bar Backtests

In minute-bar backtests, we provide an easier way to access daily pricing information.

To retrieve the daily data, when in minute mode, use the unit suffix `d`, when specifying the window length.
e.g., ``data.history(bar_count=2, period='1d', 'price')`

The daily history in relation to the minute bars, for each day, is:

- close_price, is the close of the last minute bar for that day
- open_price, is the first open of the first minute bar
- volume, is the sum of all minute `volume`s
- high, is the max of the minute `high`s
- low, is the min of the minute bar `low`s

For a window of `data.history` specified in days, e.g. `data.history(bar_count=2, period='1d', 'price')`, the returned data also includes today's values up to the current minute, labeled with the date.


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
    log.info(data.history(bar_count=2, frequncy='1d', 'price'))
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
We will provide an option for the `data.history` methods, which will disable forward filling.
e.g., `data.history(bar_count=15, period='1d', 'price', ffill=False)`
In that case, the missing day or minute data will have a `nan` value for price, open, volume, etc.

## Specifying Field and Return Type

The `data.history` functions return a pandas DataFrame populated with the values for the field
specified in the second parameter.

e.g.

```
data.history(bar_count=2, period='1d', 'price')
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
    price_history = data.history(bar_count=2, period='1m', 'price')
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
    prices = data.history(bar_count=15, period='1m', 'price')
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
    price_history = data.history(bar_count=15, "1d", 'price')
    pct_change = price_history.iloc[[0, -1]].pct_change()
    log.info(pct_change)
```

The difference code example in the Current and Previous Bar section can also be written as:

```
def handle_data(context, data):
    price_history = data.history(bar_count=15, period="1d", field='price')
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
    price_history = data.history(bar_count=30, period='1d', 'price')
    slope, intercept = ols_transform(price_history, context.sid1, context.sid2)
```



### History and Back Test Start

Unlike batch transforms, history is available on the first day of the backtest, data availability permitting.

Unlike batch transforms, the data history does not require the algorithm to run for the number
of days the window before returning a value.

The data is backfilled so that calculations can be done more immediately.

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
    price_history = data.history(bar_count=30, period='1d', field='price')
    macd_result = talib.MACD(price_history)
```

### Data Strucure

The history's data structure is a pandas DataFrame with the shape of:
- items, OHLCV data
- minor_axis = sids
- major_axis = timestamps


### Multiple Time Frequencies

It should now be easier to use multifactor time ranges and frequencies as signal inputs.

```
# An algorithm that uses the moving average both over days and minutes

def handle_data(context, data):
    daily_prices = data.history(bar_count=30, period='1d', field='price')
    minute_prices = data.history(bar_count=15, period='1m', field='price')
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
    price_history = data.history(bar_count=5, period='1d', 'price')
    log.info(price_history.std())
```

#### Moving Average

```
def handle_data(context, data):
    price_history = data.history(bar_count=5, period='1d', 'price')
    log.info(price_history.mean())
```

#### VWAP

```
# vwap

def vwap(prices, volumes):
    return (prices * volumes).sum() / volumes.sum()

def handle_data(context, data):
    prices_a = data.history(15, '1m, 'price')
    volumes_a = data.history(15, '1m', 'volume')

    prices_b = data.history(30, '1m', 'price')
    volumes_b = data.history(30, '1m', 'volume')

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
vwap_a = data.transforms.vwap(bar_count=15, period='1m')
vwap_b = data.transforms.vwap(bar_count=15, period='30m')

for s in data:
    if vwap_a[s] > vwap_b[s]:
        order(s, 50)
```

These will replace the older style of `vwap`, i.e., `data[sid(24)].vwap(10)`.
The older style will be considered deprecated and unsupported.

### Resampling

Through history we will provide data at minute and daily period, however
there are use cases for using data at other frequencies.

To start, `data.history` will not provide those frequencies, but the DataFrames returned
by history can be resampled down to meet those needs.

#### Hour

The following code examples are recipes for sampling different values down to hourly
chunks.

##### Price

```
def handle_data(context, data):
    prices = data.history(300, '1m', 'price')
    weekly_prices = prices.resample('H', how='last')
```

##### Volume

```
def handle_data(context, data):
    prices = data.history.minutes(300, '1m', 'volume')
    weekly_prices = prices.resample('H', how='sum')
```

##### Open

```
def handle_data(context, data):
    prices = data.history(300, '1m', 'open')
    weekly_prices = prices.resample('H', how='first')
```

### Extract Minutes For Day

If the current days of minutes is desired, pandas built-in indexing can be used
to extract those values.
This example uses 390 minutes, which is a normal trading day's worth of minutes,
but the indexing allows the algorithm to use just today's minutes, even on half day's.

```
def handle_data(context, data)
    prices = data.history(bar_count=390, period='1m', 'open')
    todays_prices = prices.ix[get_datelabel()]

    log.info(today_prices)
```
