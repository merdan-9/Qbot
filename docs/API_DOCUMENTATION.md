# Qbot API Documentation

## Overview

Qbot is an AI-powered quantitative trading platform that provides comprehensive tools for algorithmic trading, backtesting, and portfolio management. This documentation covers the main functions, components, and usage patterns.

## Table of Contents

1. [Core Components](#core-components)
2. [Main Entry Points](#main-entry-points)
3. [Trading Strategies](#trading-strategies)
4. [GUI Components](#gui-components)
5. [Utility Functions](#utility-functions)
6. [Backtesting Framework](#backtesting-framework)
7. [Frontend Components](#frontend-components)
8. [Usage Examples](#usage-examples)

## Core Components

### 1. Main Application Entry Points

#### main.py
**Purpose**: GUI application launcher for the Qbot platform

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import wx
from qbot.gui.mainframe import MainFrame

if __name__ == "__main__":
    app = wx.App()
    frame = MainFrame(None, title="AI智能量化投研平台")
    frame.Show()
    app.MainLoop()
```

**Usage**:
```bash
python main.py  # On Mac: pythonw main.py
```

#### qbot_main.py
**Purpose**: Real-time trading signal generation and multi-strategy fusion system

**Key Features**:
- Real-time stock data monitoring
- Multiple technical indicator calculation
- Signal fusion and decision making
- Risk management with 16% loss threshold

**Core Functions**:

```python
def send_signal_sounds(type="buy"):
    """
    Play audio alerts for trading signals
    
    Args:
        type (str): Signal type - "buy" or "sell"
    """

def send_signal_message_screen(symbol, price, type=default):
    """
    Display desktop notifications for trading signals
    
    Args:
        symbol (str): Stock symbol
        price (float): Current price
        type (str): Signal type
    """

def cal_fusion_result(signals):
    """
    Calculate fusion result from multiple trading signals
    
    Args:
        signals (list): List of signal dictionaries
        
    Returns:
        float: Weighted fusion score
    """

def get_weights_distribution(data):
    """
    Calculate mean weight distribution for signal fusion
    
    Args:
        data (dict): Weight configuration dictionary
        
    Returns:
        float: Mean weight value
    """
```

**Configuration**:
```python
# Stock pool configuration
stocks_pool = [
    {"code": "sz000063", "name": "中兴通讯", "min_threshold": "26", "max_threshold": "38"},
    {"code": "sh000016", "name": "上证50"},
    {"code": "601318", "name": "中国平安"},
]

# Strategy weights
default_weights = {
    "BIAS": 0.1, 
    "KDJ": 0.2, 
    "RSI": 0.15, 
    "BOLL": 0.25, 
    "MACD": 0.2, 
    "LSTM": 0.1
}

# Broker configuration
broker_config = [{
    "setcash": 100000, 
    "ballance": 100000, 
    "stake": 100, 
    "commission": 0.0005
}]
```

### 2. Core Trading Module (qbot.py)

**Purpose**: Core quantitative trading logic with multi-factor strategy implementation

**Key Functions**:

```python
def get_data(code, start="2023-02-01", end="2023-03-21"):
    """
    Retrieve stock data from TuShare
    
    Args:
        code (str): Stock code
        start (str): Start date (YYYY-MM-DD)
        end (str): End date (YYYY-MM-DD)
        
    Returns:
        pd.DataFrame: Stock data with OHLCV columns
    """
```

**Technical Indicators Calculated**:
- Moving Averages (MA5, MA10, MA20)
- BIAS indicators (BIAS1, BIAS2, BIAS3)
- Bollinger Bands (upper, middle, lower)
- Stochastic Oscillator (K, D lines)
- Relative Strength Index (RSI)

**Strategy Implementation**:
```python
# Strategy 1: Moving Average and BIAS signals
condition1 = (data['ma5'] > data['ma10']) & \
             (data['ma5'] > data['ma20']) & \
             (data['bias1'] > data['bias2']) & \
             (data['bias1'] > data['bias3'])
signals += 0.2 * condition1.astype(int)

# Strategy 2: Bollinger Bands breakout
condition2 = (data['close'] < data['lower'])
signals += 0.3 * condition2.astype(int)

# Strategy 3: Stochastic crossover
condition3 = (data['k'] > data['d']) & \
             (data['k'].shift() < data['d'].shift())
signals += 0.2 * condition3.astype(int)

# Strategy 4: RSI overbought/oversold
condition4 = (data['rsi'] > 80) | (data['rsi'] < 20)
signals += 0.3 * condition4.astype(int)
```

## Trading Strategies

### 1. Simple Moving Average Crossover Strategy

**File**: `qbot/strategies/sma_cross_strategy_bt.py`

```python
class SmaCross(bt.Strategy):
    """
    Simple Moving Average Crossover Strategy
    
    Parameters:
        pfast (int): Fast moving average period (default: 10)
        pslow (int): Slow moving average period (default: 30)
    """
    params = (('pfast', 10), ('pslow', 30),)

    def __init__(self):
        sma1 = btind.SMA(period=self.p.pfast)
        sma2 = btind.SMA(period=self.p.pslow)
        self.crossover = btind.CrossOver(sma1, sma2)

    def next(self):
        """
        Execute trading logic on each bar
        
        Logic:
        - Buy when fast MA crosses above slow MA
        - Sell when fast MA crosses below slow MA
        - Invest 95% of available cash
        """
        if self.position.size == 0:
            if self.crossover > 0:
                amount_to_invest = (self.broker.cash * 0.95)
                self.size = int(amount_to_invest / self.data.close)
                self.buy(size=self.size)
        elif self.position.size > 0:
            if self.crossover < 0:
                self.close()
```

**Usage Example**:
```python
# Create Cerebro engine
cerebro = bt.Cerebro()
cerebro.addstrategy(SmaCross)

# Add data
data = bt.feeds.PandasData(dataname=dataframe, fromdate=start, todate=end)
cerebro.adddata(data)

# Set initial capital and commission
cerebro.broker.setcash(10000.0)
cerebro.broker.setcommission(commission=0.001)

# Add analyzer
cerebro.addanalyzer(btanalyzers.SharpeRatio, _name='mysharpe')

# Run backtest
results = cerebro.run()
cerebro.plot()
```

### 2. Multi-Strategy Implementation

**File**: `qbot/strategies/multi_strategy_bt.py`

```python
class MultiStrategy(bt.Strategy):
    """
    Multi-factor trading strategy combining SMA and RSI
    
    Parameters:
        exitbars (int): Exit after N bars (default: 5)
        maperiod (int): Moving average period (default: 15)
    """
    params = (('exitbars', 5), ('maperiod', 15),)

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.sma = btind.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod
        )
        self.rsi = btind.RelativeStrengthIndex()

    def next(self):
        """
        Multi-factor trading logic:
        - Buy: RSI crosses above 50 AND price > SMA
        - Sell: RSI crosses below 50 AND price < SMA
        """
        if not self.position:
            if (self.rsi[0] < 50 and self.rsi[-1] >= 50 and 
                self.dataclose[0] > self.sma[0]):
                self.order = self.buy()
        else:
            if (self.rsi[0] > 50 and self.rsi[-1] <= 50 and 
                self.dataclose[0] < self.sma[0]):
                self.order = self.sell()
```

### 3. LSTM Strategy Implementation

**File**: `qbot/strategies/lstm_strategy_bt.py`

```python
class LSTMPredict(bt.Strategy):
    """
    LSTM-based prediction strategy using Keras
    
    Parameters:
        period (int): Prediction period (default: 10)
        neurons (int): LSTM neurons (default: 50)
        train_size (float): Training data ratio (default: 0.8)
        lookback (int): Lookback window (default: 20)
    """
    params = (('period', 10), ('neurons', 50), 
              ('train_size', 0.8), ('lookback', 20))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.scaler = MinMaxScaler(feature_range=(0, 1))
        self.lookback = self.p.lookback
        self.train_size = self.p.train_size
        self.train_data, self.test_data = self._prepare_data()
        self.model = self._build_model()
        self.model.fit(self.train_data['X'], self.train_data['Y'], 
                      epochs=50, batch_size=1, verbose=2)

    def _build_model(self):
        """
        Build LSTM model architecture
        
        Returns:
            keras.Model: Compiled LSTM model
        """
        model = Sequential()
        model.add(LSTM(self.p.neurons, return_sequences=True, 
                      input_shape=(self.lookback, 1)))
        model.add(Dropout(0.2))
        model.add(LSTM(self.p.neurons, return_sequences=True))
        model.add(Dropout(0.2))
        model.add(LSTM(self.p.neurons))
        model.add(Dropout(0.2))
        model.add(Dense(1))
        model.compile(optimizer='adam', loss='mean_squared_error')
        return model

    def next(self):
        """
        LSTM prediction-based trading logic
        """
        if len(self.data) > self.lookback:
            recent_data = np.array(self.dataclose.get(size=self.lookback))
            recent_data = self.scaler.transform(recent_data.reshape(-1, 1))
            recent_data = recent_data.reshape(1, self.lookback, 1)
            
            predicted_price = self.model.predict(recent_data)[0][0]
            predicted_price = self.scaler.inverse_transform([[predicted_price]])[0][0]
            
            current_price = self.dataclose[0]
            
            if predicted_price > current_price * 1.02:  # 2% threshold
                if not self.position:
                    self.buy()
            elif predicted_price < current_price * 0.98:  # -2% threshold
                if self.position:
                    self.sell()
```

## GUI Components

### MainFrame Class

**File**: `qbot/gui/mainframe.py`

```python
class MainFrame(wx.Frame):
    """
    Main GUI frame for Qbot application
    
    Features:
    - Tabbed interface for different functionalities
    - Menu bar with configuration options
    - Status bar for application state
    """
    
    def __init__(self, *args, **kw):
        """
        Initialize main frame
        
        Sets up:
        - Window size (75% of screen width, 70% of screen height)
        - Application icon
        - Status bar, menu bar, and main tabs
        """
        displaySize = wx.DisplaySize()
        displaySize = 0.75 * displaySize[0], 0.70 * displaySize[1]
        super().__init__(parent=None, 
                        title="Qbot - AI智能量化投研平台", 
                        size=displaySize)
        
        # Set application icon
        icon_file = "qbot/gui/imgs/logo.ico"
        icon = wx.Icon(icon_file, wx.BITMAP_TYPE_ICO)
        self.SetIcon(icon)
        
        self.init_statusbar()
        self.init_menu_bar()
        self.init_main_tabs()

    def init_menu_bar(self):
        """Initialize menu bar with settings and configuration options"""
        
    def init_main_tabs(self):
        """Initialize main tabbed interface with different panels"""
        
    def init_statusbar(self):
        """Initialize status bar for displaying application state"""
```

## Utility Functions

### BaseService Class

**File**: `utils/common/BaseService.py`

```python
class BaseService(object):
    """
    Base service class providing common functionality
    
    Features:
    - Logging setup
    - Path management
    - HTTP request handling
    - Date/time utilities
    """
    
    def __init__(self, logfile="default.log"):
        """
        Initialize base service
        
        Args:
            logfile (str): Log file name
        """
        self.logger = logger
        self.logger.add(logfile)
        self.init_const_data()
        self.params = None
        self.cookies = None

    def init_const_data(self):
        """Initialize common data like current date"""
        self.today = datetime.datetime.now().strftime("%Y-%m-%d")

    def check_path(self, path):
        """
        Check and create directory path if not exists
        
        Args:
            path (str): Directory path to check
        """
        if not os.path.exists(path):
            try:
                os.makedirs(path)
            except Exception as e:
                self.logger.error(e)

    def get_url_filename(self, url):
        """
        Extract filename from URL
        
        Args:
            url (str): URL string
            
        Returns:
            str: Extracted filename
        """
        # Implementation details...
```

### LarkBot Integration

**File**: `utils/larkbot.py`

```python
class LarkBot:
    """
    Lark (Feishu) bot integration for notifications
    
    Features:
    - Send trading alerts
    - Rich message formatting
    - Webhook integration
    """
    
    def __init__(self, secret=None):
        """
        Initialize Lark bot
        
        Args:
            secret (str): Webhook secret for authentication
        """
        self.secret = secret
        self.webhook_url = WEBHOOK_URL

    def send(self, content, msg_type="text"):
        """
        Send message via Lark webhook
        
        Args:
            content (str): Message content
            msg_type (str): Message type ("text", "interactive")
            
        Returns:
            bool: Success status
        """
        # Implementation with signature generation and HTTP request
```

## Backtesting Framework

### Trade Management

**File**: `pyfunds/backtest/xalpha/trade.py`

```python
def xirrcal(cftable, trades, date, startdate=None, guess=0.01):
    """
    Calculate the XIRR (Extended Internal Rate of Return)
    
    Args:
        cftable (pd.DataFrame): Cash flow table with date and cash columns
        trades (list): List of trade objects
        date (str/datetime): Date when all positions are virtually sold
        startdate (str/datetime, optional): Start date for calculation
        guess (float): Initial guess for XIRR calculation
        
    Returns:
        float: XIRR rate as decimal
        
    Example:
        >>> cftable = pd.DataFrame({
        ...     'date': ['2020-01-01', '2020-06-01', '2020-12-31'],
        ...     'cash': [-10000, -5000, 15000]
        ... })
        >>> xirr_rate = xirrcal(cftable, [], '2020-12-31')
        >>> print(f"XIRR: {xirr_rate:.2%}")
    """
```

## Frontend Components

### TypeScript Utilities

**File**: `pytrader/frontend/src/utils/tools.ts`

```typescript
/**
 * Sleep function for async operations
 * @param time Sleep duration in milliseconds
 */
export async function sleep(time: number): Promise<void> {
    await new Promise(resolve => {
        setTimeout(() => resolve, time)
    })
}

/**
 * Format currency amount
 * @param num Amount to format
 * @param symbol Currency symbol (default: '￥')
 * @returns Formatted currency string
 * 
 * @example
 * format(1234.56) // "￥1,234.56"
 * format(1234.56, '$') // "$1,234.56"
 */
export function format(num: number | string, symbol = '￥'): string {
    if(Number.isNaN(Number(num))) return `${symbol}0.00`
    return symbol + (Number(num).toFixed(2))
        .replace(/(\d)(?=(\d{3})+\.)/g, '$1,')
}

/**
 * Remove currency formatting
 * @param str Formatted currency string
 * @returns Numeric value or empty string
 */
export function unformat(str: string): number | string {
    const s = str.substr(1).replace(/\,/g, '')
    return Number.isNaN(Number(s)) || Number(s) === 0 ? '' : Number(s)
}

/**
 * Calculate table summary row
 * @param param Object containing columns and data
 * @returns Array of summary values
 */
export function tableSummaries(param: { columns: any; data: any }): Array<string | number> {
    const { columns, data } = param
    const sums: Array<string | number> = []
    
    columns.forEach((column: { property: string | number }, index: number) => {
        if (index === 0) {
            sums[index] = '合计'
            return
        }
        
        const values = data.map((item: { [x: string]: any }) => Number(item[column.property]))
        if (!values.every((value: number) => isNaN(value))) {
            sums[index] = values.reduce((prev: number, curr: number) => {
                const value = Number(curr)
                if (!isNaN(value)) {
                    return prev + curr
                } else {
                    return prev
                }
            }, 0)
        } else {
            sums[index] = ''
        }
    })
    
    return sums
}
```

### Fund Strategy Implementation

**File**: `pyfunds/fund-strategies/src/utils/fund-stragegy/index.ts`

```typescript
export interface FixedInvestOption {
    fixedInvestment: {
        amount: number // Investment amount per period
        dateOrWeek: number // Day of week/month for investment
        period: 'weekly' | 'monthly' // Investment frequency
    }, 
    range: [string | Date, string | Date] // Investment period
}

export class InvestmentStrategy {
    /**
     * Fund investment strategy implementation
     * 
     * Features:
     * - Dollar-cost averaging (DCA)
     * - Profit-taking strategies
     * - Risk management
     * - Performance tracking
     */
    
    totalAmount!: number // Initial capital
    salary!: number // Monthly salary addition
    
    /**
     * Calculate annualized returns
     */
    get annualizedRate(): {
        fundGrowth: number
        totalProfit: number
    } {
        const len = this.data.length
        if(len > 0) {
            const startFund = this.data[0]
            const endFund = this.data[len - 1]
            const rangeTime = new Date(endFund.date).getTime() - new Date(startFund.date).getTime()
            const rangeYear = rangeTime / ONE_DAY / 365
            
            return {
                fundGrowth: roundToFix(Math.pow(1 + endFund.fundGrowthRate, 1 / rangeYear) - 1, 4),
                totalProfit: roundToFix(Math.pow(1 + endFund.totalProfitRate, 1 / rangeYear) - 1, 4),
            }
        } else {
            return { fundGrowth: 0, totalProfit: 0 }
        }
    }
    
    /**
     * Execute fixed investment strategy
     * @param opt Investment options
     * @returns InvestmentStrategy instance for chaining
     */
    fixedInvest(opt: FixedInvestOption): InvestmentStrategy {
        const {range, fixedInvestment} = opt
        this.fixedConfig = opt
        
        const beginTime = new Date(range[0]).getTime()
        const endTime = new Date(range[1]).getTime()
        
        if(beginTime > endTime){
            throw new Error('range[1] should not less than range[0]')
        }
        
        let curDate = beginTime
        
        while(curDate <= endTime) {
            if(this.shouldFixedInvest(fixedInvestment, curDate)){
                this.buy(fixedInvestment.amount, curDate)
            } else {
                this.buy(0, curDate)
            }
            
            this.onEachDay(curDate)
            curDate += 24 * 60 * 60 * 1000
        }
        
        return this
    }
}
```

## Usage Examples

### 1. Basic Strategy Backtesting

```python
import backtrader as bt
import pandas as pd
import tushare as ts
from datetime import datetime

# Get data
def get_data(code, start="2020-01-01", end="2023-01-31"):
    df = ts.get_k_data(code, autype="qfq", start=start, end=end)
    df.index = pd.to_datetime(df.date)
    df["openinterest"] = 0
    df = df[["open", "high", "low", "close", "volume", "openinterest"]]
    return df

# Prepare data
dataframe = get_data("600018")
start = datetime(2020, 1, 1)
end = datetime(2021, 12, 31)

# Create Cerebro engine
cerebro = bt.Cerebro()
cerebro.addstrategy(SmaCross)

# Add data
data = bt.feeds.PandasData(dataname=dataframe, fromdate=start, todate=end)
cerebro.adddata(data)

# Set parameters
cerebro.broker.setcash(10000.0)
cerebro.broker.setcommission(commission=0.001)

# Add analyzers
cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')

# Run backtest
results = cerebro.run()
strategy = results[0]

# Print results
print(f'Sharpe Ratio: {strategy.analyzers.sharpe.get_analysis()["sharperatio"]:.4f}')
print(f'Max Drawdown: {strategy.analyzers.drawdown.get_analysis()["max"]["drawdown"]:.2f}%')

# Plot results
cerebro.plot()
```

### 2. Real-time Trading with Signal Fusion

```python
import tushare as ts
import talib
import pandas as pd
import numpy as np

# Configuration
symbol = '600519'
default_weights = {"BIAS": 0.1, "KDJ": 0.2, "RSI": 0.15, "BOLL": 0.25, "MACD": 0.2, "LSTM": 0.1}

# Get real-time data
stock_data = ts.get_realtime_quotes(symbol)
latest_price = float(stock_data['price'].iloc[0])

# Calculate indicators
close_prices = data['close']
ma_short_data = talib.SMA(close_prices, timeperiod=5)
ma_mid_data = talib.SMA(close_prices, timeperiod=10)
ma_long_data = talib.SMA(close_prices, timeperiod=20)

upper, middle, lower = talib.BBANDS(close_prices, timeperiod=20)
rsi = talib.RSI(close_prices, timeperiod=14)
kdj_k, kdj_d = talib.STOCH(data['high'], data['low'], data['close'])

# Generate signals
buy_signals = []
sell_signals = []

# BIAS strategy
if (bias1.iloc[-1] > bias2.iloc[-1] and bias1.iloc[-1] > bias3.iloc[-1]):
    buy_signals.append([{"strategy": "BIAS", "symbol": symbol, "price": latest_price}])

# BOLL strategy
if latest_price < lower.iloc[-1]:
    buy_signals.append([{"strategy": "BOLL", "symbol": symbol, "price": latest_price}])

# Calculate fusion result
if cal_fusion_result(buy_signals) > get_weights_distribution(default_weights):
    print(f"BUY SIGNAL: {symbol} at {latest_price}")
elif cal_fusion_result(sell_signals) > get_weights_distribution(default_weights):
    print(f"SELL SIGNAL: {symbol} at {latest_price}")
else:
    print("HOLD")
```

### 3. Fund Investment Strategy

```typescript
import { InvestmentStrategy } from './fund-strategy'
import FundDataJson from './fund-data.json'

// Create investment strategy
const strategy = new InvestmentStrategy({
    fundJson: FundDataJson,
    totalAmount: 50000, // Initial capital
    salary: 8000, // Monthly salary
    stop: {
        rate: 0.05, // Take profit at 5%
        minAmount: 0.1 // Minimum position 10%
    },
    tInvest: {
        rate: 0.03, // Buy dip at 3% decline
        amount: 2000 // DCA amount
    },
    shangZhengData: indexData,
    onEachDay: (date) => {
        console.log(`Processing date: ${new Date(date).toDateString()}`)
    }
})

// Execute dollar-cost averaging
strategy.fixedInvest({
    fixedInvestment: {
        amount: 2000, // Invest 2000 every month
        dateOrWeek: 15, // On 15th of each month
        period: 'monthly'
    },
    range: ['2020-01-01', '2023-01-01']
})

// Get performance metrics
const performance = strategy.annualizedRate
console.log(`Fund Growth: ${(performance.fundGrowth * 100).toFixed(2)}%`)
console.log(`Total Profit: ${(performance.totalProfit * 100).toFixed(2)}%`)
```

### 4. GUI Application Usage

```python
import wx
from qbot.gui.mainframe import MainFrame

# Create application
app = wx.App()

# Create main frame
frame = MainFrame(None, title="AI智能量化投研平台")

# Show frame
frame.Show()

# Start main loop
app.MainLoop()
```

## Configuration

### Environment Setup

1. **Python Dependencies**:
```bash
pip install -r requirements.txt
```

2. **Environment Variables**:
```bash
export PYTHONPATH=${PYTHONPATH}:$(pwd):$(pwd)/backend/multi-fact/mfm_learner
```

3. **Data Sources**:
- TuShare for Chinese stock data
- Custom data feeds support
- Real-time data integration

### Broker Configuration

```python
# Backtrader broker setup
cerebro.broker.setcash(100000.0)  # Initial capital
cerebro.broker.setcommission(commission=0.0005)  # 0.05% commission

# Real trading broker configuration
broker_config = [{
    "setcash": 100000,
    "balance": 100000,
    "stake": 100,
    "commission": 0.0005
}]
```

### Risk Management

```python
# Stop loss configuration
if broker_config[0]["balance"] < broker_config[0]["setcash"] * 0.84:
    print("⚠️ Loss exceeds 16%, stopping trading.")
    exit()

# Position sizing
amount_to_invest = broker.cash * 0.95  # Use 95% of available cash
size = int(amount_to_invest / current_price)
```

## Performance Metrics

### Available Analyzers

- **Sharpe Ratio**: Risk-adjusted returns
- **Maximum Drawdown**: Largest peak-to-trough decline
- **Win Rate**: Percentage of profitable trades
- **Profit Factor**: Gross profit / Gross loss
- **Calmar Ratio**: Annual return / Max drawdown

### Custom Metrics

```python
# Calculate custom performance metrics
def calculate_metrics(strategy_results):
    """
    Calculate comprehensive performance metrics
    
    Args:
        strategy_results: Backtrader strategy results
        
    Returns:
        dict: Performance metrics dictionary
    """
    total_trades = len(strategy_results.trades)
    winning_trades = len([t for t in strategy_results.trades if t.pnl > 0])
    
    return {
        'total_trades': total_trades,
        'win_rate': winning_trades / total_trades if total_trades > 0 else 0,
        'avg_trade': np.mean([t.pnl for t in strategy_results.trades]),
        'sharpe_ratio': strategy_results.analyzers.sharpe.get_analysis()['sharperatio'],
        'max_drawdown': strategy_results.analyzers.drawdown.get_analysis()['max']['drawdown']
    }
```

## Error Handling

### Common Issues and Solutions

1. **Data Issues**:
```python
try:
    data = ts.get_k_data(code, start=start, end=end)
    if data.empty:
        raise ValueError(f"No data available for {code}")
except Exception as e:
    logger.error(f"Data retrieval error: {e}")
    # Fallback to cached data or alternative source
```

2. **Strategy Errors**:
```python
def next(self):
    try:
        # Strategy logic here
        pass
    except Exception as e:
        self.log(f"Strategy error: {e}")
        # Implement fallback logic
```

3. **GUI Errors**:
```python
try:
    app = wx.App()
    frame = MainFrame(None, title="AI智能量化投研平台")
    frame.Show()
    app.MainLoop()
except Exception as e:
    print(f"GUI initialization error: {e}")
    # Run in headless mode or show error dialog
```

## Contributing

### Code Style Guidelines

1. **Python**: Follow PEP 8 standards
2. **TypeScript**: Use ESLint configuration
3. **Documentation**: Include docstrings for all functions
4. **Testing**: Write unit tests for new features

### Adding New Strategies

1. Create strategy class inheriting from `bt.Strategy`
2. Implement `__init__` and `next` methods
3. Add parameter validation
4. Include comprehensive docstrings
5. Add unit tests and backtesting examples

## Support and Resources

- **Documentation**: [Online Docs](https://ufund-me.github.io/Qbot/#/)
- **GitHub Issues**: [Report Issues](https://github.com/UFund-Me/Qbot/issues)
- **Community**: Join WeChat group for discussions
- **Examples**: Check `docs/tutorials_code/` for more examples

---

*This documentation covers the core functionality of Qbot. For the latest updates and additional features, please refer to the project repository and online documentation.*