# Trading At Market Close

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
