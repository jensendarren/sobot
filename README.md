# Sobot

## Sobot using Freqtrade

A bot built and configured using Freqtrade!

## Alias Docker-Compose Command

First, I recommend to alias `docker-compose` to `dc` and `docker-compose run --rm "$@"` to `dcr` to save of typing.

Put this in your `~/.bash_profile` file so that its always aliased like this!

```
alias dc=docker-compose
dcr() { docker-compose run --rm "$@"; }
```

Now run `source ~/.bash_profile`.

## Installing Freqtrade

Install and run [via Docker Quickstart](https://www.freqtrade.io/en/stable/docker_quickstart/).

Now install the necessary dependencies to run Freqtrade:

```
mkdir ft_userdata
cd ft_userdata/
# Download the dc file from the repository
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# Pull the freqtrade image
dc pull

# Create user directory structure
dcr freqtrade create-userdir --userdir user_data

# Create configuration - Requires answering interactive questions
dcr freqtrade new-config --config user_data/config.json
```

**NOTE**: Any freqtrade commands are available by running `dcr freqtrade <command> <optional arguments>`. So the only difference to run the command via `docker-compose` is to prefix the command with our new alias `dcr` (which runs `docker-compose run --rm "$@"` ... see above for details.)

## Config Bot

If you used the `new-config` sub-command (see above) when installing the bot, the installation script should have already created the default configuration file (`config.json`) for you.

The params that we will set to note are (from `config.json`). This allows all the available balance to be distrubuted accross all possible trades. So in dry run mode we have a default paper money balance of 1000 (can be changed using `dry_run_wallet` param) and if we set to have a max of 10 trades then Freqtrade would distribute the funds accrosss all 10 trades aprroximatly equally (1000 / 10 = 100 / trade).

```
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
```

The above are used for Dry Runs and is the ['Dynamic Stake Amount'](https://www.freqtrade.io/en/stable/configuration/#dynamic-stake-amount). For live trading you might want to change this. For example, only allow bot to trade 20% of excahnge account funds and cancel open orders on exit (if market goes crazy!)

```
"tradable_balance_ratio": 0.2,
"cancel_open_orders_on_exit": true
```

Update the `pair_whitelist` settings in the config to match the pairs you are interested to trade. For example:

```
"pair_whitelist": [
    "ALGO/USDT",
    "ATOM/USDT",
    "BAT/USDT"
]
```

For details of all available parameters, please refer to [the configuration parameters docs](https://www.freqtrade.io/en/stable/configuration/#configuration-parameters).

## Create a Strategy

https://www.freqtrade.io/en/stable/strategy-customization/

So I've created a 'BBRSINaiveStrategy' based on RSI and Bollenger Bands. Take a look at the file [`bbrsi_naive_strategy.py`](bbrsi_naive_strategy.py) file for details.

To tell your instance of Freqtrade about this strategy, do the following:

1. Copy the `bbrsi_naive_strategy.py` into your `ft_userdata/user_data/strategies` directory using `cp bbrsi_naive_strategy.py ft_userdata/user_data/strategies/.`
1. Open your `docker-compose.yml` file and update the strategy flag (last flag of the command) to `--strategy BBRSINaiveStrategy`

## Remove past trade data

If you have run the bot already, you will need to clear out any existing dry run trades from the database. The easiest way to do this is to delete the sqlite database by running the command `rm user_data/tradesv3.sqlite`.

## Sandbox / Dry Run

As a quick sanity check, you can now immediately start the bot in a sandbox mode and it will start trading (with paper money - not real money!).

In development / testing environment we only need to run the `Freqtrade` container. To start trading in sandbox mode, simply start the service as a daemon using Docker Compose, like so and follow the log trail as follows:

```
dc up -d freqtrade
dc ps
dc logs -f
```

## Setup a pairs file

We will use Binance so we create a data directory for binance and copy our `pairs.json` file into that directory:

```
mkdir -p user_data/data/binance
touch user_data/data/binance/pairs.json
```

## Download Data

Now that we have our pairs file in place, lets download the OHLCV data for backtesting our strategy.

```
dcr freqtrade download-data --exchange binance -t 15m
```

**NOTE**: If you already have backtesting data available in your data-directory and would like to refresh this data up to today, do not use --days or --timerange parameters. Freqtrade will keep the available data and only download the missing data. If you are updating existing data after inserting new pairs that you have no data for, use --new-pairs-days xx parameter.

List the available data using the `list-data` sub-command:

```
dcr freqtrade list-data --exchange binance
```

Manually inspect the json files to examine the data is as expected (i.e. that it contains the expected `OHLCV` data requested).

Optionally, you can confirm the timestamps are correct by checking that the first is the expected start date, the last is the expected end date and the intervals are the expected time interval. To use Python to convert timestamps to dates use the following code in a Python terminal:

```
import datetime
timestamp = 1618445100
date = datetime.datetime.fromtimestamp(1618445100)
print(date.strftime('%Y-%m-%d %H:%M:%S'))
```

## OPTIONAL: Convert the backtesting data to be more compact

Now its possible to convert to a more compact data type of `jsongz` like so:

```
dcr freqtrade convert-data --format-from json --format-to jsongz --datadir user_data/data/binance -t 1h 4h --erase
```

Note to list the available data you need to pass the `--data-format-ohlcv jsongz` flag as below:

```
dcr freqtrade list-data --exchange binance --data-format-ohlcv jsongz
```

## Backtest

Now we have the data for 15m OHLCV data for our pairs lets Backtest this strategy:

```
dcr freqtrade backtesting --datadir user_data/data/binance --export trades -s BBRSINaiveStrategy -i 15m
```

For details on interpreting the result, refer to ['Understading the backtesting result'](https://www.freqtrade.io/en/stable/backtesting/#understand-the-backtesting-result).

## Backtesting multiple strategies

You might want to test multiple strategies at once and compare the results side by side. In this case use the `--strategy-list` option like below. Note in the below example, I've included the `--timerange` option set to `20210201-202102028` which means to use downloaded test data between 1st February 2021 till 28th February 2021. You can remove this option to test against all data as well.

```
dcr freqtrade backtesting --strategy-list Strategy001 Strategy002 Strategy003 Strategy004 Strategy005 --timerange=20210201-20210228 --timeframe 5m
```

## Plotting

Plot the results to see how the bot entered and exited trades over time. Remember to change the Docker image being referenced in the docker-compose file to `freqtradeorg/freqtrade:develop_plot` before running the below command.

Note that the `plot_config` that is contained in the strategy will be applied to the chart.

```
dcr freqtrade plot-dataframe --strategy BBRSINaiveStrategy -p ALGO/USDT -i 15m
```

Once the plot is ready you will see the message `Stored plot as /freqtrade/user_data/plot/freqtrade-plot-ALGO_USDT-15m.html` which you can open in a browser window.

## Optimize

To optimize the strategy we will use the Hyperopt module of freqtrade. First up we need to create a new hyperopt file from a template:

```
dcr freqtrade new-hyperopt --hyperopt BBRSIHyperopt
```

Now add desired definitions for buy/sell guards and triggers to the Hyperopt file. Then run the optimization like so (NOTE: set the time interval and the number of epochs to test using the `-i` and `-e` flags:

```
dcr freqtrade hyperopt --hyperopt BBRSIHyperopt --hyperopt-loss SharpeHyperOptLoss --strategy BBRSINaiveStrategy -i 15m -e 5000
```

## Update Strategy

Apply the suggested optimized results from the Hyperopt to the strategy. Either replace the current strategy or create a new 'optimized' strategy.

## Backtest (with both strategies)

Now we have updated our strategy based on the result from the hyperopt lets run a backtest again:

```
dcr freqtrade backtesting --datadir user_data/data/binance --data-format-ohlcv jsongz --export trades --stake-amount 100 -s BBRSIOptimizedStrategy -i 15m
```

## Sandbox / Dry Run

Before you run the Dry Run, don't forget to check your local config.json file is configured. Particularly the `dry_run` is true, the `dry_run_wallet` is set to something reasonable (like 1000 USDT) and that the `timeframe` is set to the same that you have used when building and optimizing your strategy!

```
"max_open_trades": 10,
"stake_currency": "USDT",
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
"fiat_display_currency": "USD",
"timeframe": "15min",
"dry_run": true,
"dry_run_wallet": 1000,
```

## View Dry Run via Freq UI

For use with docker you will need to enable the api server in the Freqtrade config and set `listen_ip_address` to "0.0.0.0", and also set the `username` & `password` so that you can login like so:

```
...

"api_server": {
  "enabled": true,
  "listen_ip_address": "0.0.0.0",
  "username": "Freqtrader",
  "password": "secretpass!",
P
...
```

In the docker-compose.yml file also map the ports like so:

```
ports:
  - "127.0.0.1:8080:8080"
```

Then you can access the Freq UI via a browser at [http://127.0.0.1:8080/](http://127.0.0.1:8080/). You can also access and control the bot via a REST API too!

---

## Sqlite DB

https://www.freqtrade.io/en/stable/sql_cheatsheet/

## Plotting Profit

https://www.freqtrade.io/en/stable/plotting/#plot-dataframe-basics

## Deploying to Production

A couple of steps but might take you time! Assuming you have a server available already that you can ssh into as a sudoer user.

* Step 1: [Install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04), clone the repo and start the service.
* Step 2: [Install Nginx & Certbot containers using Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-secure-a-containerized-node-js-application-with-nginx-let-s-encrypt-and-docker-compose) to serve the bot via SSL.

Then run in production server using:

```
docker-compose up -d
```