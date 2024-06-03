---
layout: post
title: Trading&#58; Evolved Systematic Momentum Strategy in Quantconnect&#58; Part 1
---

In an effort to learn systematic trading, I am currently reading "Trading Evolved" by Andreas Clenow.
In Chapter 12, he talks about an equity strategy using momentum.

The rules are the following:
* Trading is done only monthly.
* Only stocks in the S&p 500 will be considered.
* Momentum slope will be calculated using 125 days.
* Top 30 stocks will be considered.
* Weights will be calculated for those 30 stocks using inverse volatility.
* Volatility is calculated using 20 day standard deviation.
* Trendfilter calculated based on 200 day average of the S&P 500 index.
* If the trendfilter is positive, we are allowed to buy.
* Minimum required momentum value is set to 40.
* Foreach of the 30 selected stocks if the momentum value is above 40, we buy it. If not, we leave the calculated money in cash.
* We sell stocks if they fallbelow the minimum required momentum value, or if they leave the index.
* Each month we rebalance and repeat.

Sounds promosing, right? Right?? Well, we'll see.

The first thing the chapter talks about is the investment universe, which is the S&P 500, duuh.
But we will have to make sure that we actually trade the stocks that were in the index at the time that the backtest is trading (aka avoiding survivorship bias).
If we would just trade the companies that are currently in the S&P 500, we are biased towards companies that are successful TODAY.
Nvidia was added to the S&P 500 in 2001 ([source](https://www.theregister.com/2001/11/30/nvidia_joinss_p/)), but if we would be trading it before that, we would make unrealistic profit.

So how do we do that? Unfortunately, compact data on which companies were member of the S&P 500 at whatever time is sparse. But I found some!
Checkout [this](https://github.com/fja05680/sp500) Github repo where [fja05680](https://github.com/fja05680) was nice enough to prepare it for us and constantly keeps it updated.

## Quantconnect
Andreas Clenow (the author of the book) uses [Zipline](https://github.com/stefan-jansen/zipline-reloaded) and Python to implement the strategy. However, as a C# Developer, I want to use Quantconnect because you can use it with C# (and python as well). Plus, Quantconnect has good and built in data and can be used in the cloud which means no setup hassle.

## Preprocessing the data
The first time I tried implementing this strategy, the backtest simply took too long because the Lean Engine (Quantconnect's backtest engine) had to create a universe of +/- 500 instruments every time rebalancing. I then decided to preprocess the data in fja05680's csv file and only include the top 30 symbols for each row / date, as well as their momentum score and volatility standard deviation.

## Not so fast...
Quantconnect does not have a premade "momentum score indicator" or a "volatility standard deviation indicator" that we can just use to preprocess the data, so we have to make it ourselves. Don't get me wrong, QC does have premade indicators, but just not these specific ones.

## Momentum Score Indicator
Pre Requirements: Basic C# knowledge. Free Tier Quantconnect account and have your project created. This code has only been tested in the cloud.
For the code, I won't include the using statements because it's a lot of them, but just most of them should already be added from the cloud environment.

Let's start..

The momentum score indicator will be a window indicator because it calculates the momentum score of a certain time window. Therefore, our class skeleton will look like this:
```cs
    public class ExponentialRegressionIndicator : WindowIndicator<IndicatorDataPoint>
    {
    }
```
Next, I am adding the constructor, which is rather simple here

```cs
    public class ExponentialRegressionIndicator : WindowIndicator<IndicatorDataPoint>
    {
        public double RecentScore { get; set; }
        private Action<string> LogFunc;
        public ExponentialRegressionIndicator(string name, int period, Action<string> logFunc)
            : base(name, period)
        {
            RecentScore = 0;
            this.LogFunc = logFunc;
        }
    }
```
The `LogFunc` is just so we can pass the `Debug` function from the main algorithm to this class. The `Debug` function just prints to the debug screen in Quantconnect.

`RecentScore` is the most recent score that we will use in the preprocessing.

`base(name, preriod)` will pass the selected window period to the parent class which is `WindowIndicator`.
### ComputeNextValue

Every (window) indicator has to implement the `decimal ComputeNextValue(IReadOnlyWindow<IndicatorDataPoint> window, IndicatorDataPoint input)` function.
During every trading iteration, that function is automatically called and has to return the next value (in this case momentum score).
The `window` parameter is populated by the main algorithm and includes all the data points (bar close prices etc.) of the window.

```cs

        protected override decimal ComputeNextValue(IReadOnlyWindow<IndicatorDataPoint> window, IndicatorDataPoint input)
        {
                double score = 0;
                if (window.Count < window.Size)
                {
                    return 0;
                }
                // Coming straight from https://numerics.mathdotnet.com/Regression
                double[] xdata = new double[window.Count];
                for (int i = 0; i < window.Count; i++)
                {
                    xdata[i] = i;
                }
                double[] ydata = new double[window.Count];
                for (int i = 0; i < window.Count; i++)
                {
                    ydata[i] = Math.Log((double)window[i].Value);
                }
                ydata = ydata.Reverse().ToArray(); // Because for some reason the window reverses itself.
                var regressionData = Fit.Line(xdata, ydata);
                double intercept = regressionData.Item1;
                double slope = regressionData.Item2;
                var rSquared = GoodnessOfFit.RSquared(xdata.Select(x => intercept + slope * x), ydata);
                var annualizedSlope = (Math.Pow(Math.Exp(slope), 252) - 1) * 100;
                score = annualizedSlope * rSquared;
                RecentScore = score;
            try
            {
                return (decimal)score;
            }
            catch (OverflowException)
            {
                this.logFunc($"OverflowException when casting score to decimal: {score} on {input.Symbol} : {input.EndTime}");
                return 0m;
            }
        }
```
To meausure momentum, Andreas recommends using the exponential regression and then normalizing it for outliers by multiplying the annualized slope with the R-Squared value of the regression line.
To calculate the slope, we use the Math.NET library. The example of fitting the line to the window values and calculating the R-Squared value comes straight from [here](https://numerics.mathdotnet.com/Regression).

At the end we annualize the slope with `var annualizedSlope = (Math.Pow(Math.Exp(slope), 252) - 1) * 100;`, multiply annualized slope with the R-Squared value and that really is it.
