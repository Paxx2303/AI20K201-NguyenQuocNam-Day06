# 📝 Individual Reflection — Day 06
**Họ tên:** Nguyễn Văn A | **Student ID:** AI20K001

---

## 1. Role
Data Engineer + RAG Tooling — phụ trách crawl dữ liệu, thiết kế database, xây dựng vector search tools và tích hợp agent cho hệ thống VinFast RAG.

---

## 2. Đóng góp cụ thể

- Thiết kế `community_crawler.py` thu thập dữ liệu từ vinfast.vn qua 2 nguồn: WordPress REST API (tin tức, FAQ chính thức) và scrape forum threads (hỏi đáp thực tế từ chủ xe)
- Xây dựng `pipeline.py` end-to-end: Crawl → Chunk → Embed → Upsert Qdrant, kết quả 339 chunks index thành công
- **Thiết kế và xây dựng `vinfastauto_crawler.py` với structured database SQLite3:**
  - Crawl dữ liệu thực tế từ vinfastauto.com và shop.vinfastauto.com bằng Playwright (headless Chromium), xử lý cả trang SPA có JS render
  - Thiết kế schema 4 bảng quan hệ:
    - `Vehicle_Price` — giá niêm yết từng trim (Eco/Plus/Base/Standard), hỗ trợ cập nhật theo ngày hiệu lực
    - `Vehicle_Details` — thông số kỹ thuật: kích thước, động cơ, pin, tầm vận hành, tính năng (ADAS, CarPlay, OTA, camera 360...) lưu dạng JSON trong `detailed_specs`
    - `Location_Tax_Fee` — phí đăng ký, biển số, đăng kiểm, phí đường bộ, bảo hiểm theo 6 tỉnh/thành
    - `Bank_Loan_Policy` — chính sách vay 7 ngân hàng: lãi suất ưu đãi/thường, kỳ hạn, tỷ lệ vay tối đa
  - Viết parser regex `_parse_variant_prices()` và `_parse_detailed_specs()` trích xuất thông số từ text crawl được, có fallback khi không parse được
  - Seed fallback data (giá niêm yết tháng 4/2026, phí theo tỉnh, chính sách ngân hàng) để hệ thống hoạt động ngay cả khi crawl thất bại
  - Cấu hình WAL mode + foreign keys cho SQLite3, dùng `ON CONFLICT DO UPDATE` để upsert an toàn
- Viết `qdrant_tools.py` gồm 3 LangChain tools cho agent:
  - `vinfast_semantic_search` — semantic search toàn bộ collection, không filter
  - `vinfast_search_by_doctype` — filter theo `doc_type` (spec / faq / forum_qa / installment)
  - `vinfast_search_by_model` — filter theo dòng xe (VF 3–9), tự động include `model='all'`
  - Dùng `sentence-transformers` với model `bkai-foundation-models/vietnamese-bi-encoder` để embed query
  - Thiết kế `_build_filter()` tái sử dụng cho cả 3 tools, hỗ trợ kết hợp `doc_type` + `model`
  - Singleton pattern cho `QdrantClient` và `SentenceTransformer` để tránh load lại mỗi lần gọi tool
- Viết `test_agent.py` với 2 chế độ: quick test 7 câu mẫu và chat tự do
- **Migration stack agent từ LangChain 0.x → LangGraph 1.x:** refactor toàn bộ `build_agent()` từ `AgentExecutor` + `create_tool_calling_agent` sang `create_react_agent` (langgraph.prebuilt), cập nhật invoke format `{"messages": [...]}` và viết lại `parse_result()` để đọc output từ message list thay vì `intermediate_steps`
- **Migration LLM provider từ Anthropic → OpenAI:** đổi `ChatAnthropic` → `ChatOpenAI(model="gpt-4o")`, cấu hình `OPENAI_API_KEY`

---

## 3. SPEC — Mạnh / Yếu

**Mạnh nhất — Failure Modes**
Nhóm nghĩ ra được case "triệu chứng chung chung" mà AI gợi ý quá rộng, và có mitigation cụ thể: hỏi thêm câu follow-up để thu hẹp.

**Yếu nhất — ROI**
3 kịch bản thực ra chỉ khác số user, assumption gần giống nhau. Nên tách assumption rõ hơn — ví dụ: conservative = chỉ dùng ở 1 chi nhánh, optimistic = rollout toàn hệ thống.

---

## 4. Đóng góp khác

- Inspect schema Qdrant thực tế trước khi viết tools — phát hiện 93/339 chunks có `model='unknown'` và chỉ 3 chunks `installment`, flag sớm cho nhóm
- Debug bug `Must` import không tồn tại trong `qdrant_client` version mới, fix và giải thích root cause cho cả nhóm
- **Debug và fix breaking change Qdrant client:** method `.search()` bị deprecated trong phiên bản mới, tìm ra cách thay thế bằng `.query_points()` với 3 điểm thay đổi đồng thời (`query_vector=` → `query=`, thêm `.points` để unwrap kết quả), áp dụng nhất quán cho cả 3 tools
- **Debug breaking change LangChain 1.x:** xác định `AgentExecutor` và `create_tool_calling_agent` đã bị xóa khỏi `langchain.agents`, tìm ra `create_react_agent` từ `langgraph.prebuilt` là API thay thế chính thức
- Phát hiện shop.vinfastauto.com render bằng JS — thêm `wait_for_timeout(5000)` và block resource (CSS/ảnh/font) để tăng tốc crawl, tránh timeout
- Viết Docker Compose + README để người khác có thể reproduce toàn bộ hệ thống

---

## 5. Điều học được

Trước khi làm nghĩ rằng cứ crawl được data là RAG sẽ hoạt động tốt.

Sau khi build mới hiểu: **chất lượng metadata quan trọng hơn embedding model.** 93 chunks bị `unknown` model vì URL forum không chứa tên xe — filter theo model sẽ bỏ sót gần 30% data. Embedding tốt đến đâu cũng không bù được metadata sai.

Khi thiết kế SQLite3 schema mới thấy rõ: **tách structured data (giá, thông số, phí, ngân hàng) ra khỏi vector store là quyết định đúng.** Qdrant giỏi tìm kiếm ngữ nghĩa nhưng không phù hợp để query chính xác kiểu "lãi suất VPBank là bao nhiêu" — loại câu hỏi này cần SQL với số liệu có kiểm soát, không phải embedding similarity.

Ngoài ra, khi viết `qdrant_tools.py` mới thấy rõ: **thiết kế tool interface quan trọng không kém logic bên trong.** Description của từng tool và từng field trong `args_schema` ảnh hưởng trực tiếp đến việc LLM có chọn đúng tool không.

Ngoài ra học được bài học về **dependency management:** cùng một đoạn code có thể chạy hoàn toàn khác nhau tùy version library. LangChain 0.x và 1.x, Qdrant client cũ và mới đều có breaking changes không backward compatible.

> Data pipeline không chỉ là crawl — schema design, metadata tagging, và việc chọn đúng storage (vector vs relational) cho từng loại câu hỏi mới là phần quyết định chất lượng hệ thống.

---

## 6. Nếu làm lại

- Sẽ inspect data ngay sau khi crawl xong, trước khi viết bất kỳ dòng tool nào. Lần này nhờ AI nhắc mới làm — nếu tự nhớ từ đầu sẽ tiết kiệm được 1-2 vòng sửa tool do sai field name hoặc sai filter logic.
- **Sẽ thiết kế schema SQLite3 song song với Qdrant collection ngay từ đầu** — thay vì làm vector store trước rồi mới nghĩ đến structured data sau. Hai loại storage phục vụ hai loại câu hỏi khác nhau, cần plan cùng lúc.
- **Sẽ viết tool description kỹ hơn ngay từ đầu** — đặc biệt phần `args_schema` và docstring, vì đây là thứ LLM đọc để quyết định gọi tool nào, không phải code bên trong.
- **Sẽ pin version toàn bộ dependencies ngay từ đầu** (`langchain==x.x`, `qdrant-client==x.x`) thay vì cài latest rồi xử lý breaking change sau.
- **Sẽ cung cấp context môi trường cho AI ngay từ prompt đầu tiên:** LLM provider nào, version library nào, OS nào — thay vì để AI đoán rồi phải sửa lại.

---

## 7. AI giúp gì / AI sai gì

**Giúp:**
- Claude nhắc inspect data thực tế trước khi gen tools — tư duy đúng, tránh viết tool dựa trên schema tưởng tượng
- Claude phát hiện 93 unknown chunks từ output và tự flag là điểm yếu cần cải thiện, không cần hỏi
- Brainstorm được hướng fix: dùng LLM classify model từ nội dung thay vì chỉ dựa URL
- Khi gen `qdrant_tools.py`, Claude tự thiết kế `_build_filter()` dùng chung cho cả 3 tools và thêm singleton pattern — không cần nhắc
- Khi gen SQLite3 schema, Claude tự đề xuất tách thành 4 bảng độc lập thay vì nhét tất cả vào 1 bảng, và thêm seed data fallback để hệ thống không bị broken khi crawl thất bại
- Khi gặp breaking change Qdrant, Claude giải thích rõ cả 3 điểm thay đổi cần sửa đồng thời
- Khi gặp breaking change LangChain 1.x, Claude không chỉ sửa import mà refactor lại toàn bộ flow invoke/output parsing

**Sai / Mislead:**
- Claude gen sẵn `agent_example.py` dùng `ChatAnthropic` trong khi project dùng OpenAI key — phải sửa lại
- Claude ban đầu hướng dẫn sửa import LangChain thay vì nhận ra ngay đây là breaking change cần đổi hoàn toàn sang LangGraph — mất thêm 1 vòng

> **Bài học:** Cung cấp context môi trường (key, framework, version) ngay từ prompt đầu — AI không tự đoán được. Khi AI đưa ra fix nhỏ mà vẫn còn lỗi, hãy hỏi thẳng *"đây có phải breaking change không"* thay vì chỉ hỏi cách sửa từng lỗi một.