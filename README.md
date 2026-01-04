# cex-market-client

> A unified, async-first client for centralized crypto exchanges, focused exclusively on market data (REST + WebSocket).

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Async](https://img.shields.io/badge/async-first-green.svg)](https://docs.python.org/3/library/asyncio.html)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Why cex-market-client?

Working with crypto exchange APIs is painful. Different exchanges use different formats, undocumented rate limits, unreliable WebSocket connections, and inconsistent error handling. This library solves these problems by providing a **predictable, production-ready interface** for market data.

### The Problems We Solve

- **API Inconsistency**: Each exchange has its own format, endpoints, and conventions. We unify them behind a single interface.
- **Unpredictable Rate Limits**: Limits are often undocumented, change under load, or depend on unknown factors. We make rate limiting explicit and configurable.
- **Unreliable WebSocket Streams**: Connections break silently, reconnect unpredictably, and lose messages. We provide robust connection management with explicit error signaling.
- **Inconsistent Errors**: Each exchange returns errors differently. We provide a unified error taxonomy that separates transport errors from domain errors.
- **Need for Predictability**: High-frequency data consumers need predictable behavior, not just raw speed. We prioritize stability and predictability over maximum throughput.

## What This Library Is (and Isn't)

### ✅ It Is

- A **low-level infrastructure library** for market data
- **Async-first** and production-ready
- **Explicit** about rate limits, errors, and connection states
- **Unified** interface across different exchanges
- Built for **long-running services**

### ❌ It Is Not

- ❌ A trading bot or strategy framework
- ❌ An arbitrage system
- ❌ A trading execution library (no order placement, position management, etc.)
- ❌ A high-level abstraction that hides complexity
- ❌ Suitable for simple one-off scripts

## Quick Start

### Installation

```bash
pip install cex-market-client
```

### Basic Usage

```python
import asyncio
from cex_market_client import BybitClient

async def main():
    # Initialize client
    client = BybitClient()
    
    # REST API - Get current ticker
    ticker = await client.rest.get_ticker("BTCUSDT")
    print(f"Price: ${ticker.last_price:,.2f}")
    print(f"24h Change: {ticker.change_24h:.2f}%")
    
    # REST API - Get order book
    orderbook = await client.rest.get_orderbook("BTCUSDT", depth=10)
    print(f"Best Bid: ${orderbook.bids[0].price}")
    print(f"Best Ask: ${orderbook.asks[0].price}")
    
    # WebSocket - Stream trades in real-time
    async for trade in client.ws.stream_trades("BTCUSDT"):
        print(f"Trade: {trade.price} @ {trade.size} ({trade.side})")

if __name__ == "__main__":
    asyncio.run(main())
```

### Advanced Usage

```python
import asyncio
from cex_market_client import BybitClient
from cex_market_client.exceptions import RateLimitExceeded, NetworkError

async def robust_data_collector():
    client = BybitClient(
        rate_limits={
            "global": {"rate": 100, "period": 60},  # Conservative limits
        }
    )
    
    try:
        # Stream orderbook updates with automatic reconnection
        async for update in client.ws.stream_orderbook("BTCUSDT"):
            # Process orderbook delta
            print(f"Update: {update.sequence_id}")
            
    except RateLimitExceeded as e:
        print(f"Rate limit hit. Retry after {e.retry_after}s")
        
    except NetworkError as e:
        print(f"Network issue: {e}. Connection will auto-reconnect.")

if __name__ == "__main__":
    asyncio.run(robust_data_collector())
```

## Features

### 🚀 Core Features

- **Unified REST & WebSocket Interfaces**: Same API across different exchanges
- **Normalized Data Models**: Consistent domain models regardless of exchange format
- **Explicit Rate Limiting**: Configurable token bucket algorithm with per-endpoint limits
- **Robust Error Handling**: Comprehensive error taxonomy with full context
- **Automatic Reconnection**: WebSocket streams automatically reconnect with exponential backoff
- **Async-First Design**: Built for high-concurrency scenarios
- **Type Hints**: Full type coverage for better IDE support and fewer bugs

### 🎯 Design Principles

- **Predictability > Speed**: We prioritize stable, predictable behavior
- **Explicit > Implicit**: No hidden magic—everything is configurable and observable
- **Fail-Fast**: Errors are detected early and signaled explicitly
- **Separation of Concerns**: Clear boundaries between transport, domain, and adapters

## Supported Exchanges

| Exchange | REST API | WebSocket | Status |
|----------|----------|-----------|--------|
| Bybit    | ✅        | ✅         | Planned |
| Binance  | ⏳        | ⏳         | Planned |

*More exchanges coming soon based on community demand.*

## Documentation

Comprehensive documentation is available:

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - System architecture and component design
- **[DESIGN_GOALS.md](DESIGN_GOALS.md)**: Design philosophy and trade-offs
- **[RATE_LIMITING.md](RATE_LIMITING.md)**: Rate limiting strategies and configuration
- **[ERROR_MODEL.md](ERROR_MODEL.md)**: Error taxonomy and handling
- **[ROADMAP.md](ROADMAP.md)**: Development roadmap and future plans

## Who Should Use This Library

This library is designed for:

- **Backend engineers** building market data infrastructure
- **Trading infrastructure developers** needing reliable exchange connectivity
- **Data collection systems** that need robust, long-running connections
- **Anyone building production services** that consume exchange APIs

## When NOT to Use This Library

- **Simple one-off scripts**: For a single API call, use `requests` or `httpx` directly
- **Trading operations**: If you need to place orders, this library doesn't support that
- **Maximum speed at any cost**: We sacrifice some speed for predictability
- **High-level abstractions**: If you want complexity hidden, look elsewhere

## Error Handling

The library provides a comprehensive error model:

```python
from cex_market_client.exceptions import (
    RateLimitExceeded,
    NetworkError,
    InvalidSymbol,
    ExchangeError
)

try:
    ticker = await client.rest.get_ticker("BTCUSDT")
except RateLimitExceeded as e:
    # Rate limit exceeded - wait and retry
    await asyncio.sleep(e.retry_after)
    ticker = await client.rest.get_ticker("BTCUSDT")
except InvalidSymbol as e:
    # Symbol doesn't exist
    print(f"Invalid symbol: {e.symbol}")
except NetworkError as e:
    # Transport-level error - may be retriable
    logger.error(f"Network error: {e}", exc_info=True)
except ExchangeError as e:
    # Exchange returned an error
    print(f"Exchange error {e.error_code}: {e.message}")
```

See [ERROR_MODEL.md](ERROR_MODEL.md) for the complete error taxonomy.

## Rate Limiting

Rate limiting is explicit and configurable:

```python
from cex_market_client import BybitClient

client = BybitClient(
    rate_limits={
        "global": {"rate": 120, "period": 60},  # 120 requests per minute
        "endpoints": {
            "/v2/public/tickers": {"rate": 120, "period": 60},
            "/v2/public/orderBook": {"rate": 50, "period": 60},
        }
    }
)
```

The library uses a token bucket algorithm and automatically handles HTTP 429 responses. See [RATE_LIMITING.md](RATE_LIMITING.md) for details.

## Requirements

- Python 3.8+
- asyncio-compatible environment
- Network access to exchange APIs

## Contributing

Contributions are welcome! This project is in early development. Areas where help is needed:

- Exchange adapters (Bybit, Binance, etc.)
- Documentation improvements
- Test coverage
- Performance optimizations

Please read the architecture documentation before contributing to understand the design principles.

## Roadmap

See [ROADMAP.md](ROADMAP.md) for the full development roadmap.

### MVP (Current Focus)

- ✅ Core architecture and design documentation
- ⏳ Bybit REST API support
- ⏳ Bybit WebSocket support
- ⏳ Rate limiting implementation
- ⏳ Error handling framework

### Upcoming

- Binance support
- Additional exchange adapters
- Enhanced WebSocket features
- Performance optimizations
- Monitoring and observability

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

Built for developers who need reliable, predictable access to crypto market data. No magic, no surprises—just solid infrastructure.

---

**Note**: This library is in early development. API may change before v1.0. See [ROADMAP.md](ROADMAP.md) for current status.
