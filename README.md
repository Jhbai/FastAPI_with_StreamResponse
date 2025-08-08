<img width="1500" height="950" alt="異步 + 串流" src="https://github.com/user-attachments/assets/7bde86e9-e553-44d1-9b09-b038fbf4d5ab" />

FastAPI 異步串流與 StreamResponse 實作範例
本專案旨在透過一個簡單的範例，深度解析如何在 Python FastAPI 框架中，利用 asyncio 與 StreamResponse 實現異步（Asynchronous）的串流資料傳輸。

這個技術對於需要長時間運行的背景任務特別有用，例如：

大型語言模型（LLM）的即時推理（Inference）。

分塊（Chunking）處理大型資料集。

與速度較慢的外部 API 進行通訊。

透過串流響應，客戶端無需等待整個任務完成，而是可以即時接收到部分生成的資料，大幅提升了應用的互動性與使用者體驗。

系統架構圖
下圖展示了本專案中兩種不同 API 端點的處理流程：一個是標準的同步響應，另一個則是異步的串流響應。

（請將此處的圖片連結替換為您上傳到 GitHub 或其他圖床的圖片實際連結）

核心概念解析
Event Loop (事件循環):
asyncio 的核心，負責調度所有異步任務。當一個任務（例如 await heavy_io()）進入等待 I/O 操作的狀態時，事件循環會暫停該任務，並切換到另一個就緒的任務，從而實現非阻塞式（Non-blocking）的並行處理。

StreamResponse (串流響應):
FastAPI 允許響應一個「異步生成器」（Asynchronous Generator）。StreamResponse 會迭代這個生成器，每當生成器 yield 一個資料塊時，就立即將其發送給客戶端，直到生成器執行完畢。

異步生成器 (async def with yield):
這是實現串流內容的關鍵。函數使用 async def 定義，並在其中使用 yield 來回傳資料。每次 yield 之後，函數的執行狀態會被保存，並將控制權交還給事件循環。當客戶端準備好接收下一個資料塊時，事件循環會從上次暫停的地方繼續執行該函數。

run_in_executor:
在異步環境中，絕對不能直接運行會阻塞的同步程式碼（例如 CPU 密集型計算或傳統的阻塞 I/O）。loop.run_in_executor 方法可以將這類阻塞任務提交到一個獨立的線程池（Thread Pool）中執行，從而避免主事件循環被阻塞。在圖中，heavy_io() 就是一個模擬的阻塞任務。

API 端點說明
1. /infer/home - 標準同步端點
@router.get("/home")
async def stream_response():
    return "OK"

行為: 這是一個標準的 FastAPI 異步端點。當請求到達時，它會立即處理並一次性返回完整的響應 "OK"。

流程: 請求 -> 立即處理 -> 返回完整響應

2. /infer/predict - 異步串流端點
# 模擬一個耗時的 I/O 或計算任務
def heavy_io():
    data = ["你好", "這是", "測試", "Stream", "輸出", "並且", "同步", "發送", "訊息"]
    time.sleep(1)
    return data

# 異步生成器，用於產生資料流
async def fake_stream():
    loop = asyncio.get_running_loop()
    # 在 executor 中運行阻塞函數，避免阻塞事件循環
    result = await loop.run_in_executor(None, heavy_io)
    for item in result:
        yield f"{item}\n"
        await asyncio.sleep(0.5) # 模擬每個資料塊之間的處理延遲

@router.get("/predict")
async def stream_response():
    return StreamResponse(fake_stream(), media_type="text/plain")

行為: 這個端點會返回一個 StreamResponse。當客戶端連接後，伺服器會逐步執行 fake_stream 生成器，每 yield 一次資料，就將該資料塊傳送給客戶端。

流程:

客戶端發起請求。

FastAPI 開始迭代 fake_stream 生成器。

heavy_io() 被提交到線程池執行，主事件循環不會被阻塞。

heavy_io() 完成後，for 迴圈開始。

第一次 yield "你好\n"，"你好" 被發送給客戶端。

await asyncio.sleep(0.5)，事件循環切換到其他任務。

0.5 秒後，繼續執行，第二次 yield "這是\n"，"這是" 被發送給客戶端。

重複此過程，直到生成器執行完畢。

如何運行
安裝依賴:

pip install "fastapi[all]"

保存程式碼:
將上述 Python 程式碼保存為 main.py。

啟動伺服器:

uvicorn main:app --reload

伺服器將在 http://127.0.0.1:8000 上運行。

測試輸出
您可以使用 curl 工具來觀察串流的效果。

測試 /infer/home:

curl http://127.0.0.1:8000/infer/home

輸出:

"OK"

(響應會立即返回)

測試 /infer/predict:
使用 -N 參數來禁用 curl 的緩衝，以便即時看到輸出。

curl -N http://127.0.0.1:8000/infer/predict

輸出:

你好
(等待 0.5 秒)
這是
(等待 0.5 秒)
測試
(等待 0.5 秒)
Stream
(等待 0.5 秒)
輸出
(等待 0.5 秒)
並且
(等待 0.5 秒)
同步
(等待 0.5 秒)
發送
(等待 0.5 秒)
訊息

(你會看到訊息逐行出現，而不是一次性全部顯示)
