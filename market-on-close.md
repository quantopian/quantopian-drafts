# Trading At Market Close

In minute mode `handle_data` is called every minute which is okay when checking for intraday signals, but what if you wanted to execute your code only once per day? `is_market_close()` is a method that checks if it’s currently at the market close and returns True if it is and False otherwise.

An example use of the method would be if you wanted to order a security at the close price, you would first call `is_market_close()`: e.g. 

```
def handle_data(context,data):
    if (is_market_close()):
        order(5000, sid)
```

This would execute orders at 4:00PM. It also takes into account early closes such as the Friday after Thanksgiving where the market closes at 1 PM.

`is_market_close()` takes in a `minutes_early` integer, which will return True at 4:00PM - `minutes_early`. 

```
is_market_close(5)
```

The above snippet would return True at 3:55PM, BUT will also return `True` for every minute after 3:55PM. This is to account for missing bars in data (e.g. if data doesn't exist for the minute bar at 3:55PM, the order will execute at 3:56PM instead). That being said, an order function that uses a smart target (e.g. reweighting for a target X weight of security Y) is recommended if passing in an `minutes_early` integer, otherwise, your order function will execute at 3:55PM, 3:56 PM, 3:57PM and etc. 

So the following order function would order SPY 500 time instead of just 100
```
if(is_market_close(5)):
	order(100, sid)
```
