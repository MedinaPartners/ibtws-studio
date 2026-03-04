# AI MongoDB 資料存取指南

## 背景說明

這份 prompt 是用來讓 AI 助手了解如何主動存取使用者的 MongoDB 資料庫，以便在對話中直接獲取相關上下文，而不需要反覆詢問使用者提供資料。使用者經營一個涵蓋投資組合管理、市場數據、選擇權分析等多個面向的系統，所有核心資料都儲存在 MongoDB Atlas 雲端資料庫中。當使用者詢問與這些資料相關的問題時，AI 應該主動連接資料庫查詢，直接進入對話核心，而不是要求使用者手動提供 CSV 或截圖。

## MongoDB 連線方式

所有專案都使用統一的連線模組來存取 MongoDB。主要的連線程式位於 `/Users/chouwilliam/Medina/pm/pkg/mongo_connector.py`，這個模組會從 `/Users/chouwilliam/Medina/pm/pkg/config/database.json` 讀取連線字串。連線方式很簡單，只需要在 Python 環境中執行以下程式碼即可取得資料庫連線：

```python
import sys
sys.path.insert(0, '/Users/chouwilliam/Medina/pm')
from pkg.mongo_connector import MongoConnector

mongo = MongoConnector()
client = mongo.get_cloud_conn()
db = client['DATABASE_NAME']  # 替換成需要的資料庫名稱
```

執行 Python 腳本時，需要先進入 pm 專案目錄並啟用虛擬環境：`cd /Users/chouwilliam/Medina/pm && source venv/bin/activate`。

## 主要資料庫與用途

以下是目前系統中的主要資料庫，按照功能分類說明。每個資料庫都有 DEV 和 PROD 兩個版本，查詢時應該優先使用 PROD 版本以獲取最新的真實資料。

### 投資組合與交易相關

IBFLEX_PROD 是最重要的投資組合資料庫，儲存從 Interactive Brokers Flex Query 自動抓取的資料。這個資料庫包含以下 collections：position_daily 儲存每日股票持倉快照，包含 account_id、report_date、symbol、position（股數）、close_price、position_value_usd 等欄位；nav_daily 儲存每日淨值，包含各帳戶的 cash、stock、options、total_usd 等欄位；trade_history 儲存交易紀錄，包含 trade_date、symbol、buy_sell、quantity、trade_price、fifo_pnl_realized 等欄位；cash_flow 儲存現金流資料；benchmark_daily 儲存 SPY、QQQ、IWM 等基準指數的每日收盤價和報酬率。

當使用者詢問關於美股持倉的問題時，AI 應該主動查詢 IBFLEX_PROD.position_daily 來獲取當前持倉，查詢 IBFLEX_PROD.trade_history 來了解建倉歷程，查詢 IBFLEX_PROD.nav_daily 來計算組合占比。

TW_NAS_PROD 儲存台股相關資料，包含 holdings（持倉）、trading_records（交易紀錄）、profits_by_symbol（個股損益）等 collections。TW_NAS_RAW_PROD 則儲存從券商系統抓取的原始資料。

PM_PROD 是投資組合管理的核心資料庫，儲存經過處理的組合分析資料。

### 市場數據相關

THETAV2_PROD 儲存美股選擇權相關資料，包含 options_eod（選擇權日終資料）、options_greeks（希臘字母）、options_oi（未平倉量）等。當使用者詢問選擇權相關問題時可以查詢這個資料庫。

FMPV2_PROD 儲存從 Financial Modeling Prep API 抓取的基本面資料。FMPV2JP_PROD 則是日本市場的對應資料。

BEACON_MONITOR_PROD 儲存市場監控相關資料，包含 sp500_market_breadth（市場廣度指標）、global_indices（全球指數）等。

### 其他系統

BRAIN_V2_PROD 儲存量化分析相關資料。MODELS_PROD 儲存機器學習模型相關資料。CORTEXV2_PROD 儲存 AI 助手相關資料。HUB_PROD 儲存中央協調系統資料。

## 常用查詢範例

當使用者詢問某檔股票的持倉狀況時，AI 應該執行類似以下的查詢：

```python
# 查詢特定股票的當前持倉
db = client['IBFLEX_PROD']
positions = list(db['position_daily'].find(
    {'symbol': 'AAPL'},
    {'account_id': 1, 'report_date': 1, 'position': 1, 'close_price': 1, 'position_value_usd': 1}
).sort('report_date', -1).limit(10))

# 查詢該股票的交易紀錄
trades = list(db['trade_history'].find(
    {'symbol': 'AAPL'},
    {'trade_date': 1, 'buy_sell': 1, 'quantity': 1, 'trade_price': 1, 'fifo_pnl_realized': 1}
).sort('trade_date', -1))

# 查詢最新的組合總覽
latest_nav = list(db['nav_daily'].find().sort('report_date', -1).limit(10))
```

當使用者詢問整體組合狀況時，AI 應該同時查詢 nav_daily 和 position_daily，計算各持倉的組合占比，並按市值排序呈現。

## 行為準則

第一，當對話涉及使用者的投資組合、持倉、交易紀錄等資料時，AI 應該主動連接 MongoDB 查詢，而不是要求使用者提供資料。這能讓對話更流暢，直接進入核心討論。

第二，查詢結果應該以清晰的格式呈現，包含日期、數量、價格、市值等關鍵資訊。如果資料量大，應該做適當的彙總和排序。

第三，查詢時應該注意資料的時效性，優先使用最新的 report_date 資料。如果需要歷史資料，應該明確說明資料的時間範圍。

第四，如果查詢失敗或找不到資料，應該告知使用者並詢問是否需要檢查資料庫連線或資料是否存在。

第五，在呈現持倉資料時，應該計算組合占比，讓使用者了解各部位的相對大小。這需要同時查詢 nav_daily 來獲取總資產價值。

## 讀完這份 prompt 後

AI 助手在讀完這份 prompt 後，應該具備主動查詢 MongoDB 的能力。當使用者的問題涉及投資組合相關資料時，不要詢問使用者提供資料，而是直接連接資料庫查詢，然後基於查詢結果進行分析和討論。這樣的互動方式能讓對話更有效率，直接切入問題核心。
