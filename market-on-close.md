# Trading At Market Close

In minute mode `handle_data` is called every minute which is okay when checking for intraday signals, but what if you wanted to execute your code only once and at the end of the trading day? `within_market_close()` is a method that checks if it’s currently at the market close and returns True if it is and False otherwise.

An example use of the method would be if you wanted to order a security at the close price, you would first call `within_market_close()`: e.g. 

```
def handle_data(context,data):
    if (within_market_close(0)):
        order(5000, sid)
```

`within_market_close()` would return `True` at 3:59 PM and the order would be carried out at the next available bar (e.g. 
4:00  PM). It also takes into account early closes such as the Friday after Thanksgiving where the market closes at 1:00 PM.

`within_market_close()` takes in a `minutes_early` integer, which will return True at 3:59 PM - `minutes_early`. 

So the following would return True at 3:54 PM and, if used with an order method, will execute orders at the next available
trading bar (usually the next minute e.g. 3:55 PM)

```
within_market_close(5)
```
