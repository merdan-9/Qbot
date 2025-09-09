# Qbot Component Reference

## Core Components Overview

This reference provides detailed documentation for all major components in the Qbot platform, including their APIs, configuration options, and usage patterns.

## Table of Contents

1. [Entry Points](#entry-points)
2. [Core Trading Engine](#core-trading-engine)
3. [Strategy Framework](#strategy-framework)
4. [GUI Components](#gui-components)
5. [Utility Services](#utility-services)
6. [Backtesting Framework](#backtesting-framework)
7. [Frontend Components](#frontend-components)
8. [Configuration](#configuration)

## Entry Points

### main.py - GUI Application Launcher

**Purpose**: Launch the wxPython-based GUI application

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

**Key Features**:
- Cross-platform GUI support (Windows, macOS, Linux)
- Automatic window sizing (75% width, 70% height)
- Icon and branding integration
- Exception handling for GUI initialization

**Usage**:
```bash
python main.py  # Linux/Windows
pythonw main.py  # macOS (recommended)
```

### qbot_main.py - Real-time Trading Engine

**Purpose**: Real-time market monitoring and signal generation

**Key Functions**:

#### `send_signal_sounds(type="buy")`
```python
def send_signal_sounds(type="buy"):
    """
    Play audio alerts for trading signals
    
    Args:
        type (str): Signal type - "buy" or "sell"
    
    Platform Support:
        - macOS: Uses afplay command
        - Linux: Uses play command (commented)
        - Windows: Requires additional audio library
    """
```

#### `send_signal_message_screen(symbol, price, type=default)`
```python
def send_signal_message_screen(symbol, price, type=default):
    """
    Display desktop notifications using pync
    
    Args:
        symbol (str): Stock symbol/name
        price (float): Current stock price
        type (str): Signal type for notification title
    
    Features:
        - Native macOS notifications
        - Custom icon support
        - Clickable notifications with URL
    """
```

#### `cal_fusion_result(signals)`
```python
def cal_fusion_result(signals):
    """
    Calculate weighted fusion score from multiple signals
    
    Args:
        signals (list): List of signal dictionaries containing:
            - strategy: Strategy name (BIAS, KDJ, RSI, BOLL, MACD, LSTM)
            - symbol: Stock symbol
            - values: Signal value
    
    Returns:
        float: Weighted fusion score based on default_weights
    
    Weight Configuration:
        default_weights = {
            "BIAS": 0.1,   # 10% weight
            "KDJ": 0.2,    # 20% weight  
            "RSI": 0.15,   # 15% weight
            "BOLL": 0.25,  # 25% weight
            "MACD": 0.2,   # 20% weight
            "LSTM": 0.1    # 10% weight
        }
    """
```

**Configuration Options**:
```python
# Stock pool configuration
stocks_pool = [
    {"code": "sz000063", "name": "中兴通讯", "min_threshold": "26", "max_threshold": "38"},
    {"code": "sh000016", "name": "上证50"},
    {"code": "601318", "name": "中国平安"},
]

# Technical indicator parameters
symbol = '600519'  # Stock code
ma_short = 5       # Short MA period
ma_mid = 10        # Medium MA period  
ma_long = 20       # Long MA period
boll_period = 20   # Bollinger Bands period

# Risk management
broker_config = [{
    "setcash": 100000,     # Initial capital
    "ballance": 100000,    # Current balance
    "stake": 100,          # Shares per trade
    "commission": 0.0005   # Commission rate (0.05%)
}]
```

## Core Trading Engine

### qbot.py - Multi-Factor Trading Logic

**Purpose**: Core quantitative trading implementation with technical analysis

**Key Functions**:

#### `get_data(code, start="2023-02-01", end="2023-03-21")`
```python
def get_data(code, start="2023-02-01", end="2023-03-21"):
    """
    Retrieve and format stock data from TuShare
    
    Args:
        code (str): Stock code (e.g., "600018")
        start (str): Start date in YYYY-MM-DD format
        end (str): End date in YYYY-MM-DD format
    
    Returns:
        pd.DataFrame: Formatted DataFrame with columns:
            - open: Opening price
            - high: Highest price
            - low: Lowest price  
            - close: Closing price
            - volume: Trading volume
            - openinterest: Open interest (set to 0)
    
    Data Processing:
        - Sets datetime index
        - Applies forward adjustment (autype="qfq")
        - Adds openinterest column for Backtrader compatibility
    """
```

**Technical Indicators Implemented**:

1. **Moving Averages**:
   ```python
   data['ma5'] = talib.MA(data['close'], timeperiod=5)
   data['ma10'] = talib.MA(data['close'], timeperiod=10)  
   data['ma20'] = talib.MA(data['close'], timeperiod=20)
   ```

2. **BIAS Indicators**:
   ```python
   data['bias1'] = (data['close'] - data['ma5']) / data['ma5'] * 100
   data['bias2'] = (data['close'] - data['ma10']) / data['ma10'] * 100
   data['bias3'] = (data['close'] - data['ma20']) / data['ma20'] * 100
   ```

3. **Bollinger Bands**:
   ```python
   data['upper'], data['middle'], data['lower'] = talib.BBANDS(
       data['close'], timeperiod=20, nbdevup=2, nbdevdn=2, matype=0
   )
   ```

4. **Stochastic Oscillator**:
   ```python
   data['k'], data['d'] = talib.STOCH(
       data['high'], data['low'], data['close'],
       fastk_period=9, slowk_period=3, slowk_matype=0,
       slowd_period=3, slowd_matype=0
   )
   ```

5. **RSI (Relative Strength Index)**:
   ```python
   data['rsi'] = talib.RSI(data['close'], timeperiod=14)
   ```

**Signal Generation Logic**:
```python
# Multi-strategy signal combination
signals = np.zeros(len(data))

# Strategy 1: MA and BIAS alignment (20% weight)
condition1 = (data['ma5'] > data['ma10']) & \
             (data['ma5'] > data['ma20']) & \
             (data['bias1'] > data['bias2']) & \
             (data['bias1'] > data['bias3'])
signals += 0.2 * condition1.astype(int)

# Strategy 2: Bollinger Bands breakout (30% weight)  
condition2 = (data['close'] < data['lower'])
signals += 0.3 * condition2.astype(int)

# Strategy 3: Stochastic crossover (20% weight)
condition3 = (data['k'] > data['d']) & \
             (data['k'].shift() < data['d'].shift())
signals += 0.2 * condition3.astype(int)

# Strategy 4: RSI extremes (30% weight)
condition4 = (data['rsi'] > 80) | (data['rsi'] < 20)
signals += 0.3 * condition4.astype(int)
```

## Strategy Framework

### Base Strategy Structure

All Qbot strategies inherit from Backtrader's `bt.Strategy` class:

```python
class BaseQbotStrategy(bt.Strategy):
    """Base class for all Qbot strategies"""
    
    # Common parameters
    params = (
        ('printlog', True),
        ('risk_per_trade', 0.02),
        ('max_positions', 5),
    )
    
    def __init__(self):
        """Initialize common indicators and variables"""
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None
        
    def log(self, txt, dt=None):
        """Logging function with timestamp"""
        if self.params.printlog:
            dt = dt or self.datas[0].datetime.date(0)
            print(f'{dt.isoformat()}, {txt}')
    
    def notify_order(self, order):
        """Handle order status updates"""
        if order.status in [order.Submitted, order.Accepted]:
            return
            
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(f'BUY EXECUTED, Price: {order.executed.price:.2f}, '
                        f'Cost: {order.executed.value:.2f}, '
                        f'Comm: {order.executed.comm:.2f}')
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log(f'SELL EXECUTED, Price: {order.executed.price:.2f}, '
                        f'Cost: {order.executed.value:.2f}, '
                        f'Comm: {order.executed.comm:.2f}')
                        
        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')
            
        self.order = None
    
    def notify_trade(self, trade):
        """Handle completed trades"""
        if not trade.isclosed:
            return
            
        self.log(f'OPERATION PROFIT, GROSS {trade.pnl:.2f}, NET {trade.pnlcomm:.2f}')
```

### Strategy Examples

#### 1. SMA Cross Strategy (`sma_cross_strategy_bt.py`)

```python
class SmaCross(bt.Strategy):
    """Simple Moving Average Crossover Strategy"""
    
    params = (('pfast', 10), ('pslow', 30),)

    def __init__(self):
        sma1 = btind.SMA(period=self.p.pfast)
        sma2 = btind.SMA(period=self.p.pslow)
        self.crossover = btind.CrossOver(sma1, sma2)

    def next(self):
        if self.position.size == 0:
            if self.crossover > 0:  # Golden cross
                amount_to_invest = (self.broker.cash * 0.95)
                self.size = int(amount_to_invest / self.data.close)
                self.buy(size=self.size)
        elif self.position.size > 0:
            if self.crossover < 0:  # Death cross
                self.close()
```

#### 2. Multi-Strategy (`multi_strategy_bt.py`)

```python
class MultiStrategy(bt.Strategy):
    """Multi-factor strategy combining SMA and RSI"""
    
    params = (
        ('exitbars', 5),
        ('maperiod', 15),
    )

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.sma = btind.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod
        )
        self.rsi = btind.RelativeStrengthIndex()

    def next(self):
        if not self.position:
            # Buy: RSI crosses above 50 AND price > SMA
            if (self.rsi[0] < 50 and self.rsi[-1] >= 50 and 
                self.dataclose[0] > self.sma[0]):
                self.order = self.buy()
        else:
            # Sell: RSI crosses below 50 AND price < SMA  
            if (self.rsi[0] > 50 and self.rsi[-1] <= 50 and 
                self.dataclose[0] < self.sma[0]):
                self.order = self.sell()
```

#### 3. LSTM Strategy (`lstm_strategy_bt.py`)

```python
class LSTMPredict(bt.Strategy):
    """LSTM Neural Network Prediction Strategy"""
    
    params = (
        ('period', 10), 
        ('neurons', 50), 
        ('train_size', 0.8), 
        ('lookback', 20)
    )

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.scaler = MinMaxScaler(feature_range=(0, 1))
        self.lookback = self.p.lookback
        self.train_size = self.p.train_size
        self.train_data, self.test_data = self._prepare_data()
        self.model = self._build_model()
        self._train_model()

    def _build_model(self):
        """Build LSTM model architecture"""
        model = Sequential([
            LSTM(self.p.neurons, return_sequences=True, 
                 input_shape=(self.lookback, 1)),
            Dropout(0.2),
            LSTM(self.p.neurons, return_sequences=True),
            Dropout(0.2), 
            LSTM(self.p.neurons),
            Dropout(0.2),
            Dense(1)
        ])
        model.compile(optimizer='adam', loss='mean_squared_error')
        return model
```

## GUI Components

### MainFrame (`qbot/gui/mainframe.py`)

**Purpose**: Main application window with tabbed interface

```python
class MainFrame(wx.Frame):
    """Main GUI frame for Qbot application"""
    
    def __init__(self, *args, **kw):
        """Initialize main frame with responsive sizing"""
        displaySize = wx.DisplaySize()
        displaySize = 0.75 * displaySize[0], 0.70 * displaySize[1]
        super().__init__(parent=None, 
                        title="Qbot - AI智能量化投研平台", 
                        size=displaySize)
        
        # Set application icon
        icon_file = "qbot/gui/imgs/logo.ico"
        icon = wx.Icon(icon_file, wx.BITMAP_TYPE_ICO)
        self.SetIcon(icon)
        
        # Initialize UI components
        self.init_statusbar()
        self.init_menu_bar()
        self.init_main_tabs()

    def init_menu_bar(self):
        """Initialize menu bar with configuration options"""
        menuBar = wx.MenuBar(style=wx.MB_DOCKABLE)
        self.SetMenuBar(menuBar)
        self.SetMinSize((1618, 902))  # Minimum window size
        
        # Settings menu
        setting = wx.Menu()
        menuBar.Append(setting, "&设置")
        params_conf = wx.MenuItem(setting, 0, "&参数配置")
        setting.Append(params_conf)
        self.Bind(wx.EVT_MENU, self.on_params_conf, params_conf)

    def init_main_tabs(self):
        """Initialize tabbed interface"""
        notebook = wx.Notebook(self)
        
        # Add panels for different functionalities
        self.backtest_panel = PanelBacktest(notebook)
        self.trade_panel = TradePanel(notebook)
        self.zhiku_panel = ZhikuPanel(notebook)
        self.web_panel = WebPanel(notebook)
        
        notebook.AddPage(self.backtest_panel, "策略回测")
        notebook.AddPage(self.trade_panel, "实盘交易")
        notebook.AddPage(self.zhiku_panel, "智库分析")
        notebook.AddPage(self.web_panel, "在线工具")

    def on_params_conf(self, event):
        """Handle parameter configuration dialog"""
        dialog = ParamsConfigDialog(self)
        dialog.ShowModal()
        dialog.Destroy()
```

**Panel Components**:

1. **PanelBacktest**: Strategy backtesting interface
2. **TradePanel**: Live trading controls
3. **ZhikuPanel**: Analysis and research tools
4. **WebPanel**: Web-based tools integration

## Utility Services

### BaseService (`utils/common/BaseService.py`)

**Purpose**: Base service class with common functionality

```python
class BaseService(object):
    """Base service providing logging, HTTP, and utility functions"""
    
    def __init__(self, logfile="default.log"):
        """Initialize service with logging"""
        self.logger = logger
        self.logger.add(logfile)
        self.init_const_data()
        self.params = None
        self.cookies = None

    def init_const_data(self):
        """Initialize common constants"""
        self.today = datetime.datetime.now().strftime("%Y-%m-%d")

    def check_path(self, path):
        """Create directory path if it doesn't exist"""
        if not os.path.exists(path):
            try:
                os.makedirs(path)
            except Exception as e:
                self.logger.error(e)

    def get_url_filename(self, url):
        """Extract filename from URL"""
        import urllib.parse
        parsed = urllib.parse.urlparse(url)
        return os.path.basename(parsed.path)

    def make_request(self, url, method='GET', **kwargs):
        """Make HTTP request with error handling"""
        try:
            response = requests.request(method, url, **kwargs)
            response.raise_for_status()
            return response
        except requests.RequestException as e:
            self.logger.error(f"Request failed: {e}")
            return None
```

### LarkBot (`utils/larkbot.py`)

**Purpose**: Lark/Feishu webhook integration for notifications

```python
class LarkBot:
    """Lark Bot for sending trading notifications"""
    
    def __init__(self, secret=None):
        """Initialize with webhook secret"""
        self.secret = secret
        self.webhook_url = WEBHOOK_URL

    def generate_sign(self, timestamp):
        """Generate signature for webhook authentication"""
        string_to_sign = f"{timestamp}\n{self.secret}"
        hmac_code = hmac.new(
            string_to_sign.encode("utf-8"),
            digestmod=hashlib.sha256
        ).digest()
        sign = base64.b64encode(hmac_code).decode('utf-8')
        return sign

    def send(self, content, msg_type="text"):
        """Send message via Lark webhook"""
        timestamp = str(int(datetime.now().timestamp()))
        sign = self.generate_sign(timestamp) if self.secret else ""
        
        if msg_type == "text":
            payload = {
                "msg_type": "text",
                "timestamp": timestamp,
                "sign": sign,
                "content": {"text": content}
            }
        elif msg_type == "interactive":
            # Rich card format
            payload = {
                "msg_type": "interactive", 
                "timestamp": timestamp,
                "sign": sign,
                "card": self._build_card(content)
            }
        
        try:
            response = requests.post(self.webhook_url, json=payload)
            return response.status_code == 200
        except Exception as e:
            print(f"Failed to send message: {e}")
            return False
```

## Backtesting Framework

### Trade Management (`pyfunds/backtest/xalpha/trade.py`)

**Purpose**: Portfolio and trade management for backtesting

#### Key Functions:

```python
def xirrcal(cftable, trades, date, startdate=None, guess=0.01):
    """
    Calculate XIRR (Extended Internal Rate of Return)
    
    Args:
        cftable (pd.DataFrame): Cash flow table with 'date' and 'cash' columns
        trades (list): List of trade objects  
        date (str/datetime): End date for calculation
        startdate (str/datetime, optional): Start date
        guess (float): Initial guess for IRR calculation
        
    Returns:
        float: XIRR rate as decimal
        
    Example:
        >>> import pandas as pd
        >>> cftable = pd.DataFrame({
        ...     'date': pd.to_datetime(['2020-01-01', '2020-06-01', '2020-12-31']),
        ...     'cash': [-10000, -5000, 16000]
        ... })
        >>> xirr = xirrcal(cftable, [], '2020-12-31')
        >>> print(f"XIRR: {xirr:.2%}")
    """
    date = convert_date(date)
    partcftb = cftable[cftable["date"] <= date]
    
    if len(partcftb) == 0:
        return 0
        
    if not startdate:
        cashflow = [(row["date"], row["cash"]) for i, row in partcftb.iterrows()]
    else:
        # Calculate starting value
        startdate = convert_date(startdate)  
        start_cash = sum(fund.briefdailyreport(startdate).get("currentvalue", 0) 
                        for fund in trades)
        cashflow = [(startdate, -start_cash)]
        cashflow.extend([(row["date"], row["cash"]) for i, row in partcftb.iterrows()])
    
    # Add final value
    final_value = sum(fund.briefdailyreport(date).get("currentvalue", 0) 
                     for fund in trades)
    cashflow.append((date, final_value))
    
    return xirr(cashflow, guess=guess)
```

### Portfolio Class Structure:

```python
class Portfolio:
    """Portfolio management for backtesting"""
    
    def __init__(self, initial_cash=100000):
        self.initial_cash = initial_cash
        self.cash = initial_cash
        self.positions = {}
        self.trades = []
        self.daily_values = []
        
    def buy(self, symbol, quantity, price, date):
        """Execute buy order"""
        cost = quantity * price
        if self.cash >= cost:
            self.cash -= cost
            if symbol in self.positions:
                # Average cost calculation
                current_qty = self.positions[symbol]['quantity']
                current_cost = self.positions[symbol]['avg_cost']
                new_avg_cost = (current_qty * current_cost + cost) / (current_qty + quantity)
                
                self.positions[symbol] = {
                    'quantity': current_qty + quantity,
                    'avg_cost': new_avg_cost
                }
            else:
                self.positions[symbol] = {
                    'quantity': quantity,
                    'avg_cost': price
                }
            
            self.trades.append({
                'date': date,
                'symbol': symbol,
                'action': 'BUY',
                'quantity': quantity,
                'price': price,
                'value': cost
            })
            return True
        return False
    
    def sell(self, symbol, quantity, price, date):
        """Execute sell order"""
        if symbol in self.positions and self.positions[symbol]['quantity'] >= quantity:
            revenue = quantity * price
            self.cash += revenue
            
            self.positions[symbol]['quantity'] -= quantity
            if self.positions[symbol]['quantity'] == 0:
                del self.positions[symbol]
            
            self.trades.append({
                'date': date,
                'symbol': symbol, 
                'action': 'SELL',
                'quantity': quantity,
                'price': price,
                'value': revenue
            })
            return True
        return False
    
    def get_portfolio_value(self, current_prices):
        """Calculate total portfolio value"""
        total_value = self.cash
        for symbol, position in self.positions.items():
            if symbol in current_prices:
                total_value += position['quantity'] * current_prices[symbol]
        return total_value
```

## Frontend Components

### TypeScript Utilities (`pytrader/frontend/src/utils/tools.ts`)

```typescript
/**
 * Utility functions for frontend operations
 */

export interface ILocalStore {
    key: string
    value: any
    expire?: number
}

export interface IMenubarList {
    id: string
    name: string
    icon?: string
    children?: IMenubarList[]
}

/**
 * Sleep function for async operations
 */
export async function sleep(time: number): Promise<void> {
    return new Promise(resolve => {
        setTimeout(resolve, time)
    })
}

/**
 * Format currency with symbol and thousands separator
 */
export function format(num: number | string, symbol = '￥'): string {
    if (Number.isNaN(Number(num))) return `${symbol}0.00`
    return symbol + (Number(num).toFixed(2))
        .replace(/(\d)(?=(\d{3})+\.)/g, '$1,')
}

/**
 * Remove currency formatting
 */
export function unformat(str: string): number | string {
    const s = str.substr(1).replace(/\,/g, '')
    return Number.isNaN(Number(s)) || Number(s) === 0 ? '' : Number(s)
}

/**
 * Calculate table summary row for financial data
 */
export function tableSummaries(
    param: { columns: any; data: any }
): Array<string | number> {
    const { columns, data } = param
    const sums: Array<string | number> = []
    
    columns.forEach((column: { property: string | number }, index: number) => {
        if (index === 0) {
            sums[index] = '合计'
            return
        }
        
        const values = data.map((item: { [x: string]: any }) => 
            Number(item[column.property])
        )
        
        if (!values.every((value: number) => isNaN(value))) {
            sums[index] = values.reduce((prev: number, curr: number) => {
                const value = Number(curr)
                return !isNaN(value) ? prev + curr : prev
            }, 0)
        } else {
            sums[index] = ''
        }
    })
    
    return sums
}

/**
 * Local storage management with expiration
 */
export class LocalStorageManager {
    static set(key: string, value: any, expire?: number): void {
        const item: ILocalStore = { key, value }
        if (expire) {
            item.expire = Date.now() + expire * 1000
        }
        localStorage.setItem(key, JSON.stringify(item))
    }
    
    static get(key: string): any {
        const item = localStorage.getItem(key)
        if (!item) return null
        
        const parsed: ILocalStore = JSON.parse(item)
        
        if (parsed.expire && Date.now() > parsed.expire) {
            localStorage.removeItem(key)
            return null
        }
        
        return parsed.value
    }
    
    static remove(key: string): void {
        localStorage.removeItem(key)
    }
    
    static clear(): void {
        localStorage.clear()
    }
}
```

### Fund Strategy Components (`pyfunds/fund-strategies/src/utils/fund-stragegy/index.ts`)

```typescript
/**
 * Fund investment strategy implementation
 */

export interface FixedInvestOption {
    fixedInvestment: {
        amount: number        // Investment amount per period
        dateOrWeek: number   // Day of week/month for investment  
        period: 'weekly' | 'monthly'  // Investment frequency
    }
    range: [string | Date, string | Date]  // Investment period
}

export class InvestmentStrategy {
    totalAmount!: number     // Initial capital
    salary!: number          // Monthly salary addition
    onEachDay!: Function     // Daily callback function
    
    latestInvestment!: InvestDateSnapshot
    shangZhengData!: Record<string, IndexData>
    indexData?: Record<string, IndexData>
    fundJson!: FundJson
    
    // Fee configuration
    buyFeeRate: number = 0.0015   // 0.15% buy fee
    sellFeeRate: number = 0.005   // 0.5% sell fee
    
    // Risk management
    stop!: {
        rate: number      // Profit taking rate (e.g., 5%)
        minAmount: number // Minimum position threshold
    }
    
    tInvest!: {
        rate: number      // Dip buying threshold
        amount: number    // DCA amount
    }
    
    data: InvestDateSnapshot[] = []
    dataMap: Record<string, InvestDateSnapshot> = {}
    fixedConfig!: FixedInvestOption
    
    /**
     * Calculate annualized returns
     */
    get annualizedRate(): {
        fundGrowth: number
        totalProfit: number
    } {
        const len = this.data.length
        if (len > 0) {
            const startFund = this.data[0]
            const endFund = this.data[len - 1]
            const rangeTime = new Date(endFund.date).getTime() - 
                             new Date(startFund.date).getTime()
            const rangeYear = rangeTime / (24 * 60 * 60 * 1000) / 365
            
            return {
                fundGrowth: roundToFix(
                    Math.pow(1 + endFund.fundGrowthRate, 1 / rangeYear) - 1, 4
                ),
                totalProfit: roundToFix(
                    Math.pow(1 + endFund.totalProfitRate, 1 / rangeYear) - 1, 4
                )
            }
        }
        return { fundGrowth: 0, totalProfit: 0 }
    }
    
    /**
     * Execute fixed investment strategy (Dollar-Cost Averaging)
     */
    fixedInvest(opt: FixedInvestOption): InvestmentStrategy {
        const { range, fixedInvestment } = opt
        this.fixedConfig = opt
        
        const beginTime = new Date(range[0]).getTime()
        const endTime = new Date(range[1]).getTime()
        
        if (beginTime > endTime) {
            throw new Error('End date must be after start date')
        }
        
        let curDate = beginTime
        
        while (curDate <= endTime) {
            if (this.shouldFixedInvest(fixedInvestment, curDate)) {
                this.buy(fixedInvestment.amount, curDate)
            } else {
                this.buy(0, curDate)  // Update data even on non-investment days
            }
            
            this.onEachDay(curDate)
            curDate += 24 * 60 * 60 * 1000  // Next day
        }
        
        return this
    }
    
    /**
     * Buy fund shares
     */
    buy(amount: number, date: any): InvestmentStrategy {
        const dateStr = dateFormat(date)
        const invest = this.getSnapshotInstance(dateStr).buy(amount)
        this.pushData(invest)
        return this
    }
    
    /**
     * Sell fund shares
     */
    sell(amount: number | 'all', date: any): InvestmentStrategy {
        const dateStr = dateFormat(date)
        const invest = this.getSnapshotInstance(dateStr)
        
        if (amount === 'all') {
            invest.sell('all')
        } else {
            invest.sell({ amount })
        }
        
        this.pushData(invest)
        return this
    }
    
    private shouldFixedInvest(
        fixedInvestment: FixedInvestOption['fixedInvestment'], 
        date: any
    ): boolean {
        const now = new Date(date)
        if (fixedInvestment.period === 'monthly') {
            return now.getDate() === fixedInvestment.dateOrWeek
        } else if (fixedInvestment.period === 'weekly') {
            return now.getDay() === fixedInvestment.dateOrWeek
        }
        return false
    }
}
```

## Configuration

### Global Configuration Files

#### `config/settings.py` (Example)
```python
# Trading configuration
TRADING_CONFIG = {
    'default_cash': 100000,
    'default_commission': 0.001,
    'max_positions': 5,
    'risk_per_trade': 0.02,
    'stop_loss_atr': 2.0,
}

# Data source configuration  
DATA_CONFIG = {
    'tushare_token': 'your_tushare_token',
    'data_path': './data/',
    'cache_enabled': True,
    'cache_duration': 3600,  # 1 hour
}

# Notification configuration
NOTIFICATION_CONFIG = {
    'lark_webhook': 'your_lark_webhook_url',
    'lark_secret': 'your_lark_secret',
    'email_enabled': False,
    'sound_enabled': True,
}

# Strategy configuration
STRATEGY_CONFIG = {
    'sma_cross': {
        'pfast': 10,
        'pslow': 30,
    },
    'multi_strategy': {
        'maperiod': 15,
        'exitbars': 5,
    },
    'lstm_strategy': {
        'period': 10,
        'neurons': 50,
        'train_size': 0.8,
        'lookback': 20,
    }
}
```

#### Environment Variables
```bash
# Required environment variables
export QBOT_DATA_PATH="/path/to/data"
export QBOT_LOG_LEVEL="INFO"
export TUSHARE_TOKEN="your_token"
export LARK_WEBHOOK_SECRET="your_secret"

# Optional environment variables
export QBOT_GUI_THEME="dark"
export QBOT_CACHE_ENABLED="true"
export QBOT_SOUND_ENABLED="true"
```

This component reference provides comprehensive documentation for all major Qbot components. Use it as a guide for understanding the architecture, extending functionality, and integrating with external systems.