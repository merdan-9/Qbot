# Qbot Trading Strategy Development Guide

## Overview

This guide provides comprehensive information on developing, testing, and implementing trading strategies in the Qbot platform. It covers both traditional technical analysis strategies and AI-powered approaches.

## Table of Contents

1. [Strategy Architecture](#strategy-architecture)
2. [Built-in Strategies](#built-in-strategies)
3. [Creating Custom Strategies](#creating-custom-strategies)
4. [AI/ML Strategies](#aiml-strategies)
5. [Strategy Testing](#strategy-testing)
6. [Performance Optimization](#performance-optimization)
7. [Risk Management](#risk-management)

## Strategy Architecture

### Base Strategy Structure

All Qbot strategies inherit from Backtrader's `bt.Strategy` class and follow this structure:

```python
import backtrader as bt
import backtrader.indicators as btind

class MyStrategy(bt.Strategy):
    # Strategy parameters
    params = (
        ('param1', default_value1),
        ('param2', default_value2),
    )
    
    def __init__(self):
        """Initialize indicators and variables"""
        self.dataclose = self.datas[0].close
        self.order = None
        # Initialize indicators here
    
    def notify_order(self, order):
        """Handle order status updates"""
        pass
    
    def notify_trade(self, trade):
        """Handle trade completion notifications"""
        pass
    
    def next(self):
        """Main strategy logic executed on each bar"""
        pass
    
    def stop(self):
        """Called when strategy execution ends"""
        pass
```

### Strategy Lifecycle

1. **Initialization** (`__init__`): Set up indicators and variables
2. **Execution** (`next`): Process each data point
3. **Order Management** (`notify_order`): Handle order updates
4. **Trade Management** (`notify_trade`): Handle completed trades
5. **Cleanup** (`stop`): Final calculations and cleanup

## Built-in Strategies

### 1. Simple Moving Average Crossover

**File**: `qbot/strategies/sma_cross_strategy_bt.py`

**Concept**: Buy when fast MA crosses above slow MA, sell when it crosses below.

```python
class SmaCross(bt.Strategy):
    params = (('pfast', 10), ('pslow', 30),)

    def __init__(self):
        sma1 = btind.SMA(period=self.p.pfast)
        sma2 = btind.SMA(period=self.p.pslow)
        self.crossover = btind.CrossOver(sma1, sma2)

    def next(self):
        if self.position.size == 0:
            if self.crossover > 0:  # Fast MA crosses above slow MA
                amount_to_invest = (self.broker.cash * 0.95)
                self.size = int(amount_to_invest / self.data.close)
                self.buy(size=self.size)
        elif self.position.size > 0:
            if self.crossover < 0:  # Fast MA crosses below slow MA
                self.close()
```

**Key Features**:
- Uses 95% of available cash for trades
- Simple crossover logic
- Suitable for trending markets

### 2. Multi-Factor Strategy

**File**: `qbot/strategies/multi_strategy_bt.py`

**Concept**: Combines SMA and RSI indicators for enhanced signal quality.

```python
class MultiStrategy(bt.Strategy):
    params = (('exitbars', 5), ('maperiod', 15),)

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.sma = btind.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod
        )
        self.rsi = btind.RelativeStrengthIndex()

    def next(self):
        if not self.position:
            # Buy conditions: RSI crosses above 50 AND price > SMA
            if (self.rsi[0] < 50 and self.rsi[-1] >= 50 and 
                self.dataclose[0] > self.sma[0]):
                self.order = self.buy()
        else:
            # Sell conditions: RSI crosses below 50 AND price < SMA
            if (self.rsi[0] > 50 and self.rsi[-1] <= 50 and 
                self.dataclose[0] < self.sma[0]):
                self.order = self.sell()
```

**Key Features**:
- Dual confirmation system
- Reduced false signals
- Better suited for volatile markets

### 3. Bollinger Bands Strategy

**File**: `qbot/strategies/boll_strategy_bt.py`

**Concept**: Mean reversion strategy using Bollinger Bands.

```python
class BollingerBandsStrategy(bt.Strategy):
    params = (
        ('period', 20),
        ('devfactor', 2.0),
        ('rsi_period', 14),
        ('rsi_oversold', 30),
        ('rsi_overbought', 70),
    )

    def __init__(self):
        self.boll = btind.BollingerBands(
            period=self.p.period, 
            devfactor=self.p.devfactor
        )
        self.rsi = btind.RSI(period=self.p.rsi_period)
        
        # Buy when price touches lower band and RSI is oversold
        self.buy_signal = btind.And(
            self.data.close <= self.boll.lines.bot,
            self.rsi <= self.p.rsi_oversold
        )
        
        # Sell when price touches upper band and RSI is overbought
        self.sell_signal = btind.And(
            self.data.close >= self.boll.lines.top,
            self.rsi >= self.p.rsi_overbought
        )

    def next(self):
        if not self.position:
            if self.buy_signal:
                self.buy()
        else:
            if self.sell_signal:
                self.sell()
```

### 4. LSTM Deep Learning Strategy

**File**: `qbot/strategies/lstm_strategy_bt.py`

**Concept**: Uses LSTM neural networks for price prediction.

```python
from keras.models import Sequential
from keras.layers import Dense, LSTM, Dropout
from sklearn.preprocessing import MinMaxScaler

class LSTMPredict(bt.Strategy):
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

    def _prepare_data(self):
        """Prepare training and testing datasets"""
        data = np.array(self.dataclose)
        data = np.reshape(data, (-1, 1))
        data = self.scaler.fit_transform(data)
        
        train_size = int(len(data) * self.train_size)
        train_data = data[0:train_size, :]
        test_data = data[train_size:len(data), :]
        
        # Create sequences for LSTM
        def create_dataset(dataset, look_back=1):
            X, Y = [], []
            for i in range(look_back, len(dataset)):
                X.append(dataset[i-look_back:i, 0])
                Y.append(dataset[i, 0])
            return np.array(X), np.array(Y)
        
        trainX, trainY = create_dataset(train_data, self.lookback)
        testX, testY = create_dataset(test_data, self.lookback)
        
        trainX = np.reshape(trainX, (trainX.shape[0], trainX.shape[1], 1))
        testX = np.reshape(testX, (testX.shape[0], testX.shape[1], 1))
        
        return {'X': trainX, 'Y': trainY}, {'X': testX, 'Y': testY}

    def next(self):
        """LSTM prediction-based trading logic"""
        if len(self.data) > self.lookback:
            # Prepare recent data for prediction
            recent_data = np.array(self.dataclose.get(size=self.lookback))
            recent_data = self.scaler.transform(recent_data.reshape(-1, 1))
            recent_data = recent_data.reshape(1, self.lookback, 1)
            
            # Make prediction
            predicted_price = self.model.predict(recent_data)[0][0]
            predicted_price = self.scaler.inverse_transform([[predicted_price]])[0][0]
            
            current_price = self.dataclose[0]
            
            # Trading logic based on prediction
            if predicted_price > current_price * 1.02:  # 2% threshold
                if not self.position:
                    self.buy()
            elif predicted_price < current_price * 0.98:  # -2% threshold
                if self.position:
                    self.sell()
```

## Creating Custom Strategies

### Step 1: Define Strategy Class

```python
class MyCustomStrategy(bt.Strategy):
    # Define parameters with default values
    params = (
        ('fast_period', 12),
        ('slow_period', 26),
        ('signal_period', 9),
        ('rsi_period', 14),
        ('rsi_overbought', 70),
        ('rsi_oversold', 30),
    )
    
    def __init__(self):
        # Access price data
        self.dataclose = self.datas[0].close
        self.datahigh = self.datas[0].high
        self.datalow = self.datas[0].low
        
        # Initialize indicators
        self.macd = btind.MACD(
            period_me1=self.p.fast_period,
            period_me2=self.p.slow_period,
            period_signal=self.p.signal_period
        )
        self.rsi = btind.RSI(period=self.p.rsi_period)
        
        # Order management
        self.order = None
        self.buyprice = None
        self.buycomm = None
```

### Step 2: Implement Trading Logic

```python
def next(self):
    # Skip if there's a pending order
    if self.order:
        return
    
    # Entry conditions
    if not self.position:
        # MACD bullish crossover + RSI oversold
        if (self.macd.macd[0] > self.macd.signal[0] and
            self.macd.macd[-1] <= self.macd.signal[-1] and
            self.rsi[0] < self.p.rsi_oversold):
            
            self.log('BUY CREATE, %.2f' % self.dataclose[0])
            self.order = self.buy()
    
    # Exit conditions
    else:
        # MACD bearish crossover + RSI overbought
        if (self.macd.macd[0] < self.macd.signal[0] and
            self.macd.macd[-1] >= self.macd.signal[-1] and
            self.rsi[0] > self.p.rsi_overbought):
            
            self.log('SELL CREATE, %.2f' % self.dataclose[0])
            self.order = self.sell()
```

### Step 3: Add Order and Trade Notifications

```python
def notify_order(self, order):
    if order.status in [order.Submitted, order.Accepted]:
        return
    
    if order.status in [order.Completed]:
        if order.isbuy():
            self.log(
                'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                (order.executed.price,
                 order.executed.value,
                 order.executed.comm))
            self.buyprice = order.executed.price
            self.buycomm = order.executed.comm
        else:
            self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                     (order.executed.price,
                      order.executed.value,
                      order.executed.comm))
        
        self.bar_executed = len(self)
    
    elif order.status in [order.Canceled, order.Margin, order.Rejected]:
        self.log('Order Canceled/Margin/Rejected')
    
    self.order = None

def notify_trade(self, trade):
    if not trade.isclosed:
        return
    
    self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
             (trade.pnl, trade.pnlcomm))

def log(self, txt, dt=None):
    dt = dt or self.datas[0].datetime.date(0)
    print('%s, %s' % (dt.isoformat(), txt))
```

## AI/ML Strategies

### 1. Reinforcement Learning Strategy

```python
import gym
import numpy as np
from stable_baselines3 import PPO

class RLStrategy(bt.Strategy):
    params = (
        ('lookback', 30),
        ('model_path', 'models/ppo_trading_model'),
    )
    
    def __init__(self):
        self.dataclose = self.datas[0].close
        self.datahigh = self.datas[0].high
        self.datalow = self.datas[0].low
        self.datavolume = self.datas[0].volume
        
        # Load pre-trained RL model
        self.model = PPO.load(self.p.model_path)
        
        # State buffer for RL agent
        self.state_buffer = []
        
    def _get_state(self):
        """Get current market state for RL agent"""
        if len(self.state_buffer) < self.p.lookback:
            return None
        
        # Normalize features
        prices = np.array([s['close'] for s in self.state_buffer[-self.p.lookback:]])
        volumes = np.array([s['volume'] for s in self.state_buffer[-self.p.lookback:]])
        
        price_returns = np.diff(prices) / prices[:-1]
        volume_ma = np.mean(volumes)
        
        # Technical indicators as features
        rsi = self._calculate_rsi(prices)
        macd = self._calculate_macd(prices)
        
        state = np.concatenate([
            price_returns,
            [rsi, macd, volume_ma, self.broker.getvalue()]
        ])
        
        return state
    
    def next(self):
        # Update state buffer
        current_state = {
            'close': self.dataclose[0],
            'high': self.datahigh[0],
            'low': self.datalow[0],
            'volume': self.datavolume[0]
        }
        self.state_buffer.append(current_state)
        
        # Get action from RL model
        state = self._get_state()
        if state is not None:
            action, _ = self.model.predict(state)
            
            # Execute action: 0=hold, 1=buy, 2=sell
            if action == 1 and not self.position:
                self.buy()
            elif action == 2 and self.position:
                self.sell()
```

### 2. Ensemble Strategy

```python
from sklearn.ensemble import RandomForestClassifier, VotingClassifier
from sklearn.svm import SVC
from sklearn.linear_model import LogisticRegression

class EnsembleStrategy(bt.Strategy):
    params = (
        ('lookback', 20),
        ('retrain_period', 252),  # Retrain yearly
    )
    
    def __init__(self):
        self.dataclose = self.datas[0].close
        self.datahigh = self.datas[0].high
        self.datalow = self.datas[0].low
        self.datavolume = self.datas[0].volume
        
        # Technical indicators
        self.sma_fast = btind.SMA(period=10)
        self.sma_slow = btind.SMA(period=20)
        self.rsi = btind.RSI(period=14)
        self.macd = btind.MACD()
        self.bb = btind.BollingerBands()
        
        # Ensemble model
        self.model = self._create_ensemble_model()
        self.last_retrain = 0
        self.feature_buffer = []
        self.target_buffer = []
        
    def _create_ensemble_model(self):
        """Create ensemble model with multiple algorithms"""
        rf = RandomForestClassifier(n_estimators=100, random_state=42)
        svm = SVC(probability=True, random_state=42)
        lr = LogisticRegression(random_state=42)
        
        ensemble = VotingClassifier(
            estimators=[('rf', rf), ('svm', svm), ('lr', lr)],
            voting='soft'
        )
        
        return ensemble
    
    def _extract_features(self):
        """Extract features for ML model"""
        if len(self.data) < self.p.lookback:
            return None
        
        features = [
            self.sma_fast[0] / self.dataclose[0] - 1,  # SMA fast ratio
            self.sma_slow[0] / self.dataclose[0] - 1,  # SMA slow ratio
            self.rsi[0] / 100,  # Normalized RSI
            (self.dataclose[0] - self.bb.lines.mid[0]) / self.bb.lines.mid[0],  # BB position
            self.macd.macd[0],  # MACD line
            self.macd.signal[0],  # MACD signal
            np.log(self.datavolume[0] / np.mean([self.datavolume[-i] for i in range(1, 11)])),  # Volume ratio
        ]
        
        return np.array(features)
    
    def _get_target(self, forward_days=5):
        """Calculate future return for training"""
        if len(self.data) < forward_days:
            return None
        
        current_price = self.dataclose[0]
        future_price = self.dataclose[-forward_days]
        return_rate = (future_price - current_price) / current_price
        
        # Convert to classification: 0=sell, 1=hold, 2=buy
        if return_rate > 0.02:  # 2% threshold
            return 2
        elif return_rate < -0.02:
            return 0
        else:
            return 1
    
    def next(self):
        features = self._extract_features()
        if features is None:
            return
        
        # Collect training data
        target = self._get_target()
        if target is not None:
            self.feature_buffer.append(features)
            self.target_buffer.append(target)
        
        # Retrain model periodically
        if (len(self.data) - self.last_retrain > self.p.retrain_period and 
            len(self.feature_buffer) > 100):
            
            X = np.array(self.feature_buffer)
            y = np.array(self.target_buffer)
            self.model.fit(X, y)
            self.last_retrain = len(self.data)
        
        # Make prediction and trade
        if hasattr(self.model, 'predict'):
            try:
                prediction = self.model.predict([features])[0]
                probability = self.model.predict_proba([features])[0]
                
                # Only trade if confidence is high
                max_prob = np.max(probability)
                if max_prob > 0.7:  # 70% confidence threshold
                    if prediction == 2 and not self.position:  # Buy
                        self.buy()
                    elif prediction == 0 and self.position:  # Sell
                        self.sell()
            except:
                pass  # Model not trained yet
```

## Strategy Testing

### 1. Basic Backtesting Setup

```python
import backtrader as bt
import backtrader.analyzers as btanalyzers
import pandas as pd

def run_backtest(strategy_class, data, **kwargs):
    """
    Run backtest for a given strategy
    
    Args:
        strategy_class: Strategy class to test
        data: Price data (pandas DataFrame)
        **kwargs: Strategy parameters
    
    Returns:
        dict: Backtest results
    """
    cerebro = bt.Cerebro()
    
    # Add strategy with parameters
    cerebro.addstrategy(strategy_class, **kwargs)
    
    # Add data
    data_feed = bt.feeds.PandasData(dataname=data)
    cerebro.adddata(data_feed)
    
    # Set broker parameters
    cerebro.broker.setcash(100000.0)
    cerebro.broker.setcommission(commission=0.001)
    
    # Add analyzers
    cerebro.addanalyzer(btanalyzers.SharpeRatio, _name='sharpe')
    cerebro.addanalyzer(btanalyzers.DrawDown, _name='drawdown')
    cerebro.addanalyzer(btanalyzers.TradeAnalyzer, _name='trades')
    cerebro.addanalyzer(btanalyzers.Returns, _name='returns')
    
    # Run backtest
    results = cerebro.run()
    strategy_result = results[0]
    
    # Extract metrics
    sharpe = strategy_result.analyzers.sharpe.get_analysis()
    drawdown = strategy_result.analyzers.drawdown.get_analysis()
    trades = strategy_result.analyzers.trades.get_analysis()
    returns = strategy_result.analyzers.returns.get_analysis()
    
    return {
        'final_value': cerebro.broker.getvalue(),
        'sharpe_ratio': sharpe.get('sharperatio', 0),
        'max_drawdown': drawdown.get('max', {}).get('drawdown', 0),
        'total_trades': trades.get('total', {}).get('total', 0),
        'win_rate': trades.get('won', {}).get('total', 0) / max(trades.get('total', {}).get('total', 1), 1),
        'annual_return': returns.get('rnorm100', 0),
        'strategy': strategy_result
    }
```

### 2. Parameter Optimization

```python
def optimize_strategy(strategy_class, data, param_ranges):
    """
    Optimize strategy parameters using grid search
    
    Args:
        strategy_class: Strategy class to optimize
        data: Price data
        param_ranges: Dictionary of parameter ranges
    
    Returns:
        dict: Best parameters and performance
    """
    best_sharpe = -999
    best_params = None
    best_results = None
    
    # Generate parameter combinations
    param_names = list(param_ranges.keys())
    param_values = list(param_ranges.values())
    
    from itertools import product
    for param_combo in product(*param_values):
        params = dict(zip(param_names, param_combo))
        
        try:
            results = run_backtest(strategy_class, data, **params)
            
            if results['sharpe_ratio'] > best_sharpe:
                best_sharpe = results['sharpe_ratio']
                best_params = params
                best_results = results
                
        except Exception as e:
            print(f"Error with params {params}: {e}")
            continue
    
    return {
        'best_params': best_params,
        'best_performance': best_results,
        'best_sharpe': best_sharpe
    }

# Example usage
param_ranges = {
    'pfast': [5, 10, 15, 20],
    'pslow': [20, 30, 40, 50],
}

optimization_results = optimize_strategy(
    SmaCross, 
    stock_data, 
    param_ranges
)
```

### 3. Walk-Forward Analysis

```python
def walk_forward_analysis(strategy_class, data, train_periods=252, test_periods=63):
    """
    Perform walk-forward analysis
    
    Args:
        strategy_class: Strategy to test
        data: Full dataset
        train_periods: Training period length
        test_periods: Testing period length
    
    Returns:
        list: Results for each test period
    """
    results = []
    
    for i in range(train_periods, len(data) - test_periods, test_periods):
        # Split data
        train_data = data.iloc[i-train_periods:i]
        test_data = data.iloc[i:i+test_periods]
        
        # Optimize on training data
        param_ranges = {
            'pfast': [5, 10, 15],
            'pslow': [20, 30, 40],
        }
        
        optimization = optimize_strategy(strategy_class, train_data, param_ranges)
        best_params = optimization['best_params']
        
        # Test on out-of-sample data
        test_results = run_backtest(strategy_class, test_data, **best_params)
        
        results.append({
            'train_period': (i-train_periods, i),
            'test_period': (i, i+test_periods),
            'optimized_params': best_params,
            'test_performance': test_results
        })
    
    return results
```

## Performance Optimization

### 1. Vectorized Calculations

```python
class OptimizedStrategy(bt.Strategy):
    def __init__(self):
        # Pre-calculate all indicators at once
        self.sma_fast = btind.SMA(period=10)
        self.sma_slow = btind.SMA(period=20)
        
        # Use vectorized crossover
        self.crossover = btind.CrossOver(self.sma_fast, self.sma_slow)
        
        # Pre-calculate buy/sell signals
        self.buy_signal = self.crossover > 0
        self.sell_signal = self.crossover < 0
    
    def next(self):
        # Simple boolean logic instead of complex calculations
        if not self.position and self.buy_signal[0]:
            self.buy()
        elif self.position and self.sell_signal[0]:
            self.sell()
```

### 2. Memory Management

```python
class MemoryEfficientStrategy(bt.Strategy):
    def __init__(self):
        # Limit indicator memory usage
        self.sma = btind.SMA(period=20)
        self.sma.plotinfo.plot = False  # Don't store plotting data
        
        # Use minimal data storage
        self.data.plotinfo.plot = False
        
        # Clean up old data periodically
        self.cleanup_counter = 0
    
    def next(self):
        # Strategy logic here
        
        # Cleanup every 1000 bars
        self.cleanup_counter += 1
        if self.cleanup_counter % 1000 == 0:
            # Force garbage collection
            import gc
            gc.collect()
```

## Risk Management

### 1. Position Sizing

```python
class PositionSizedStrategy(bt.Strategy):
    params = (
        ('risk_per_trade', 0.02),  # 2% risk per trade
        ('max_position_size', 0.1),  # 10% max position size
    )
    
    def __init__(self):
        self.atr = btind.ATR(period=14)  # Average True Range for volatility
        self.sma = btind.SMA(period=20)
    
    def calculate_position_size(self, entry_price, stop_loss):
        """Calculate position size based on risk management"""
        account_value = self.broker.getvalue()
        risk_amount = account_value * self.p.risk_per_trade
        
        # Calculate risk per share
        risk_per_share = abs(entry_price - stop_loss)
        
        if risk_per_share > 0:
            # Position size based on risk
            shares = int(risk_amount / risk_per_share)
            
            # Apply maximum position size limit
            max_shares = int(account_value * self.p.max_position_size / entry_price)
            shares = min(shares, max_shares)
            
            return shares
        
        return 0
    
    def next(self):
        if not self.position:
            # Entry signal
            if self.data.close[0] > self.sma[0]:
                entry_price = self.data.close[0]
                stop_loss = entry_price - (2 * self.atr[0])  # 2 ATR stop
                
                size = self.calculate_position_size(entry_price, stop_loss)
                if size > 0:
                    self.buy(size=size)
                    # Set stop loss
                    self.sell(exectype=bt.Order.Stop, price=stop_loss, size=size)
```

### 2. Portfolio-Level Risk Management

```python
class RiskManagedPortfolio:
    def __init__(self, max_drawdown=0.15, max_correlation=0.7):
        self.max_drawdown = max_drawdown
        self.max_correlation = max_correlation
        self.positions = {}
        self.peak_value = 0
        
    def check_drawdown(self, current_value):
        """Check if drawdown limit is exceeded"""
        self.peak_value = max(self.peak_value, current_value)
        current_drawdown = (self.peak_value - current_value) / self.peak_value
        
        return current_drawdown < self.max_drawdown
    
    def check_correlation(self, new_symbol, existing_symbols):
        """Check correlation with existing positions"""
        # Implementation would calculate correlation matrix
        # and ensure new position doesn't exceed correlation limit
        pass
    
    def should_enter_position(self, symbol, current_portfolio_value):
        """Determine if new position should be entered"""
        if not self.check_drawdown(current_portfolio_value):
            return False, "Drawdown limit exceeded"
        
        if len(self.positions) > 0:
            if not self.check_correlation(symbol, list(self.positions.keys())):
                return False, "Correlation limit exceeded"
        
        return True, "OK"
```

### 3. Dynamic Risk Adjustment

```python
class AdaptiveRiskStrategy(bt.Strategy):
    params = (
        ('base_risk', 0.02),
        ('volatility_lookback', 20),
    )
    
    def __init__(self):
        self.returns = btind.PctChange(self.data.close, period=1)
        self.volatility = btind.StdDev(self.returns, period=self.p.volatility_lookback)
        self.sma = btind.SMA(period=20)
        
    def calculate_adaptive_risk(self):
        """Adjust risk based on market volatility"""
        current_vol = self.volatility[0]
        historical_vol = np.mean([self.volatility[-i] for i in range(1, 252)])  # 1-year average
        
        vol_ratio = current_vol / historical_vol if historical_vol > 0 else 1
        
        # Reduce risk when volatility is high
        if vol_ratio > 1.5:
            risk_multiplier = 0.5
        elif vol_ratio > 1.2:
            risk_multiplier = 0.75
        elif vol_ratio < 0.8:
            risk_multiplier = 1.25
        else:
            risk_multiplier = 1.0
        
        return self.p.base_risk * risk_multiplier
    
    def next(self):
        if not self.position:
            if self.data.close[0] > self.sma[0]:
                # Calculate adaptive position size
                adaptive_risk = self.calculate_adaptive_risk()
                account_value = self.broker.getvalue()
                position_value = account_value * adaptive_risk
                size = int(position_value / self.data.close[0])
                
                if size > 0:
                    self.buy(size=size)
```

## Best Practices

### 1. Code Organization

```python
# strategy_base.py
class BaseStrategy(bt.Strategy):
    """Base strategy with common functionality"""
    
    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print(f'{dt.isoformat()}, {txt}')
    
    def notify_order(self, order):
        # Common order handling logic
        pass
    
    def notify_trade(self, trade):
        # Common trade handling logic
        pass

# my_strategy.py
from strategy_base import BaseStrategy

class MyStrategy(BaseStrategy):
    def __init__(self):
        super().__init__()
        # Strategy-specific initialization
    
    def next(self):
        # Strategy-specific logic
        pass
```

### 2. Configuration Management

```python
# config.py
STRATEGY_CONFIGS = {
    'sma_cross': {
        'pfast': 10,
        'pslow': 30,
        'risk_per_trade': 0.02,
    },
    'multi_factor': {
        'maperiod': 15,
        'rsi_period': 14,
        'risk_per_trade': 0.015,
    }
}

BROKER_CONFIG = {
    'cash': 100000,
    'commission': 0.001,
}

# strategy_runner.py
from config import STRATEGY_CONFIGS, BROKER_CONFIG

def run_strategy(strategy_name, data):
    config = STRATEGY_CONFIGS[strategy_name]
    # Use config to run strategy
```

### 3. Testing Framework

```python
# test_strategies.py
import unittest
from unittest.mock import Mock
import pandas as pd

class TestStrategies(unittest.TestCase):
    def setUp(self):
        # Create mock data
        self.test_data = pd.DataFrame({
            'open': [100, 101, 102, 103],
            'high': [101, 102, 103, 104],
            'low': [99, 100, 101, 102],
            'close': [100.5, 101.5, 102.5, 103.5],
            'volume': [1000, 1100, 1200, 1300]
        })
    
    def test_sma_crossover_signals(self):
        # Test SMA crossover logic
        strategy = SmaCross()
        # Mock the strategy setup
        # Test signal generation
        pass
    
    def test_risk_management(self):
        # Test position sizing
        # Test stop losses
        # Test drawdown limits
        pass

if __name__ == '__main__':
    unittest.main()
```

This comprehensive guide covers the essential aspects of trading strategy development in Qbot. Remember to always backtest thoroughly, implement proper risk management, and validate your strategies on out-of-sample data before deploying them in live trading.