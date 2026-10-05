# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đào Ngọc Bình Thiên  **MSSV:** 2A202602814  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    207.9
graph       196     34619     5704   0.00574    280.5

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       70   0.00010     3.74
graph       0.94   1.83     5754      158   0.00064     5.59
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00574 | +$0.00574 |
| Indexing giây | 207.9 | 280.5 | ×1.35 |
| Mỗi câu: USD | $0.00010 | $0.00064 | ×6.40 |
| Mỗi câu: giây | 3.74 | 5.59 | ×1.49 |
| Mỗi câu: in_tok | 696 | 5754 | ×8.27 |

**Chi phí tăng thêm đến từ đâu?**
> - **Ở giai đoạn Indexing (dựng hệ thống):** Chi phí tăng thêm xuất phát từ 20 lượt gọi LLM phân tích ngữ nghĩa và trích xuất thực thể JSON từ 20 bài báo tin tức (tiêu tốn 34.619 input tokens và 5.704 output tokens, tốn $0.00574 và thêm 72.6 giây để nạp 204 nodes và 383 quan hệ vào Neo4j).
> - **Ở giai đoạn Querying (mỗi câu hỏi):** Chi phí tăng gấp 6.4 lần về USD và 8.27 lần về input tokens do ngoài 3 chunks văn bản thu được từ vector search, GraphRAG phải nhồi thêm toàn bộ danh sách các dữ kiện có cấu trúc (facts) mở rộng từ đồ thị tri thức (khoản luật, tội danh, tiền án, chất liên quan) vào `GRAPH_PROMPT` gửi tới LLM.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một Điều luật (Điều 2 Luật PCMT) nên vector search đơn thuần đã truy xuất đầy đủ. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Danh sách bị cáo tử hình nằm trọn vẹn trong một bài báo về vụ án 36kg ma túy tại TP.HCM. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ lấy được tin tức nên không biết Điều 251 BLHS và khung cơ bản; GraphRAG đi qua node cầu nối `Crime` để lấy trọn vẹn căn cứ pháp lý từ KB Luật. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | **Graph** | Flat RAG thiếu căn cứ pháp lý; GraphRAG tìm ra Điều 255 BLHS qua liên kết thực thể `aliases` ("Hoàng Nato") và tội danh tổ chức sử dụng. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | GraphRAG kết hợp đồng thời tội danh và node `Substance` (MDMA > 9,6kg) để suy luận chính xác điểm b khoản 4 Điều 250 với mức án tù 20 năm, chung thân hoặc tử hình. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | **Graph** | Flat RAG chỉ trích dẫn chung chung "vụ việc 1, 2, 3" thiếu tên riêng; GraphRAG truy vấn ngược từ node `Substance` tổng hợp được đầy đủ tên các vụ án/bị cáo (Huy, Thành, Viện Pháp y tâm thần). |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Missing Legal Context for Maximum Penalty)

- **Hiện tượng:** Ở câu hỏi **Q4** (*"Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"*), GraphRAG xác định đúng tội danh và Điều 255 BLHS nhưng chỉ cung cấp khung hình phạt của Khoản 1 (tù từ 02 năm đến 07 năm), bỏ sót khung phạt tối đa (tù 20 năm hoặc tù chung thân ở Khoản 4).
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời của GraphRAG ở Q4:
    > *"Theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy (khoản 1), hành vi này có khung hình phạt tù từ 02 năm đến 07 năm (ngữ cảnh không đề cập các khoản nặng hơn của Điều 255)."*
  - Truy vấn Cypher kiểm tra trực tiếp trong Neo4j:
    ```cypher
    MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS number, cl.penalty AS penalty
    ORDER BY cl.number;
    ```
  - Kết quả trả về từ đồ thị:
    ```
    number | penalty
    1      | "phạt tù từ 02 năm đến 07 năm"
    2      | "phạt tù từ 07 năm đến 15 năm"
    3      | "phạt tù từ 15 năm đến 20 năm"
    4      | "phạt tù 20 năm hoặc tù chung thân"
    ```
- **Nguyên nhân:** Nằm ở logic truy vấn Cypher trong hàm `Neo4jGraph.context`. Khi đi từ `Case` sang `Article`, thuật toán chỉ giữ lại: (1) Khoản 1 (khung cơ bản) và (2) Các khoản có quan hệ `MENTIONS` tới `Substance` mà vụ án liên quan. Vụ án bắt giữ Hoàng Nato trong tin tức là vụ triệt phá tụ điểm chưa có kết luận giám định cụ thể khối lượng để liên kết tới khoản tăng nặng, và thuật toán không có nhánh xử lý riêng cho câu hỏi hỏi về *"mức tối đa"* để lấy thêm Khoản cuối cùng (`ORDER BY cl.number DESC LIMIT 1`).
- **Đề xuất sửa:**
  - Trong `src/graph.py`, phân tích câu hỏi bằng regex: nếu phát hiện các từ khóa mang tính cực trị như `"tối đa"`, `"nặng nhất"`, `"cao nhất"`, Cypher sẽ truy vấn thêm khoản có số thứ tự lớn nhất của Điều luật:
    ```cypher
    OPTIONAL MATCH (a)-[:HAS_CLAUSE]->(max_cl:Clause)
    WITH a, max_cl ORDER BY max_cl.number DESC
    WITH a, head(collect(max_cl)) AS highest_clause
    ```
  - **Đánh đổi:** Tăng thêm 1 dữ kiện vào context, tốn thêm khoảng 30–50 input tokens cho các câu hỏi cực trị, nhưng đảm bảo độ phủ 100% cho các câu hỏi về khung hình phạt cao nhất.

---

### Lỗi E4: Phép đo sai lệch giữa Keyword Recall và LLM Judge (Metric Discrepancy)

- **Hiện tượng:** Ở câu hỏi **Q6** (*"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*), hệ thống Flat RAG nhận điểm **LLM Judge = 2/2** (đánh giá câu trả lời đúng và đủ ý), nhưng điểm **Keyword Recall lại bằng 0.00 (0%)**!
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời của Flat RAG ở Q6:
    > *"Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:*
    > *- Vụ việc [1]: Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh, kết quả giám định xác định toàn bộ là MDMA (khối lượng gần 4,3kg).*
    > *- Vụ việc [2]: Thành bị bắt quả tang khi mang 5 viên ma túy đi bán, kết luận giám định xác định số viên nén này là ma túy MDMA.*
    > *- Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA (tổng khối lượng hơn 5,3kg)."*
  - Danh sách từ khóa bắt buộc trong `benchmark_kg.json`:
    ```json
    "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
    ```
- **Nguyên nhân:** Nằm ở sự hạn chế của phương pháp đo lường `keyword_recall`:
  - Hàm `keyword_recall` kiểm tra khớp chuỗi con thô (`k.lower() in answer.lower()`).
  - Flat RAG trích xuất đúng 3 vụ án từ 3 đoạn văn bản nhưng gọi tên theo số thứ tự trích dẫn `Vụ việc [1]`, `Vụ việc [2]` và chỉ gọi bị cáo là `"Thành"` (thiếu họ `"Lê Minh"`). Vụ việc tại Viện Pháp y tâm thần không được nhắc đến do đoạn văn bản top-k không chứa từ khóa này.
  - LLM-as-judge hiểu được ngữ nghĩa rằng câu trả lời đã chỉ ra được các sự kiện thực tế có thật về MDMA trong ngữ cảnh nên đã hào phóng cho điểm tối đa 2/2, tạo ra sự mâu thuẫn trực tiếp giữa điểm định lượng máy móc (Recall = 0.00) và điểm định tính ngữ nghĩa (Judge = 2).
- **Đề xuất sửa:**
  - Điều chỉnh prompt yêu cầu mô hình khi tổng hợp danh sách vụ việc bắt buộc phải nêu rõ **Họ tên đầy đủ của người cầm đầu/bị cáo** hoặc **Tên tổ chức/địa danh xảy ra vụ việc**.
  - Trong benchmark, thay thế phép đo `keyword_recall` bằng kiểm tra thực thể thực tế (Named Entity Matching) hoặc dùng bộ từ khóa mềm (cho phép match các biến thể như `Thành` hoặc `Lê Minh Thành`).

---

## 4. Kết luận (5 điểm)

Từ số liệu thực nghiệm thu được:

1. **Khi nào nên dùng Knowledge Graph (GraphRAG):**
   - **Khi câu hỏi mang tính liên kết đa tài liệu (Cross-KB):** Ở các câu Q3 và Q5, Flat RAG hoàn toàn bất lực trong việc tìm ra Điều luật và khung hình phạt (Recall chỉ đạt 0.33 và 0.40, điểm Judge dừng ở mức 1). GraphRAG với node cầu nối `Crime` và `Substance` đã xuất sắc đạt **Recall 1.00 và Judge 2.00**, trả lời trọn vẹn cả tội danh, điều khoản và khung hình phạt từ tử hình đến tù có thời hạn.
   - **Khi câu hỏi mang tính tổng hợp toàn cục (Global Aggregation):** Ở câu Q6, GraphRAG vượt trội nhờ khả năng duyệt ngược từ node thực thể (`Substance {name: 'MDMA'}`) đến tất cả các vụ án phân tán trong toàn bộ corpus, đạt Recall 1.00 so với 0.00 của Flat RAG.
2. **Khi nào Flat RAG là đủ:**
   - **Khi thông tin mang tính cục bộ (Single-hop):** Ở câu Q1 và Q2, toàn bộ câu trả lời nằm trọn vẹn trong một văn bản (1 Điều luật hoặc 1 bài báo). Cả Flat RAG và GraphRAG đều đạt điểm tuyệt đối (**Recall 1.00, Judge 2.00**).
   - Trong tình huống này, việc dùng Flat RAG mang lại lợi thế vượt trội về hiệu năng kinh tế: **tiết kiệm 84.4% chi phí API** ($0.00010 so với $0.00064 mỗi câu), **giảm 87.9% token đầu vào** (696 so với 5.754 tokens) và **tốc độ phản hồi nhanh hơn 33%** (3.74s so với 5.59s).
3. **Quy luật tổng kết:** Nếu dữ liệu có tính cô lập cao và câu hỏi mang tính tìm kiếm đoạn trích cục bộ, Flat RAG là giải pháp tối ưu chi phí. Khi dữ liệu có tính quan hệ mạng lưới phức tạp và câu hỏi đòi hỏi suy luận xuyên văn bản (cross-boundary reasoning), Knowledge Graph là khoản đầu tư hoàn toàn xứng đáng.

---

## 5. Tự kiểm (5 điểm)

Log chạy kiểm tra tự động:

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.08s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

- **Ảnh Neo4j đã chuẩn bị:** `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
- **Người đã chọn cho `kg_my_case.png`:** `Cái Quang Huy` (Bị cáo trong vụ án vận chuyển ma túy xuyên quốc gia qua sân bay Nội Bài, truy tố theo Điều 250 BLHS).

---

## Vấn đề gặp phải (không tính điểm)

- **Vấn đề 1 (Giới hạn tần suất Free-tier API):** Khi nạp đồng thời 20 bài báo và đánh giá liên tiếp 6 câu hỏi, Google Gemini API trả về mã lỗi `HTTP 429 (Resource Exhausted)`. Đã xử lý triệt để bằng cách tích hợp thuật toán retry backoff với thời gian chờ trích xuất trực tiếp từ header `retryDelay` trong `src/llm.py`.
- **Vấn đề 2 (Mã hóa ký tự Unicode trên Windows PowerShell):** Mặc định PowerShell Windows dùng bảng mã cp1252 gây lỗi `UnicodeEncodeError`. Đã xử lý bằng cách thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"` trước khi thực thi script.
