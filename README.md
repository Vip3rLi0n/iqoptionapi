## Community based API for IQ Option
- Do note that I am not actively developing this API *publicly* so I may or may not update this repository at all.
- PLEASE, feel free to do pull request or create an issue if you found an issue that I might be able to fix.

# Use the original documentation which is:
https://iqoptionapi.github.io/iqoptionapi

# Using session.json
- This is the template for now, since I am not configuring a proper format yet. If you want to skip 2fa/login, just login from browser and get the ssid and the timestamp and paste here.
```json
{
  "ssid": "example123",
  "cookies": {
    "Iq": "GOOD",
    "ssid": "example123"
  },
  "timestamp": "2024-03-11T02:51:17.484Z"
}
```

# Picture
![image](https://github.com/user-attachments/assets/14e5e058-8ec4-4598-9852-f40f10ca6807)

# Script example (Not perfect since it doesn't really save the session properly yet, as using connect() will leads to logout once finished.)
```python
import time
import json
import os
import logging
from iqoptionapi.stable_api import IQ_Option
from dotenv import load_dotenv
from datetime import datetime, timedelta, timezone
from pathlib import Path

# Configure logging
logging.basicConfig(level=logging.ERROR, format='%(asctime)s - %(levelname)s - %(message)s')

# Load environment variables
load_dotenv()
email = os.getenv('EMAIL')
password = os.getenv('PASS')

# Session file path
SESSION_FILE = Path('session.json')

# Default headers
header = {
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.1.0 Safari/605.1.1"
}

def save_session(api, ssid, cookies):
    """Save the session data and cookies to a JSON file."""
    session_data = {
        'ssid': ssid,
        'cookies': cookies,
        'timestamp': datetime.now(timezone.utc).isoformat()
    }
    try:
        with SESSION_FILE.open('w') as f:
            json.dump(session_data, f, indent=4)
        logging.info(f"[?] Session saved successfully")
    except Exception as e:
        logging.error(f"[!] Failed to save session: {str(e)}")

def load_session(api):
    """Load session data and attempt to reconnect without login."""
    if not SESSION_FILE.exists():
        logging.info(f"[!] No session file found")
        return False, None, None
    
    print(f"[?] Session found, connecting using the session provided.")

    try:
        with SESSION_FILE.open('r') as f:
            session_data = json.load(f)
        
        ssid = session_data.get('ssid')
        cookies = session_data.get('cookies', {})
        # Parse timestamp and ensure it's offset-aware
        timestamp_str = session_data.get('timestamp', '1970-01-01T00:00:00')
        if '+' not in timestamp_str and 'Z' not in timestamp_str:
            timestamp_str += '+00:00'
        timestamp = datetime.fromisoformat(timestamp_str.replace('Z', '+00:00'))
        
        # Check if session is still valid (within 30 days)
        if datetime.now(timezone.utc) - timestamp > timedelta(days=30):
            logging.error(f"[!] Session expired")
            return False, None, None

        # Attempt to reconnect with cookies from session.json
        logging.info(f"[?] Attempting to reconnect with saved session")
        check, reason = api.connect_without_login(cookies=cookies)
        
        # Verify connection
        if check and api.check_connect():
            logging.info(f"[?] Reconnected successfully using saved session")
            return True, ssid, cookies
        
        logging.error(f"[!] Saved session invalid: {reason}")
        return False, None, None
    
    except Exception as e:
        logging.error(f"[!] Failed to load session: {str(e)}")
        return False, None, None

def main():
    print(f"[?] Connecting...")
    
    # Initialize API
    api = IQ_Option(email, password)
    
    # Try loading saved session
    success, ssid, cookies = load_session(api)
    
    if not success and SESSION_FILE.exists():
        logging.error(f"[!] Session file exists but reconnection failed. Delete session.json to attempt a new login.")
        return
    
    if not success:
        # Perform full login
        api.set_session(header, {'Iq': 'GOOD'})
        status, reason = api.connect()
        print(f"[?] Result")
        print(f"[Status] {status}")
        print(f"[Reason] {reason}")
        print(f"[Email] {email}")
        
        if not status:
            logging.error(f"[!] Login failed")
            return
        
        # Save session data
        ssid = api.ssid
        if not ssid:
            logging.error(f"[!] Failed to retrieve SSID after login")
            return
        cookies = api.get_session_cookies()
        save_session(api, ssid, cookies)
    else:
        print(f"[?] Reconnected using saved session")
        print(f"[Email] {email}")
    
    # Switch to practice account
    api.change_balance("PRACTICE")
    
    # Fetch balance
    def get_usd_balance(api):
        balances = api.get_balances()
        if isinstance(balances, dict) and 'msg' in balances:
            for balance in balances['msg']:
                if balance.get('currency') == 'USD':
                    return balance.get('amount')
        return None

    bal = get_usd_balance(api)
    if bal is not None:
        print(f"[Balance] ${bal:.2f}")
    else:
        print(f"[!] Failed to fetch USD balance")

if __name__ == "__main__":
    main()
```
