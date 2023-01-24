# v1_momentum_tradingbot
version 1 of my trading bot using VWAP. This isnt the operational model using intraday data. 

	#Import used libs
	
	import datetime
	import pandas as pd
	import numpy as np 
	import matplotlib.pyplot as plt 
	plt.style.use('dark_background')
	#API import Alphavantage
	from alpha_vantage.timeseries import TimeSeries

	#insert private key 
	key = open('api_key_alphavantage.txt').read()

	#get two diff types data and meta create var for them, this is the api call
	ts = TimeSeries(key, output_format='pandas')
	data , meta = ts.get_daily_adjusted('SQQQ', outputsize='full')
	##pick stock symbol in "ts.get_daily_adjusted('####', outputsize='full')"

	#Change data here to reflect different outlook
	data1 = data[:'2022-01-01']
	data1

	new_columns = ['open', 'high', 'low', 'close', 'adj close','volume','dividend amount','split_coeff']
	data1.columns = new_columns

	plt.plot(data1[ 'adj close'])

##Bellow is the actual momentum strategy used later in intraday data in a different model

	def momentum_strategy(prices, volumes, vwap_period, threshold, total_cash, shares, cost_basis, is_bear_market):
  	#Calculate the VWAP of the security over the specified period
    vwap = sum([price * volume for price, volume in zip(prices[-vwap_period:], volumes[-vwap_period:])]) / (sum(volumes[-vwap_period:]) + 0.0001)

    if is_bear_market:
        # If the market is a bear market, modify the threshold levels
        threshold = threshold / 2  # Decrease the threshold level
    if prices[-1] > vwap + threshold:
        # If the current price is above the VWAP by the threshold amount, buy the security
        # Calculate the maximum amount that can be spent on the trade
        max_spend = total_cash * 0.2
          # Calculate the number of shares that can be bought with the maximum spend
        trade_shares = max_spend // prices[-1]
          # Deduct the cost of the trade from the total cash
        total_cash -= trade_shares * prices[-1]
          # Increase the number of shares held
        shares += trade_shares
          # Update the cost basis (the average price paid per share)
        cost_basis = (cost_basis * shares + trade_shares * prices[-1]) / (shares + trade_shares)
          # Return the trade type, the number of shares, and the current date
        return "BUY", trade_shares, datetime.datetime.now()
    elif prices[-1] < vwap - threshold:
          # If the current price is below the VWAP by the threshold amount, sell the security
          # Calculate the number of shares to sell
        trade_shares = shares
          # Add the proceeds of the sale to the total cash
        total_cash += trade_shares * prices[-1]
          # Reduce the number of shares held
        shares -= trade_shares
          # Update the cost basis
        cost_basis = 0
          # Return the trade type, the number of shares, and the current date
        return "SELL", trade_shares, datetime.datetime.now()
    else:
          # If the current price is neither above nor below the VWAP by the threshold amount, hold the security
          # Return the trade type, the number of shares, and the current date
        return "HOLD", 0, datetime.datetime.now()
	# Initialize variables
	vwap_period = 10
	threshold = 2
	total_cash = 4000
	shares = 0
	cost_basis = 0
	is_bear_market = 0
	initial_cash = total_cash
	trade_dates = []
	returns = []

##Bellow is script to backtest the above momentum strategy 

	def momentum_strategy(prices, vwap_period, threshold, total_cash, shares):
    # Calculate the VWAP of the security over the specified period
    vwap = sum(prices[-vwap_period:]) / vwap_period
    # Compare the current price to the VWAP
    if prices[-1] > vwap + threshold:
        # If the current price is above the VWAP by the threshold amount, buy the security
        # Calculate the maximum amount that can be spent on the trade
        max_spend = total_cash * 0.2
        # Calculate the number of shares that can be bought with the maximum spend
        trade_shares = max_spend // prices[-1]
        # Deduct the cost of the trade from the total cash
        total_cash -= trade_shares * prices[-1]
        # Increase the number of shares held
        shares += trade_shares
        return "BUY", trade_shares
    elif prices[-1] < vwap - threshold:
        # If the current price is below the VWAP by the threshold amount, sell the security
        # Calculate the number of shares to sell
        trade_shares = shares
        # Add the proceeds of the sale to the total cash
        total_cash += trade_shares * prices[-1]
        # Reduce the number of shares held
        shares -= trade_shares
        return "SELL", trade_shares
    else:
        # If the current price is neither above nor below the VWAP by the threshold amount, hold the security
        return "HOLD", 0
    


	# Define the parameters for the strategy
	vwap_period = 20 # number of periods to calculate the VWAP over
	threshold = 0.03 # threshold for buying or selling

	# Initialize variables for the strategy
	shares = 0
	cost_basis = 0
	trade_dates = []
	returns = []

	# Iterate through the prices data
	for i in range(vwap_period, len(data)):
    # Calculate the VWAP
    vwap = sum(data['prices'][i-vwap_period:i] * data['volume'][i-vwap_period:i]) / sum(data['volume'][i-vwap_period:i])
    
    # Compare the current price to the VWAP
    if data['prices'][i] > vwap + threshold:
        # If the current price is above the VWAP by the threshold amount, buy the security
        shares += 1
        cost_basis = data['prices'][i]
        trade_dates.append(data['date'][i])
        plt.scatter(data['date'][i], data['prices'][i], color='green', marker='^')
    elif data['prices'][i] < vwap - threshold:
        # If the current price is below the VWAP by the threshold amount, sell the security
        shares -= 1
        trade_dates.append(data['date'][i])
        plt.scatter(data['date'][i], data['prices'][i], color='red', marker='v')
    returns.append((shares*data['prices'][i]) - cost_basis)

	# Plot the strategy's performance
	plt.figure(figsize=(25,7))
	plt.plot(data['date'], data['price'])
	plt.xlabel('Date')
	plt.ylabel('Price')
	plt.title('Backtesting the Momentum Strategy')
	plt.show()

	# Print the profit/loss of the strategy
	profit_loss = (shares*data['prices'][-1]) - cost_basis
	print("Profit/Loss: ${:,.2f}".format(profit_loss))

	# Initialize lists to store the results
	trade_dates = []
	returns = []
	trade_count = 0
	# Test the strategy with a series of prices and volumes
	dates = data1.index.tolist()
	prices = data1['adj close'].tolist()
	volumes =  data1['volume'].tolist()
	vwap_period = 10
	threshold = 2
	total_cash = 4000
	shares = 0
	cost_basis = 0
	avg_price = 0 
	is_bear_market = 0

	for price, volume, date in zip(prices, volumes, dates):
    trade_type, trade_shares, trade_date = momentum_strategy([price], [volume], vwap_period, threshold, total_cash, shares, cost_basis,is_bear_market)
    trade_dates.append(trade_date)
    if trade_type == "BUY":
        # Deduct the cost of the trade from the total cash
        total_cash -= trade_shares * price
        # Increase the number of shares held
        shares += trade_shares
        # Update the cost basis (the average price paid per share)
        cost_basis = (cost_basis * shares + trade_shares * price) / (shares + trade_shares)
    elif trade_type == "SELL":
	# Add the proceeds of the trade to the total cash
        total_cash += trade_shares * price
	# Decrease the number of shares held
        shares -= trade_shares
	# Reset the cost basis
        cost_basis = 0
	# Update the average price of the total position
	if shares > 0:
    avg_price = (avg_price * shares - trade_shares * price) / (shares - trade_shares)
	else:
        avg_price = 0

	# If a trade was made, increment the counter
	if trade_type in ["BUY", "SELL"]:
        trade_count += 1

	print("Number of trades made:", trade_count)
	print(trade_type, trade_shares)
	print("Total cash:", total_cash)
	print("Shares held:", shares)
	print("Cost basis:", cost_basis)
	print("Average price:", avg_price)
	print()



	# Plot the prices and the trades
	plt.figure(figsize=(14,7))
	plt.title('Adjusted Close Price With Buy & Sell Signals')
	plt.plot(dates, prices)
	for i, trade_type in enumerate(trade_dates):
    if trade_type == "BUY":
        plt.scatter(dates[i], prices[i], color='green', marker='^')
    elif trade_type == "SELL":
        plt.scatter(dates[i], prices[i], color='red',  marker='v')
	plt.show()


	#Calculate the profit or loss of the trades
	profit_loss = total_cash - 4000
	print("Profit/loss:", profit_loss)



