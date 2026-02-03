# Binance-Futures-Testnet

1.Project structure:

binance_futures/
├── config.py
├── client.py
├── exceptions.py
├── logger.py
├── main.py
└── requirements.txt

2.requirements.txt:

requests
python-dotenv

3.config.py:

import os
from dotenv import load_dotenv

load_dotenv()

BINANCE_TESTNET_BASE_URL = "https://testnet.binancefuture.com"

API_KEY = os.getenv("BINANCE_API_KEY")
API_SECRET = os.getenv("BINANCE_API_SECRET")

REQUEST_TIMEOUT = 10

4.logger.py:

import logging

def setup_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)

    formatter = logging.Formatter(
        "[%(asctime)s] [%(levelname)s] %(name)s - %(message)s"
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)

    if not logger.handlers:
        logger.addHandler(handler)

    return logger
5.exceptions.py:

class BinanceAPIException(Exception):
    pass


class BinanceRequestException(Exception):
    pass
    
6.client.py:

import time
import hmac
import hashlib
import requests
from urllib.parse import urlencode

from config import (
    BINANCE_TESTNET_BASE_URL,
    API_KEY,
    API_SECRET,
    REQUEST_TIMEOUT,
)
from logger import setup_logger
from exceptions import BinanceAPIException, BinanceRequestException

logger = setup_logger("BinanceClient")


class BinanceFuturesClient:
    def __init__(self):
        if not API_KEY or not API_SECRET:
            raise ValueError("API_KEY and API_SECRET must be set")

        self.base_url = BINANCE_TESTNET_BASE_URL
        self.session = requests.Session()
        self.session.headers.update({"X-MBX-APIKEY": API_KEY})

    def _sign_params(self, params: dict) -> dict:
        query_string = urlencode(params)
        signature = hmac.new(
            API_SECRET.encode(),
            query_string.encode(),
            hashlib.sha256,
        ).hexdigest()

        params["signature"] = signature
        return params

    def _request(self, method: str, path: str, params: dict = None, signed: bool = False):
        url = f"{self.base_url}{path}"
        params = params or {}

        if signed:
            params["timestamp"] = int(time.time() * 1000)
            params = self._sign_params(params)

        try:
            response = self.session.request(
                method=method,
                url=url,
                params=params,
                timeout=REQUEST_TIMEOUT,
            )
        except requests.RequestException as e:
            logger.error(f"Request error: {e}")
            raise BinanceRequestException(str(e))

        if response.status_code != 200:
            logger.error(f"API error [{response.status_code}]: {response.text}")
            raise BinanceAPIException(response.text)

        return response.json()

    # ==========================
    # Public / Account Endpoints
    # ==========================

    def get_account_info(self):
        return self._request("GET", "/fapi/v2/account", signed=True)

    def place_order(
        self,
        symbol: str,
        side: str,
        order_type: str,
        quantity: float,
        price: float | None = None,
        time_in_force: str = "GTC",
    ):
        params = {
            "symbol": symbol,
            "side": side.upper(),
            "type": order_type.upper(),
            "quantity": quantity,
        }

        if order_type.upper() == "LIMIT":
            if price is None:
                raise ValueError("LIMIT orders require a price")
            params.update({
                "price": price,
                "timeInForce": time_in_force,
            })

        logger.info(f"Placing order: {params}")
        return self._request("POST", "/fapi/v1/order", params=params, signed=True)
        
7.main.py:

from client import BinanceFuturesClient
from logger import setup_logger
from exceptions import BinanceAPIException, BinanceRequestException

logger = setup_logger("Main")


def main():
    client = BinanceFuturesClient()

    try:
        account = client.get_account_info()
        logger.info(f"Account balance: {account['totalWalletBalance']} USDT")

        order = client.place_order(
            symbol="BTCUSDT",
            side="BUY",
            order_type="LIMIT",
            quantity=0.001,
            price=20000,
        )

        logger.info(f"Order placed successfully: {order}")

    except (BinanceAPIException, BinanceRequestException, ValueError) as e:
        logger.error(f"Operation failed: {e}")


if __name__ == "__main__":
    main()
    
8.Environment variables (.env):

BINANCE_API_KEY=your_testnet_key
BINANCE_API_SECRET=your_testnet_secret
