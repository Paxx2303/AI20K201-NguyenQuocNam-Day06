# 📎 Extras — Day 06

**Họ tên:** Nguyễn Quốc Nam | **Student ID:** AI20K201

---

## 🗂️ Prompt Test Logs

Ghi lại các prompt thực tế đã dùng với AI assistant trong quá trình build hệ thống VinFast RAG.

---

### Session 1 — Khởi tạo tools

**Prompt:**
> "tạo tools để lấy dữ liệu liên quan cho agent"

**Context cung cấp:** `community_crawler.py`, `pipeline.py`

**Kết quả:** AI hỏi lại framework (LangChain), loại filter cần thiết, có crawl on-demand không → gen `vinfast_tools.py` với 3 tools.

**Nhận xét:** AI hỏi làm rõ yêu cầu trước giúp output sát thực tế — nếu trả lời đủ context ngay từ đầu sẽ nhanh hơn 1 vòng.

---

### Session 2 — Inspect data trước khi gen tool

**Prompt:**
> "kiểm tra các bảng data trước r hẵng gen tool"

**Script AI cung cấp:**
```python
from qdrant_client import QdrantClient
from collections import Counter

client = QdrantClient(host="localhost", port=7333)
print(client.get_collection("vinfast_rag"))

results, _ = client.scroll("vinfast_rag", limit=3, with_payload=True, with_vectors=False)
for r in results:
    print(r.payload)

all_points, _ = client.scroll("vinfast_rag", limit=10000, with_payload=True, with_vectors=False)
print("doc_type:", dict(Counter(p.payload.get("doc_type") for p in all_points)))
print("model:",    dict(Counter(p.payload.get("model")    for p in all_points)))
```

**Output thực tế:**
```
doc_type: {'spec': 186, 'faq': 24, 'forum_qa': 126, 'installment': 3}
model: {'VF 3': 7, 'VF 5': 24, 'VF 6': 33, 'VF 7': 21,
        'VF 8': 73, 'VF 9': 43, 'all': 45, 'unknown': 93}
```

**Phát hiện:** 93 chunks `unknown` model (~27% data) và chỉ 3 chunks `installment` — AI tự flag 2 điểm này là yếu mà không cần hỏi. Nhờ inspect sớm mà thiết kế tool đúng từ đầu, không phải sửa lại sau.

---

### Session 3 — Debug Qdrant API version cũ

**Lỗi:**
```
AttributeError: 'CollectionInfo' object has no attribute 'vectors_count'
```

**Fix:** Dùng `print(col)` để xem raw object thay vì access field cụ thể — Qdrant client mới đổi tên field.

---

### Session 4 — Debug `Must` import không tồn tại

**Lỗi:**
```
Cannot find reference 'Must' in '__init__.py'
```

**Root cause:** `qdrant_client` phiên bản mới không export `Must` class — `Filter(must=[...])` nhận list điều kiện trực tiếp, không cần wrap thêm.

**Fix:**
```python
# Sai
from qdrant_client.models import Filter as QFilter, Must

# Đúng
from qdrant_client.models import Filter, FieldCondition, MatchAny, Filter as QFilter
return QFilter(must=conditions)  # Must không cần import
```

---

### Session 5 — Debug `.search()` deprecated

**Lỗi:** Method `.search()` bị deprecated trong Qdrant client mới, không trả về kết quả đúng.

**Fix — 3 điểm thay đổi đồng thời:**
```python
# Trước
hits = client.search(
    collection_name=COLLECTION,
    query_vector=vector,
    query_filter=qdrant_filter,
    limit=top_k,
    with_payload=True,
)

# Sau
hits = client.query_points(
    collection_name=COLLECTION,
    query=vector,               # query_vector= → query=
    query_filter=qdrant_filter,
    limit=top_k,
    with_payload=True,
).points                        # unwrap ScoredPoint list
```

**Nhận xét:** AI giải thích rõ cả 3 điểm cần sửa đồng thời — không phải patch từng lỗi một.

---

### Session 6 — Migration LangChain 0.x → LangGraph 1.x

**Lỗi:**
```
ImportError: cannot import name 'create_tool_calling_agent' from 'langchain.agents'
```

**Root cause:** `AgentExecutor` và `create_tool_calling_agent` bị xóa khỏi LangChain 1.x — API thay thế chính thức là `create_react_agent` từ `langgraph.prebuilt`.

**Thay đổi chính:**
```python
# Trước (LangChain 0.x)
from langchain.agents import AgentExecutor, create_tool_calling_agent
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, ...)
result = executor.invoke({"input": question})
answer = result["output"]

# Sau (LangGraph 1.x)
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, tools, prompt=system_prompt)
result = agent.invoke({"messages": [("human", question)]})
answer = result["messages"][-1].content
```

**Nhận xét:** AI ban đầu hướng dẫn sửa import nhỏ lẻ — chỉ sau khi hỏi thẳng *"đây có phải breaking change không"* mới nhận ra cần refactor toàn bộ sang LangGraph.

---

### Session 7 — Migration LLM provider Anthropic → OpenAI

**Prompt:**
> "dùng openai key"

**Thay đổi:** `ChatAnthropic` → `ChatOpenAI(model="gpt-4o")`, cập nhật `requirements.txt`.

**Nhận xét:** AI gen mặc định theo Anthropic vì không được cung cấp context provider từ đầu — bài học: nói rõ LLM provider ngay từ prompt đầu tiên.

---

## 🔬 Research Notes

### Tại sao dùng `bkai-foundation-models/vietnamese-bi-encoder`?

Tiếng Việt có đặc thù từ ghép và dấu thanh ảnh hưởng nghĩa — multilingual model (e.g. `multilingual-e5`) thường underfit. `vietnamese-bi-encoder` được train trên corpus tiếng Việt → recall tốt hơn với query tự nhiên. Vector size 768 cân bằng giữa chất lượng và tốc độ search.

### Tại sao tách SQLite3 và Qdrant?

| Loại câu hỏi | Storage phù hợp |
|---|---|
| "VF 8 quãng đường bao nhiêu km?" | Qdrant — semantic search |
| "Lãi suất VPBank tháng này là bao nhiêu?" | SQLite3 — SQL query chính xác |
| "Chủ xe VF 6 nói gì về pin?" | Qdrant — forum_qa similarity |
| "Phí đăng ký xe ở Hà Nội?" | SQLite3 — lookup theo tỉnh |

Qdrant giỏi tìm kiếm ngữ nghĩa nhưng không phù hợp để trả lời câu hỏi cần số liệu chính xác. Hai loại storage phục vụ hai loại intent khác nhau — cần cả hai.

### Vấn đề 93 chunks `unknown` model

**Nguyên nhân:** `extract_model_from_url()` chỉ check URL chứa `vf3`...`vf9` — forum threads có URL dạng `/dien-dan/chu-de/hoi-nhanh-dap-gon/ten-bai/` không chứa tên model.

**Hướng fix đề xuất:**
```python
def classify_model_from_text(text: str) -> str:
    for m in ["VF 3", "VF 5", "VF 6", "VF 7", "VF 8", "VF 9"]:
        if m.lower().replace(" ", "") in text.lower().replace(" ", ""):
            return m
    return "all"  # Áp dụng chung nếu không xác định được
```

### Dependency pinning — bài học từ breaking changes

| Library | Breaking change gặp phải |
|---|---|
| `qdrant-client` | `vectors_count` field đổi tên; `Must` class bị xóa; `.search()` deprecated → `.query_points()` |
| `langchain` 1.x | `AgentExecutor`, `create_tool_calling_agent` bị xóa → dùng `langgraph.prebuilt` |

→ Pin version trong `requirements.txt` ngay từ đầu, không cài `latest`.