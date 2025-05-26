import os



# यदि 'requests' इंस्टॉल नहीं है, तो इंस्टॉल करें
try:
    import requests
except ImportError:
    os.system('pip install requests')
    import requests

# यदि 'dateutil' इंस्टॉल नहीं है, तो इंस्टॉल करें
try:
    import dateutil
    from dateutil.parser import parse
except ImportError:
    os.system('pip install python-dateutil')
    from dateutil.parser import parse


def get_enctoken(userid, password, twofa):
    session = requests.Session()

    # Step 1: Kite username & password से login
    login_resp = session.post('https://kite.zerodha.com/api/login', data={
        "user_id": userid,
        "password": password
    })

    # Step 2: दोबारा 2FA (TOTP/PIN) भेजना
    if login_resp.status_code == 200 and "data" in login_resp.json():
        request_id = login_resp.json()['data']['request_id']
        user_id = login_resp.json()['data']['user_id']
    else:
        raise Exception("Login Failed - Check Username/Password")

    twofa_resp = session.post('https://kite.zerodha.com/api/twofa', data={
        "request_id": request_id,
        "twofa_value": twofa,
        "user_id": user_id
    })

    # Enctoken प्राप्त करें
    enctoken = twofa_resp.cookies.get('enctoken')
    if enctoken:
        return enctoken
    else:
        raise Exception("2FA Failed - Check TOTP or PIN")


import requests
class KiteApp:
    # ✅ Class-level constants (Zerodha के लिए जरूरी मानक):

    # ट्रेडिंग प्रोडक्ट के प्रकार
    PRODUCT_MIS = "MIS"  # Intraday order
    PRODUCT_CNC = "CNC"  # Delivery
    PRODUCT_NRML = "NRML"  # Futures/Options normal
    PRODUCT_CO = "CO"  # Cover Order

    # ऑर्डर टाइप
    ORDER_TYPE_MARKET = "MARKET"  # Market Price
    ORDER_TYPE_LIMIT = "LIMIT"  # Limit Price
    ORDER_TYPE_SLM = "SL-M"  # Stop Loss Market
    ORDER_TYPE_SL = "SL"  # Stop Loss Limit

    # Zerodha वेरायटी
    VARIETY_REGULAR = "regular"  # Regular Order
    VARIETY_CO = "co"  # Cover Order
    VARIETY_AMO = "amo"  # After Market Order

    # खरीद / बिक्री
    TRANSACTION_TYPE_BUY = "BUY"
    TRANSACTION_TYPE_SELL = "SELL"

    # ऑर्डर वैलिडिटी
    VALIDITY_DAY = "DAY"
    VALIDITY_IOC = "IOC"

    # एक्सचेंज
    EXCHANGE_NSE = "NSE"
    EXCHANGE_BSE = "BSE"
    EXCHANGE_NFO = "NFO"
    EXCHANGE_CDS = "CDS"
    EXCHANGE_BFO = "BFO"
    EXCHANGE_MCX = "MCX"

    # ✅ Constructor: जब आप class का object बनाते हैं, यह function चलता है
    def __init__(self, enctoken):
        self.enctoken = enctoken
        self.headers = {"Authorization": f"enctoken {self.enctoken}"}
        self.session = requests.Session()
        self.root_url = "https://kite.zerodha.com/oms"

        # Zerodha Kite के साथ session चालू करें
        self.session.get(self.root_url, headers=self.headers)
    def instruments(self, exchange=None):
        url = f"{self.root_url}/instruments"
        data = self.session.get(url, headers=self.headers).text.split("\n")

        result = []
        for line in data[1:-1]:
            row = line.split(",")
            if exchange is None or exchange == row[11]:
                record = {
                    'instrument_token': int(row[0]),
                    'exchange_token': row[1],
                    'tradingsymbol': row[2],
                    'name': row[3][1:-1],
                    'last_price': float(row[4]),
                    'expiry': parse(row[5]).date() if row[5] else None,
                    'strike': float(row[6]),
                    'tick_size': float(row[7]),
                    'lot_size': int(row[8]),
                    'instrument_type': row[9],
                    'segment': row[10],
                    'exchange': row[11],
                }
                result.append(record)
        return result
    def quote(self, instruments):
        url = f"{self.root_url}/quote"
        params = {"i": instruments}
        return self.session.get(url, params=params, headers=self.headers).json()["data"]

    def ltp(self, instruments):
        url = f"{self.root_url}/quote/ltp"
        params = {"i": instruments}
        return self.session.get(url, params=params, headers=self.headers).json()["data"]
    def historical_data(self, instrument_token, from_date, to_date, interval, continuous=False, oi=False):
        url = f"{self.root_url}/instruments/historical/{instrument_token}/{interval}"
        params = {
            "from": from_date,
            "to": to_date,
            "interval": interval,
            "continuous": 1 if continuous else 0,
            "oi": 1 if oi else 0
        }

        candles = self.session.get(url, params=params, headers=self.headers).json()["data"]["candles"]
        result = []
        for item in candles:
            record = {
                "date": parse(item[0]),
                "open": item[1],
                "high": item[2],
                "low": item[3],
                "close": item[4],
                "volume": item[5]
            }
            if len(item) == 7:
                record["oi"] = item[6]
            result.append(record)
        return result
    def margins(self):
        url = f"{self.root_url}/user/margins"
        return self.session.get(url, headers=self.headers).json()["data"]

    def orders(self):
        url = f"{self.root_url}/orders"
        return self.session.get(url, headers=self.headers).json()["data"]

    def positions(self):
        url = f"{self.root_url}/portfolio/positions"
        return self.session.get(url, headers=self.headers).json()["data"]

    def place_order(self, variety, exchange, tradingsymbol, transaction_type, quantity,
                    product, order_type, price=None, validity=None,
                    disclosed_quantity=None, trigger_price=None, squareoff=None,
                    stoploss=None, trailing_stoploss=None, tag=None):
        url = f"{self.root_url}/orders/{variety}"
        params = locals()
        del params["self"]
        data = {k: v for k, v in params.items() if v is not None}
        return self.session.post(url, data=data, headers=self.headers).json()["data"]["order_id"]

    def modify_order(self, variety, order_id, parent_order_id=None, quantity=None,
                     price=None, order_type=None, trigger_price=None, validity=None,
                     disclosed_quantity=None):
        url = f"{self.root_url}/orders/{variety}/{order_id}"
        params = locals()
        del params["self"]
        data = {k: v for k, v in params.items() if v is not None}
        return self.session.put(url, data=data, headers=self.headers).json()["data"]["order_id"]

    def cancel_order(self, variety, order_id, parent_order_id=None):
        url = f"{self.root_url}/orders/{variety}/{order_id}"
        data = {"parent_order_id": parent_order_id} if parent_order_id else {}
        return self.session.delete(url, data=data, headers=self.headers).json()["data"]["order_id"]
