# Qbot Usage Examples and Tutorials

## Quick Start Guide

### 1. Installation and Setup

```bash
# Clone the repository
git clone https://github.com/UFund-Me/Qbot --depth 1
cd Qbot

# Install dependencies
pip install -r dev/requirements.txt

# Set environment variables
export PYTHONPATH=${PYTHONPATH}:$(pwd):$(pwd)/backend/multi-fact/mfm_learner

# Run the GUI application
python main.py  # On Mac: pythonw main.py
```

### 2. Basic Strategy Backtesting

```python
import backtrader as bt
import pandas as pd
import tushare as ts
from datetime import datetime
from qbot.strategies.sma_cross_strategy_bt import SmaCross

# Get stock data
def get_data(code, start="2020-01-01", end="2023-01-31"):
    df = ts.get_k_data(code, autype="qfq", start=start, end=end)
    df.index = pd.to_datetime(df.date)
    df["openinterest"] = 0
    df = df[["open", "high", "low", "close", "volume", "openinterest"]]
    return df

# Prepare data
dataframe = get_data("600018")  # 上港集团
start = datetime(2020, 1, 1)
end = datetime(2022, 12, 31)

# Create Cerebro engine
cerebro = bt.Cerebro()

# Add strategy with custom parameters
cerebro.addstrategy(SmaCross, pfast=10, pslow=30)

# Add data feed
data = bt.feeds.PandasData(dataname=dataframe, fromdate=start, todate=end)
cerebro.adddata(data)

# Set broker parameters
cerebro.broker.setcash(100000.0)  # Starting cash
cerebro.broker.setcommission(commission=0.001)  # 0.1% commission

# Add performance analyzers
cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')
cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')

# Run backtest
print(f'Starting Portfolio Value: {cerebro.broker.getvalue():.2f}')
results = cerebro.run()
print(f'Final Portfolio Value: {cerebro.broker.getvalue():.2f}')

# Extract performance metrics
strategy = results[0]
sharpe = strategy.analyzers.sharpe.get_analysis()
drawdown = strategy.analyzers.drawdown.get_analysis()
trades = strategy.analyzers.trades.get_analysis()

print(f'Sharpe Ratio: {sharpe.get("sharperatio", 0):.4f}')
print(f'Max Drawdown: {drawdown.get("max", {}).get("drawdown", 0):.2f}%')
print(f'Total Trades: {trades.get("total", {}).get("total", 0)}')
print(f'Win Rate: {trades.get("won", {}).get("total", 0) / max(trades.get("total", {}).get("total", 1), 1):.2%}')

# Plot results
cerebro.plot(style='candlestick')
```

### 3. Real-time Signal Generation

```python
import time
import tushare as ts
import talib
import pandas as pd
import numpy as np
from utils.larkbot import LarkBot

# Configuration
symbol = '600519'  # 贵州茅台
check_interval = 60  # Check every 60 seconds

# Initialize notification bot
bot = LarkBot(secret="your_webhook_secret")

def get_realtime_data(symbol):
    """Get real-time stock data"""
    try:
        stock_data = ts.get_realtime_quotes(symbol)
        return {
            'price': float(stock_data['price'].iloc[0]),
            'high': float(stock_data['high'].iloc[0]),
            'low': float(stock_data['low'].iloc[0]),
            'volume': float(stock_data['volume'].iloc[0])
        }
    except Exception as e:
        print(f"Error getting data: {e}")
        return None

def calculate_signals(historical_data, current_data):
    """Calculate trading signals"""
    # Technical indicators
    close_prices = historical_data['close']
    
    # Moving averages
    ma5 = talib.SMA(close_prices, timeperiod=5)
    ma20 = talib.SMA(close_prices, timeperiod=20)
    
    # RSI
    rsi = talib.RSI(close_prices, timeperiod=14)
    
    # MACD
    macd, signal, hist = talib.MACD(close_prices)
    
    # Bollinger Bands
    upper, middle, lower = talib.BBANDS(close_prices, timeperiod=20)
    
    # Generate signals
    signals = {}
    
    # MA Signal
    if ma5.iloc[-1] > ma20.iloc[-1]:
        signals['MA'] = 'BUY'
    elif ma5.iloc[-1] < ma20.iloc[-1]:
        signals['MA'] = 'SELL'
    else:
        signals['MA'] = 'HOLD'
    
    # RSI Signal
    if rsi.iloc[-1] < 30:
        signals['RSI'] = 'BUY'
    elif rsi.iloc[-1] > 70:
        signals['RSI'] = 'SELL'
    else:
        signals['RSI'] = 'HOLD'
    
    # MACD Signal
    if macd.iloc[-1] > signal.iloc[-1] and macd.iloc[-2] <= signal.iloc[-2]:
        signals['MACD'] = 'BUY'
    elif macd.iloc[-1] < signal.iloc[-1] and macd.iloc[-2] >= signal.iloc[-2]:
        signals['MACD'] = 'SELL'
    else:
        signals['MACD'] = 'HOLD'
    
    return signals

# Main monitoring loop
def monitor_stock(symbol):
    """Monitor stock and generate alerts"""
    while True:
        try:
            # Get historical data for indicators
            historical_data = ts.get_k_data(symbol, ktype='D')
            
            # Get current real-time data
            current_data = get_realtime_data(symbol)
            
            if current_data and not historical_data.empty:
                # Calculate signals
                signals = calculate_signals(historical_data, current_data)
                
                # Check for strong signals (2+ indicators agree)
                buy_signals = sum(1 for s in signals.values() if s == 'BUY')
                sell_signals = sum(1 for s in signals.values() if s == 'SELL')
                
                current_time = time.strftime('%Y-%m-%d %H:%M:%S')
                
                if buy_signals >= 2:
                    message = f"🟢 BUY SIGNAL - {symbol}\n"
                    message += f"Price: {current_data['price']}\n"
                    message += f"Time: {current_time}\n"
                    message += f"Signals: {signals}"
                    
                    print(message)
                    bot.send(message)
                
                elif sell_signals >= 2:
                    message = f"🔴 SELL SIGNAL - {symbol}\n"
                    message += f"Price: {current_data['price']}\n"
                    message += f"Time: {current_time}\n"
                    message += f"Signals: {signals}"
                    
                    print(message)
                    bot.send(message)
                
                else:
                    print(f"[{current_time}] {symbol}: {current_data['price']} - HOLD")
            
            time.sleep(check_interval)
            
        except KeyboardInterrupt:
            print("Monitoring stopped by user")
            break
        except Exception as e:
            print(f"Error in monitoring: {e}")
            time.sleep(check_interval)

# Start monitoring
if __name__ == "__main__":
    monitor_stock(symbol)
```

### 4. Multi-Strategy Portfolio

```python
import backtrader as bt
from qbot.strategies.sma_cross_strategy_bt import SmaCross
from qbot.strategies.multi_strategy_bt import MultiStrategy
from qbot.strategies.lstm_strategy_bt import LSTMPredict

class PortfolioStrategy(bt.Strategy):
    """Portfolio strategy that combines multiple sub-strategies"""
    
    params = (
        ('allocation_sma', 0.4),      # 40% to SMA strategy
        ('allocation_multi', 0.4),    # 40% to Multi strategy
        ('allocation_lstm', 0.2),     # 20% to LSTM strategy
    )
    
    def __init__(self):
        # Initialize sub-strategies
        self.sma_strategy = SmaCross()
        self.multi_strategy = MultiStrategy()
        self.lstm_strategy = LSTMPredict()
        
        # Portfolio allocation
        self.allocations = {
            'sma': self.p.allocation_sma,
            'multi': self.p.allocation_multi,
            'lstm': self.p.allocation_lstm
        }
        
        # Track positions for each strategy
        self.strategy_positions = {
            'sma': 0,
            'multi': 0,
            'lstm': 0
        }
    
    def get_strategy_signals(self):
        """Get signals from all sub-strategies"""
        signals = {}
        
        # SMA Crossover Signal
        sma_fast = bt.indicators.SMA(period=10)
        sma_slow = bt.indicators.SMA(period=30)
        sma_cross = sma_fast > sma_slow
        
        if sma_cross[0] and not sma_cross[-1]:
            signals['sma'] = 'BUY'
        elif not sma_cross[0] and sma_cross[-1]:
            signals['sma'] = 'SELL'
        else:
            signals['sma'] = 'HOLD'
        
        # Multi-factor Signal
        rsi = bt.indicators.RSI()
        sma_15 = bt.indicators.SMA(period=15)
        
        if (rsi[0] < 50 and rsi[-1] >= 50 and 
            self.data.close[0] > sma_15[0]):
            signals['multi'] = 'BUY'
        elif (rsi[0] > 50 and rsi[-1] <= 50 and 
              self.data.close[0] < sma_15[0]):
            signals['multi'] = 'SELL'
        else:
            signals['multi'] = 'HOLD'
        
        # LSTM Signal (simplified)
        # In practice, this would use the actual LSTM model
        signals['lstm'] = 'HOLD'  # Placeholder
        
        return signals
    
    def calculate_position_sizes(self, total_value):
        """Calculate position sizes based on allocations"""
        sizes = {}
        for strategy, allocation in self.allocations.items():
            allocated_value = total_value * allocation
            if self.data.close[0] > 0:
                sizes[strategy] = int(allocated_value / self.data.close[0])
            else:
                sizes[strategy] = 0
        return sizes
    
    def next(self):
        """Execute portfolio strategy"""
        signals = self.get_strategy_signals()
        total_value = self.broker.getvalue()
        position_sizes = self.calculate_position_sizes(total_value)
        
        for strategy_name, signal in signals.items():
            current_pos = self.strategy_positions[strategy_name]
            target_size = position_sizes[strategy_name]
            
            if signal == 'BUY' and current_pos < target_size:
                # Buy more shares for this strategy
                shares_to_buy = target_size - current_pos
                self.buy(size=shares_to_buy)
                self.strategy_positions[strategy_name] = target_size
                
            elif signal == 'SELL' and current_pos > 0:
                # Sell all shares for this strategy
                self.sell(size=current_pos)
                self.strategy_positions[strategy_name] = 0

# Run portfolio strategy
cerebro = bt.Cerebro()
cerebro.addstrategy(PortfolioStrategy)

# Add multiple data feeds for different stocks
stocks = ['600519', '000001', '000858']
for stock in stocks:
    data = get_data(stock)
    feed = bt.feeds.PandasData(dataname=data)
    cerebro.adddata(feed)

cerebro.broker.setcash(500000.0)  # Larger portfolio
cerebro.run()
```

### 5. Parameter Optimization

```python
import backtrader as bt
import itertools
import pandas as pd

def optimize_strategy_parameters():
    """Optimize strategy parameters using grid search"""
    
    # Parameter ranges to test
    fast_periods = [5, 10, 15, 20]
    slow_periods = [20, 30, 40, 50]
    
    results = []
    
    for pfast, pslow in itertools.product(fast_periods, slow_periods):
        if pfast >= pslow:  # Skip invalid combinations
            continue
        
        # Create cerebro instance
        cerebro = bt.Cerebro()
        
        # Add strategy with parameters
        cerebro.addstrategy(SmaCross, pfast=pfast, pslow=pslow)
        
        # Add data
        data = get_data("600519")
        feed = bt.feeds.PandasData(dataname=data)
        cerebro.adddata(feed)
        
        # Set broker
        cerebro.broker.setcash(100000.0)
        cerebro.broker.setcommission(commission=0.001)
        
        # Add analyzers
        cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
        cerebro.addanalyzer(bt.analyzers.Returns, _name='returns')
        
        try:
            # Run backtest
            result = cerebro.run()
            strategy = result[0]
            
            # Extract metrics
            sharpe = strategy.analyzers.sharpe.get_analysis()
            returns = strategy.analyzers.returns.get_analysis()
            
            results.append({
                'pfast': pfast,
                'pslow': pslow,
                'final_value': cerebro.broker.getvalue(),
                'sharpe_ratio': sharpe.get('sharperatio', 0),
                'total_return': returns.get('rtot', 0)
            })
            
            print(f"pfast={pfast}, pslow={pslow}: Sharpe={sharpe.get('sharperatio', 0):.3f}")
            
        except Exception as e:
            print(f"Error with pfast={pfast}, pslow={pslow}: {e}")
    
    # Convert to DataFrame and find best parameters
    df_results = pd.DataFrame(results)
    best_params = df_results.loc[df_results['sharpe_ratio'].idxmax()]
    
    print("\nBest Parameters:")
    print(best_params)
    
    return df_results

# Run optimization
optimization_results = optimize_strategy_parameters()
```

### 6. Fund Investment Strategy

```python
from pyfunds.fund_strategies.src.utils.fund_stragegy.index import InvestmentStrategy
import json

# Load fund data (example format)
fund_data = {
    "name": "华夏沪深300ETF",
    "code": "510300",
    "all": {
        "2020-01-01": {"val": 4.123, "date": "2020-01-01"},
        "2020-01-02": {"val": 4.156, "date": "2020-01-02"},
        # ... more data
    },
    "bonus": {
        "2020-06-15": {"val": 4.200, "bonus": 0.05, "date": "2020-06-15"}
    }
}

# Create investment strategy
strategy = InvestmentStrategy({
    'fundJson': fund_data,
    'totalAmount': 50000,  # Initial capital
    'salary': 8000,        # Monthly salary
    'stop': {
        'rate': 0.05,      # Take profit at 5%
        'minAmount': 0.1   # Minimum position 10%
    },
    'tInvest': {
        'rate': 0.03,      # Buy dip at 3% decline
        'amount': 2000     # DCA amount
    },
    'shangZhengData': {},  # Index data for comparison
    'onEachDay': lambda date: print(f"Processing: {date}")
})

# Execute dollar-cost averaging strategy
strategy.fixedInvest({
    'fixedInvestment': {
        'amount': 3000,    # Invest 3000 monthly
        'dateOrWeek': 15,  # On 15th of each month
        'period': 'monthly'
    },
    'range': ['2020-01-01', '2023-01-01']
})

# Get performance results
performance = strategy.annualizedRate
print(f"Fund Growth Rate: {performance['fundGrowth']:.2%}")
print(f"Total Profit Rate: {performance['totalProfit']:.2%}")

# Print investment summary
total_data = len(strategy.data)
if total_data > 0:
    final_snapshot = strategy.data[-1]
    print(f"Final Portfolio Value: {final_snapshot.totalAmount:.2f}")
    print(f"Total Investment: {final_snapshot.totalBuyAmount:.2f}")
    print(f"Total Profit: {final_snapshot.accumulatedProfit:.2f}")
```

### 7. Risk Management Example

```python
class RiskManagedStrategy(bt.Strategy):
    """Strategy with comprehensive risk management"""
    
    params = (
        ('max_drawdown', 0.15),      # 15% max drawdown
        ('risk_per_trade', 0.02),    # 2% risk per trade
        ('max_positions', 3),        # Max 3 positions
        ('stop_loss_atr', 2.0),      # 2 ATR stop loss
    )
    
    def __init__(self):
        self.sma_fast = bt.indicators.SMA(period=10)
        self.sma_slow = bt.indicators.SMA(period=30)
        self.atr = bt.indicators.ATR(period=14)
        self.rsi = bt.indicators.RSI(period=14)
        
        # Risk management variables
        self.peak_value = 0
        self.position_count = 0
        self.stop_orders = {}
        
    def check_drawdown(self):
        """Check if drawdown limit is exceeded"""
        current_value = self.broker.getvalue()
        self.peak_value = max(self.peak_value, current_value)
        
        if self.peak_value > 0:
            drawdown = (self.peak_value - current_value) / self.peak_value
            return drawdown < self.p.max_drawdown
        return True
    
    def calculate_position_size(self, entry_price):
        """Calculate position size based on risk management"""
        account_value = self.broker.getvalue()
        risk_amount = account_value * self.p.risk_per_trade
        
        # Stop loss based on ATR
        stop_loss = entry_price - (self.p.stop_loss_atr * self.atr[0])
        risk_per_share = entry_price - stop_loss
        
        if risk_per_share > 0:
            position_size = int(risk_amount / risk_per_share)
            return max(1, position_size)  # At least 1 share
        return 0
    
    def next(self):
        # Check risk limits
        if not self.check_drawdown():
            self.log("Drawdown limit exceeded - no new positions")
            return
        
        if self.position_count >= self.p.max_positions:
            self.log("Maximum positions reached")
            return
        
        # Entry signal
        if (self.sma_fast[0] > self.sma_slow[0] and 
            self.sma_fast[-1] <= self.sma_slow[-1] and
            self.rsi[0] > 50):
            
            entry_price = self.data.close[0]
            position_size = self.calculate_position_size(entry_price)
            
            if position_size > 0:
                # Place buy order
                buy_order = self.buy(size=position_size)
                
                # Place stop loss order
                stop_price = entry_price - (self.p.stop_loss_atr * self.atr[0])
                stop_order = self.sell(
                    size=position_size,
                    exectype=bt.Order.Stop,
                    price=stop_price
                )
                
                self.stop_orders[buy_order] = stop_order
                self.position_count += 1
                
                self.log(f'BUY {position_size} shares at {entry_price:.2f}, Stop: {stop_price:.2f}')
    
    def notify_order(self, order):
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(f'BUY EXECUTED: {order.executed.price:.2f}')
            else:
                self.log(f'SELL EXECUTED: {order.executed.price:.2f}')
                self.position_count = max(0, self.position_count - 1)
    
    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print(f'{dt.isoformat()}, {txt}')

# Run risk-managed strategy
cerebro = bt.Cerebro()
cerebro.addstrategy(RiskManagedStrategy)

data = get_data("600519")
feed = bt.feeds.PandasData(dataname=data)
cerebro.adddata(feed)

cerebro.broker.setcash(100000.0)
cerebro.broker.setcommission(commission=0.001)

results = cerebro.run()
cerebro.plot()
```

### 8. Live Trading Integration

```python
import time
import threading
from datetime import datetime
import queue

class LiveTradingBot:
    """Live trading bot with paper trading simulation"""
    
    def __init__(self, initial_cash=100000):
        self.cash = initial_cash
        self.positions = {}
        self.orders = queue.Queue()
        self.running = False
        
    def start(self):
        """Start the trading bot"""
        self.running = True
        
        # Start data feed thread
        data_thread = threading.Thread(target=self._data_feed_loop)
        data_thread.start()
        
        # Start order processing thread
        order_thread = threading.Thread(target=self._order_processing_loop)
        order_thread.start()
        
        print("Trading bot started...")
    
    def stop(self):
        """Stop the trading bot"""
        self.running = False
        print("Trading bot stopped.")
    
    def _data_feed_loop(self):
        """Continuously fetch market data and generate signals"""
        while self.running:
            try:
                # Get real-time data
                symbol = '600519'
                current_data = self._get_realtime_data(symbol)
                
                if current_data:
                    # Generate trading signals
                    signal = self._generate_signal(symbol, current_data)
                    
                    if signal and signal != 'HOLD':
                        # Place order
                        order = {
                            'symbol': symbol,
                            'action': signal,
                            'quantity': 100,
                            'price': current_data['price'],
                            'timestamp': datetime.now()
                        }
                        self.orders.put(order)
                
                time.sleep(60)  # Check every minute
                
            except Exception as e:
                print(f"Error in data feed: {e}")
                time.sleep(60)
    
    def _order_processing_loop(self):
        """Process pending orders"""
        while self.running:
            try:
                # Get order from queue (wait up to 1 second)
                order = self.orders.get(timeout=1)
                
                # Execute order
                self._execute_order(order)
                
            except queue.Empty:
                continue
            except Exception as e:
                print(f"Error processing order: {e}")
    
    def _get_realtime_data(self, symbol):
        """Get real-time market data"""
        try:
            stock_data = ts.get_realtime_quotes(symbol)
            return {
                'price': float(stock_data['price'].iloc[0]),
                'volume': float(stock_data['volume'].iloc[0]),
                'high': float(stock_data['high'].iloc[0]),
                'low': float(stock_data['low'].iloc[0])
            }
        except:
            return None
    
    def _generate_signal(self, symbol, current_data):
        """Generate trading signal based on current data"""
        # Simplified signal generation
        # In practice, this would use more sophisticated logic
        
        # Get historical data for indicators
        try:
            hist_data = ts.get_k_data(symbol, ktype='D')
            if len(hist_data) < 20:
                return 'HOLD'
            
            # Calculate simple moving averages
            ma_short = hist_data['close'].rolling(5).mean().iloc[-1]
            ma_long = hist_data['close'].rolling(20).mean().iloc[-1]
            current_price = current_data['price']
            
            # Simple crossover strategy
            if current_price > ma_short > ma_long:
                return 'BUY'
            elif current_price < ma_short < ma_long:
                return 'SELL'
            else:
                return 'HOLD'
                
        except Exception as e:
            print(f"Error generating signal: {e}")
            return 'HOLD'
    
    def _execute_order(self, order):
        """Execute trading order (paper trading)"""
        symbol = order['symbol']
        action = order['action']
        quantity = order['quantity']
        price = order['price']
        
        if action == 'BUY':
            cost = quantity * price
            if self.cash >= cost:
                self.cash -= cost
                if symbol in self.positions:
                    self.positions[symbol] += quantity
                else:
                    self.positions[symbol] = quantity
                
                print(f"BUY: {quantity} shares of {symbol} at {price:.2f}")
                print(f"Remaining cash: {self.cash:.2f}")
            else:
                print(f"Insufficient cash for BUY order: {symbol}")
        
        elif action == 'SELL':
            if symbol in self.positions and self.positions[symbol] >= quantity:
                self.positions[symbol] -= quantity
                self.cash += quantity * price
                
                if self.positions[symbol] == 0:
                    del self.positions[symbol]
                
                print(f"SELL: {quantity} shares of {symbol} at {price:.2f}")
                print(f"Current cash: {self.cash:.2f}")
            else:
                print(f"Insufficient shares for SELL order: {symbol}")
    
    def get_portfolio_value(self):
        """Calculate total portfolio value"""
        total_value = self.cash
        
        for symbol, quantity in self.positions.items():
            try:
                current_data = self._get_realtime_data(symbol)
                if current_data:
                    total_value += quantity * current_data['price']
            except:
                pass
        
        return total_value
    
    def print_status(self):
        """Print current portfolio status"""
        print(f"\n=== Portfolio Status ===")
        print(f"Cash: {self.cash:.2f}")
        print(f"Positions: {self.positions}")
        print(f"Total Value: {self.get_portfolio_value():.2f}")
        print("========================\n")

# Example usage
if __name__ == "__main__":
    # Create and start trading bot
    bot = LiveTradingBot(initial_cash=100000)
    bot.start()
    
    # Monitor for 1 hour, then stop
    try:
        for i in range(60):  # 60 minutes
            time.sleep(60)
            bot.print_status()
    except KeyboardInterrupt:
        print("Stopping bot...")
    finally:
        bot.stop()
```

## Common Use Cases

### 1. Educational Backtesting
- Learn quantitative trading concepts
- Test different indicators and strategies
- Understand risk-return relationships

### 2. Strategy Development
- Develop and validate new trading strategies
- Compare performance across different markets
- Optimize parameters for better returns

### 3. Portfolio Management
- Implement multi-asset strategies
- Balance risk across different positions
- Monitor and rebalance portfolios

### 4. Research and Analysis
- Analyze market patterns and trends
- Test academic trading theories
- Generate research reports

### 5. Automated Trading
- Deploy strategies in live markets
- Implement risk management rules
- Monitor and alert on market conditions

## Best Practices

1. **Always backtest thoroughly** before live trading
2. **Implement proper risk management** (stop losses, position sizing)
3. **Use out-of-sample testing** to validate strategies
4. **Start with paper trading** before real money
5. **Monitor and log all activities** for analysis
6. **Keep strategies simple and robust**
7. **Regular performance review** and strategy adjustment

## Troubleshooting

### Common Issues

1. **Data Feed Problems**: Check TuShare API limits and network connectivity
2. **Strategy Errors**: Validate indicator calculations and logic
3. **Performance Issues**: Optimize code and reduce memory usage
4. **GUI Problems**: Check wxPython installation and display settings

### Getting Help

- Check the [FAQ](https://ufund-me.github.io/Qbot/#/04-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98/FQA)
- Join the WeChat community group
- Submit issues on [GitHub](https://github.com/UFund-Me/Qbot/issues)
- Read the comprehensive documentation

---

*These examples provide a solid foundation for using Qbot. Modify and extend them based on your specific trading requirements and risk tolerance.*