# =============================
# 📊 Futures Long/Short Dashboard — Streamlit Version
# =============================

import streamlit as st
import requests
from bs4 import BeautifulSoup
import pandas as pd

# -----------------------------
# CONFIG
# -----------------------------

URL = "https://tradingtick.com/futures/longshort"

# -----------------------------
# MAIN FUNCTION
# -----------------------------

@st.cache_data(ttl=900)
def get_top_long_short():
    response = requests.get(URL)
    soup = BeautifulSoup(response.text, "html.parser")
    table = soup.find("table")

    if table is None:
        st.error("❌ Could not find table on the page.")
        return None, None

    df = pd.read_html(str(table))[0]

    # Rename columns as needed
    if 'Price % Chg' in df.columns and 'OI % Chg' in df.columns:
        df = df.rename(columns={'Price % Chg': 'Price Change', 'OI % Chg': 'OI Change'})
    elif 'Price Change' in df.columns and 'OI Change' in df.columns:
        pass
    else:
        st.error(f"❌ Unexpected columns: {df.columns}")
        return None, None

    df['Price Change'] = pd.to_numeric(df['Price Change'], errors='coerce')
    df['OI Change'] = pd.to_numeric(df['OI Change'], errors='coerce')
    df['OI'] = pd.to_numeric(df['OI'], errors='coerce')

    def signal(row):
        if row['OI Change'] > 0 and row['Price Change'] > 0:
            return 'Long Build Up'
        elif row['OI Change'] > 0 and row['Price Change'] < 0:
            return 'Short Build Up'
        elif row['OI Change'] < 0 and row['Price Change'] > 0:
            return 'Short Covering'
        elif row['OI Change'] < 0 and row['Price Change'] < 0:
            return 'Long Unwinding'
        else:
            return 'No Signal'

    df['Signal'] = df.apply(signal, axis=1)

    longs = df[df['Signal'] == 'Long Build Up'].sort_values(by='OI', ascending=False).head(5)
    shorts = df[df['Signal'] == 'Short Build Up'].sort_values(by='OI', ascending=False).head(5)

    return longs, shorts

# -----------------------------
# STREAMLIT APP
# -----------------------------

st.set_page_config(page_title="Futures Long/Short Dashboard", layout="wide")

st.title("📈 NSE Futures Long & Short Positions Dashboard")
st.markdown("Real-time Top 5 by Open Interest (TradingTick Source)")

longs, shorts = get_top_long_short()

if longs is not None and shorts is not None:
    col1, col2 = st.columns(2)

    with col1:
        st.subheader("🟢 Top 5 Long Positions (Long Build Up)")
        st.dataframe(longs[['Symbol', 'OI', 'Price Change', 'OI Change']].reset_index(drop=True))

    with col2:
        st.subheader("🔴 Top 5 Short Positions (Short Build Up)")
        st.dataframe(shorts[['Symbol', 'OI', 'Price Change', 'OI Change']].reset_index(drop=True))

    st.info("✅ Data updates automatically every 15 minutes.")
else:
    st.warning("⚠️ Could not fetch data. Please check the page or try again later.")
