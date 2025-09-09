# Qbot Documentation Index

## Overview

Welcome to the comprehensive documentation for Qbot - an AI-powered quantitative trading platform. This documentation covers all aspects of the platform, from basic usage to advanced strategy development.

## 📚 Documentation Structure

### 1. [API Documentation](./API_DOCUMENTATION.md)
**Complete API reference for all Qbot components**
- Core Components Overview
- Main Entry Points (`main.py`, `qbot_main.py`)
- Trading Strategies Framework
- GUI Components
- Utility Functions
- Backtesting Framework
- Frontend Components
- Usage Examples and Best Practices

**Best for**: Developers wanting to understand the complete API surface and integration points.

### 2. [Strategy Development Guide](./STRATEGY_GUIDE.md)
**Comprehensive guide for creating and optimizing trading strategies**
- Strategy Architecture and Lifecycle
- Built-in Strategy Examples
- Creating Custom Strategies
- AI/ML Strategy Implementation
- Strategy Testing and Optimization
- Performance Optimization
- Risk Management
- Best Practices and Code Organization

**Best for**: Quantitative analysts and developers creating new trading strategies.

### 3. [Usage Examples and Tutorials](./USAGE_EXAMPLES.md)
**Practical examples and step-by-step tutorials**
- Quick Start Guide
- Basic Strategy Backtesting
- Real-time Signal Generation
- Multi-Strategy Portfolio Management
- Parameter Optimization
- Fund Investment Strategies
- Risk Management Implementation
- Live Trading Integration
- Common Use Cases and Troubleshooting

**Best for**: Users getting started with Qbot or looking for specific implementation examples.

### 4. [Component Reference](./COMPONENT_REFERENCE.md)
**Detailed reference for all platform components**
- Entry Points Documentation
- Core Trading Engine
- Strategy Framework
- GUI Components
- Utility Services
- Backtesting Framework
- Frontend Components
- Configuration Options

**Best for**: Developers needing detailed information about specific components and their APIs.

## 🚀 Quick Start

### For New Users
1. Start with [Usage Examples](./USAGE_EXAMPLES.md#quick-start-guide)
2. Follow the installation and setup guide
3. Run your first backtest
4. Explore the GUI application

### For Strategy Developers
1. Read the [Strategy Development Guide](./STRATEGY_GUIDE.md)
2. Study the built-in strategy examples
3. Create your first custom strategy
4. Learn about risk management and optimization

### For System Integrators
1. Review the [API Documentation](./API_DOCUMENTATION.md)
2. Study the [Component Reference](./COMPONENT_REFERENCE.md)
3. Understand the architecture and data flows
4. Plan your integration approach

## 📋 Feature Overview

### Core Features
- **Multi-Strategy Trading**: Combine multiple strategies for robust trading
- **AI/ML Integration**: LSTM, reinforcement learning, and ensemble methods
- **Real-time Monitoring**: Live market data and signal generation
- **Comprehensive Backtesting**: Historical performance analysis
- **Risk Management**: Position sizing, stop losses, drawdown control
- **GUI Interface**: User-friendly desktop application
- **Notification System**: Email, Lark/Feishu, and desktop notifications

### Supported Markets
- **Chinese A-Shares**: Full support via TuShare integration
- **Funds**: Mutual funds and ETFs
- **Futures**: Commodity and financial futures
- **Cryptocurrencies**: Digital asset trading (via exchanges)

### Technical Indicators
- Moving Averages (SMA, EMA, WMA)
- MACD (Moving Average Convergence Divergence)
- RSI (Relative Strength Index)
- Bollinger Bands
- Stochastic Oscillator (KDJ)
- BIAS (Price Deviation)
- And 30+ more indicators

## 🏗️ Architecture Overview

```
Qbot Platform Architecture
├── GUI Layer (wxPython)
│   ├── MainFrame - Main application window
│   ├── Trading Panels - Strategy and portfolio management
│   └── Analysis Tools - Performance and risk analysis
├── Core Engine
│   ├── Strategy Framework - Backtrader-based strategy system
│   ├── Data Management - Real-time and historical data
│   ├── Signal Generation - Multi-factor signal fusion
│   └── Risk Management - Position sizing and risk controls
├── AI/ML Layer
│   ├── Traditional ML - Random Forest, SVM, XGBoost
│   ├── Deep Learning - LSTM, CNN, Transformer
│   └── Reinforcement Learning - PPO, DQN, A3C
├── Integration Layer
│   ├── Data Sources - TuShare, Yahoo Finance, Custom APIs
│   ├── Brokers - Simulated and live trading interfaces
│   └── Notifications - Lark, Email, Desktop alerts
└── Utilities
    ├── BaseService - Common functionality
    ├── Configuration - Settings management
    └── Logging - Comprehensive logging system
```

## 📊 Performance Metrics

### Supported Analyzers
- **Sharpe Ratio**: Risk-adjusted returns
- **Maximum Drawdown**: Largest peak-to-trough decline
- **Calmar Ratio**: Annual return / Max drawdown
- **Win Rate**: Percentage of profitable trades
- **Profit Factor**: Gross profit / Gross loss
- **Sortino Ratio**: Downside deviation-adjusted returns

### Risk Management
- **Position Sizing**: Kelly criterion, fixed percentage, volatility-based
- **Stop Losses**: ATR-based, percentage-based, trailing stops
- **Portfolio Limits**: Maximum positions, sector concentration
- **Drawdown Control**: Dynamic position sizing based on drawdown

## 🔧 Installation and Setup

### System Requirements
- **Python**: 3.8 or 3.9 (recommended)
- **Operating System**: Windows, macOS, Linux
- **Memory**: 4GB RAM minimum, 8GB recommended
- **Storage**: 2GB for installation, additional space for data

### Installation Steps
```bash
# Clone the repository
git clone https://github.com/UFund-Me/Qbot --depth 1
cd Qbot

# Install dependencies
pip install -r dev/requirements.txt

# Set environment variables
export PYTHONPATH=${PYTHONPATH}:$(pwd):$(pwd)/backend/multi-fact/mfm_learner

# Run the application
python main.py  # GUI mode
# or
python qbot_main.py  # Real-time trading mode
```

### Configuration
1. **Data Sources**: Configure TuShare token for Chinese market data
2. **Brokers**: Set up paper trading or live broker connections
3. **Notifications**: Configure Lark webhook for alerts
4. **Strategies**: Customize strategy parameters in configuration files

## 📈 Strategy Examples

### 1. Simple Moving Average Crossover
```python
from qbot.strategies.sma_cross_strategy_bt import SmaCross

# Basic usage
cerebro = bt.Cerebro()
cerebro.addstrategy(SmaCross, pfast=10, pslow=30)
results = cerebro.run()
```

### 2. Multi-Factor Strategy
```python
from qbot.strategies.multi_strategy_bt import MultiStrategy

# Combines SMA and RSI signals
cerebro.addstrategy(MultiStrategy, maperiod=15, exitbars=5)
```

### 3. LSTM Deep Learning
```python
from qbot.strategies.lstm_strategy_bt import LSTMPredict

# Neural network-based prediction
cerebro.addstrategy(LSTMPredict, neurons=50, lookback=20)
```

## 🔍 Common Use Cases

### 1. Educational Learning
- Study quantitative trading concepts
- Experiment with different strategies
- Understand risk-return relationships

### 2. Strategy Research
- Develop and test new trading ideas
- Analyze market patterns and anomalies
- Academic research and paper validation

### 3. Portfolio Management
- Multi-asset portfolio optimization
- Risk management and diversification
- Performance monitoring and reporting

### 4. Automated Trading
- Deploy strategies in live markets
- Risk-controlled execution
- Real-time monitoring and alerts

## 🛠️ Development and Contribution

### Code Structure
```
Qbot/
├── main.py                 # GUI application entry point
├── qbot_main.py           # Real-time trading entry point
├── qbot/                  # Core platform code
│   ├── strategies/        # Trading strategies
│   ├── gui/              # GUI components
│   └── plugins/          # Extensions and plugins
├── utils/                # Utility functions
├── pyfunds/              # Fund-specific modules
├── pyfutures/            # Futures trading modules
├── pytrader/             # Web interface
├── docs/                 # Documentation
└── tests/                # Test suites
```

### Contributing
1. **Fork** the repository
2. **Create** a feature branch
3. **Implement** your changes with tests
4. **Submit** a pull request
5. **Follow** the code style guidelines

### Testing
```bash
# Run unit tests
python -m pytest tests/

# Run strategy backtests
python -m pytest tests/test_strategies.py

# Run integration tests
python -m pytest tests/integration/
```

## 📞 Support and Community

### Getting Help
- **Documentation**: Start with this comprehensive guide
- **GitHub Issues**: Report bugs and request features
- **Community**: Join the WeChat group for discussions
- **Email**: Contact the development team

### Resources
- **Official Website**: [Qbot Website](https://ufund-me.github.io/Qbot/#/)
- **GitHub Repository**: [UFund-Me/Qbot](https://github.com/UFund-Me/Qbot)
- **Online Documentation**: [Documentation Portal](https://ufund-me.github.io/Qbot/#/)
- **Tutorial Videos**: Available in the docs/tutorials_code directory

## ⚠️ Important Disclaimers

### Risk Warning
- **Past Performance**: Does not guarantee future results
- **Market Risk**: All trading involves risk of loss
- **Paper Trading**: Test strategies thoroughly before live trading
- **Due Diligence**: Understand strategies before deployment

### Legal Notice
- This software is for educational and research purposes
- Users are responsible for compliance with local regulations
- No investment advice is provided
- Use at your own risk

## 🔄 Version History and Roadmap

### Current Version: 1.0.1
- Multi-strategy framework
- AI/ML integration
- GUI application
- Comprehensive documentation

### Planned Features
- Cloud deployment support
- More AI/ML models
- Additional data sources
- Mobile application
- Advanced portfolio optimization

---

## Navigation Quick Links

| Document | Purpose | Target Audience |
|----------|---------|-----------------|
| [API Documentation](./API_DOCUMENTATION.md) | Complete API reference | Developers, Integrators |
| [Strategy Guide](./STRATEGY_GUIDE.md) | Strategy development | Quants, Researchers |
| [Usage Examples](./USAGE_EXAMPLES.md) | Practical tutorials | All users |
| [Component Reference](./COMPONENT_REFERENCE.md) | Detailed component docs | Developers |

**Start with the document that best matches your needs and experience level. Each document cross-references others for comprehensive coverage.**

---

*Last updated: 2024*
*Version: 1.0.1*
*Platform: Qbot AI Quantitative Trading Platform*