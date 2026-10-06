# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Nam Anh  **MSSV:** 2A202602901  **Ngày:** 2026-10-06

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu khớp 100% với file `ket_qua_benchmark_kg.txt` sinh ra từ code. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    114.1
graph       196     34619     6063   0.00589    193.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       85   0.00010     2.15
graph       0.89   2.00     5159      125   0.00057     3.16
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00589 | +0.00589 USD |
| Indexing giây | 114.1 | 193.3 | 1.69× |
| Mỗi câu: USD | 0.00010 | 0.00057 | 5.70× |
| Mỗi câu: giây | 2.15 | 3.16 | 1.47× |
| Mỗi câu: in_tok | 696 | 5159 | 7.41× |

**Chi phí tăng thêm đến từ đâu?**
> - **Giai đoạn Indexing:** Flat RAG chỉ chạy embedding cho 176 chunks nên không phát sinh chi phí LLM chat. Ngược lại, GraphRAG phải gọi thêm 20 lượt LLM để trích xuất cấu trúc thực thể từ 20 bài báo tin tức (tốn 34,619 input tokens và 6,063 output tokens), kéo dài thời gian dựng thêm ~79.2 giây và tốn 0.00589 USD.
> - **Giai đoạn Querying:** Flat RAG chỉ đưa top-k chunk vector vào ngữ cảnh (trung bình 696 input tokens). Trong khi đó, GraphRAG nối thêm danh sách facts mở rộng từ Neo4j vào prompt, đẩy input prompt trung bình lên 5,159 tokens (gấp 7.41 lần), khiến chi phí mỗi câu truy vấn tăng 5.7 lần (từ $0.00010 lên $0.00057) và độ trễ tăng từ 2.15s lên 3.16s (+47%).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một điều luật của Luật PCMT 2021 nên cả hai pipeline đều tìm trúng đoạn văn và trả lời chuẩn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Danh sách bị cáo nhận án tử hình nằm trong cùng bài báo về vụ 36kg ma túy, vector search lấy đủ chunk cần thiết. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG bị đứt liên kết nên không biết số Điều luật và khung hình phạt cơ bản, còn GraphRAG đi trọn vẹn từ Thành $\rightarrow$ Vụ án $\rightarrow$ Tội danh $\rightarrow$ Điều 251 $\rightarrow$ Khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ tìm được hành vi của Hoàng Nato mà không tra được khung phạt tối đa trong luật; GraphRAG lần ra Điều 255 và lấy đúng khung tù chung thân ở khoản 4. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG thiếu hẳn khung hình phạt của luật, trong khi GraphRAG đối chiếu được tang vật 9,6kg MDMA với ngưỡng định khung $\ge$ 100g ở khoản 4 Điều 250 (khung tử hình). |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 2 | **Graph** | Flat RAG bị nghẽn ở top-k chunks nên bỏ sót nhiều bài báo; GraphRAG gom đủ 5 vụ việc liên quan đến MDMA qua node Substance và được LLM judge chấm điểm tối đa 2/2. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Case không có quan hệ CHARGED_WITH sang Crime)

- **Hiện tượng:** Có 6 vụ án (`Case`) trong đồ thị hoàn toàn bị cô lập, không có cạnh `CHARGED_WITH` nối sang bất kỳ tội danh (`Crime`) nào, khiến đường truy vấn từ vụ án sang văn bản luật bị đứt gãy.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các `Case` mồ côi tội danh:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() 
RETURN k.name AS name, k.doc_id AS doc_id
```

```
Số vụ không nối được sang Crime: 6
1. Vụ vận chuyển hơn 800kg chất nghi là ma túy tại Preah Sihanouk | doc_id: news-100260924145818945
2. Vụ phát hiện bao tải chứa 20kg nghi ma túy dạt vào bờ biển Phú Quốc | doc_id: news-100260927182621527
3. Vụ phát hiện kiện hàng nghi chứa 20kg ma túy dạt vào bờ biển Phú Quốc ngày 25-9 | doc_id: news-100260927182621527
4. Vụ liên quan 126 người bị bắt vì ma túy tại TP.HCM | doc_id: news-100260927182621527
5. Triệt phá chuyên án A3-626P | doc_id: news-100261002184934505
6. Vụ vận chuyển ma túy qua sân bay Nội Bài | doc_id: news-100260918080821054
```

- **Nguyên nhân:** Đọc lại các bài báo gốc thì thấy đây là các bài đưa tin nhanh về giai đoạn phát hiện tang vật trôi dạt bờ biển hoặc vụ việc xảy ra ở nước ngoài. Lúc này cơ quan công an mới chỉ thu giữ tang vật chứ chưa có quyết định khởi tố vụ án, khởi tố bị can hay định danh tội danh cụ thể. Do bài báo không có từ khóa tội danh chính thức trong Bộ luật Hình sự, prompt trích xuất và hàm `link_entity` không thể ép vào tội danh nào nên trường `charges` bị rỗng.
- **Đề xuất sửa:** Cho phép hệ thống suy đoán một tội danh dự kiến (provisional charge) dựa trên hành vi khách quan (ví dụ: phát hiện tang vật $\rightarrow$ liên kết tạm với tội tàng trữ hoặc vận chuyển với nhãn cảnh báo), hoặc bổ sung đường nối trực tiếp giữa `Substance` của vụ án với `Substance` trong `Clause` mà không nhất thiết phải bắt buộc đi qua node trung gian `Crime`.

---

### Lỗi E3: Trùng thực thể (Thực thể Case và Substance bị phân mảnh thành nhiều node)

- **Hiện tượng:** Cùng một vụ án ngoài đời thực hoặc cùng một chất ma túy nhưng lại bị nhân bản thành nhiều node riêng biệt trong đồ thị.
- **Bằng chứng:** Truy vấn Cypher tìm các vụ án liên quan đến bị can `Dương Minh Tuấn` (Hoàng Nato):

```cypher
MATCH (p:Person {name: 'Dương Minh Tuấn'})-[:INVOLVED_IN]->(k:Case)
RETURN p.name AS person, k.name AS case_name
```

```
Person: Dương Minh Tuấn | Case: Vụ bắt giữ giang hồ Hoàng Nato và 126 người liên quan 8 đường dây ma túy tại TP.HCM
Person: Dương Minh Tuấn | Case: Vụ bắt giữ TikToker Phannhibeauty và giang hồ Hoàng Nato tại TP.HCM
Person: Dương Minh Tuấn | Case: Vụ sử dụng ma túy etomidate của Hoàng Nato và TikToker Phannhibeauty tại TP.HCM
Person: Dương Minh Tuấn | Case: Vụ triệt phá 8 đường dây ma túy liên quan đến 'Hoàng Nato' tại TP.HCM
```
Ngoài ra, danh sách `Substance` xuất hiện thêm một node tên chung chung là `"ma túy"` đứng song song với các chất cụ thể như `MDMA`, `Heroine`.

- **Nguyên nhân:** Khóa định danh của `Case` hiện đang `MERGE` hoàn toàn theo chuỗi `name` do LLM tự do đặt tên trong JSON. Mỗi bài báo phản ánh chuyên án ở một khía cạnh hoặc thời điểm khác nhau (lúc bắt giữ, lúc khởi tố, lúc điều tra mở rộng) khiến LLM đặt 4 cái tên khác nhau, dẫn tới việc Neo4j tạo ra 4 node `Case` riêng biệt. Tương tự, một số bài báo chỉ viết từ "ma túy" thay vì tên chất cụ thể, khiến LLM tạo ra node chất chung chung.
- **Đề xuất sửa:** Khóa định danh của `Case` không nên dùng tên tự do mà nên kết hợp mã chuyên án hoặc bộ khóa tổng hợp: `(primary_suspect, location, month_year)`. Với `Substance`, cần bổ sung một từ điển chuẩn hóa để lọc bỏ các danh từ chung như "ma túy", "chất gây nghiện" trước khi nạp vào đồ thị.

---

### Lỗi E4: Phép đo sai (Mâu thuẫn giữa Keyword Recall và LLM Judge ở câu Q6)

- **Hiện tượng:** Ở câu hỏi tổng hợp Q6, GraphRAG trả lời rất chi tiết, bao quát đầy đủ 5 vụ án và được LLM Judge chấm điểm tuyệt đối (2/2 điểm), nhưng chỉ số `recall` đo bằng từ khóa chỉ đạt mức 0.33.
- **Bằng chứng:** Đối chiếu câu trả lời thực tế của GraphRAG với danh sách từ khóa bắt buộc `must_include` trong `data/benchmark_kg.json`:

```json
"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```

Trích câu trả lời của GraphRAG ở câu Q6 (`ket_qua_benchmark_kg.txt`):
> *"1. Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài...*  
> *2. Vụ mua bán ma túy tổ chức sinh nhật tại Hà Nội...*  
> *3. Vụ tổ chức sử dụng và tàng trữ trái phép chất ma túy tại Sầm Sơn và Viện Pháp y tâm thần Trung ương...*  
> *4. Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương...*  
> *5. Vụ bắt giữ giang hồ Hoàng Nato và 126 người liên quan 8 đường dây ma túy tại TP.HCM"*

- **Nguyên nhân:** Hàm `keyword_recall` hoạt động theo cơ chế tìm kiếm xâu con chính xác (`k.lower() in answer.lower()`). Khi GraphRAG tổng hợp từ đồ thị, model đã hành văn tự nhiên bằng cách liệt kê theo tên vụ án (gọi là *"Vụ vận chuyển hơn 10kg ma túy từ Đức..."* thay vì nhắc tên *"Cái Quang Huy"*; gọi là *"Vụ mua bán ma túy tổ chức sinh nhật..."* thay vì nhắc tên *"Lê Minh Thành"*). Mặc dù câu trả lời chính xác và đầy đủ về mặt bản chất, phép đo từ khóa máy móc vẫn chấm rớt điểm (0.33/1.0).
- **Đề xuất sửa:** Không nên dùng keyword matching cứng nhắc cho các câu hỏi tổng hợp kiến thức. Nên đổi sang đánh giá bằng semantic similarity hoặc dùng LLM-as-a-judge làm trọng số chính. Nếu vẫn giữ keyword recall thì danh sách `must_include` cần hỗ trợ tập từ khóa thay thế (synonym sets: `["Cái Quang Huy" OR "vận chuyển ma túy từ Đức"]`).

## 4. Kết luận (5 điểm)

Từ kết quả benchmark thực nghiệm giữa hai pipeline:
1. **Khi nào Flat RAG là đủ:**
   - Khi bài toán chỉ gồm các câu hỏi đơn chặng (`single-hop`), thông tin cần tìm nằm trọn vẹn trong một đoạn văn bản (như Q1 về định nghĩa tiền chất và Q2 về mức án tử hình). Ở các câu này, Flat RAG trả lời chính xác tuyệt đối (`recall = 1.00`, `judge = 2.0`) nhưng có độ trễ nhanh hơn (2.15s so với 3.16s) và chi phí rẻ hơn gần 6 lần so với GraphRAG ($0.00010 so với $0.00057 mỗi câu).
2. **Khi nào bắt buộc phải dùng Knowledge Graph (GraphRAG):**
   - Khi cần trả lời các câu hỏi đa chặng (`cross-kb`, `multi-hop`), thông tin nằm rải rác ở nhiều nguồn dữ liệu khác nhau (bài báo chỉ có tên người và tang vật, còn khung phạt lại nằm ở điều luật như Q3, Q4, Q5). Ở nhóm câu này, Flat RAG hoàn toàn bất lực vì vector search không thể tự nối logic giữa các văn bản rời rạc (`judge = 1.0`, `recall = 0.33–0.40`). Ngược lại, GraphRAG đạt điểm tuyệt đối (`judge = 2.0`, `recall = 1.00`) nhờ khả năng duyệt qua node cầu nối `Crime`.
   - Khi cần thực hiện các câu hỏi gom nhóm tổng hợp (`aggregation` như Q6), đồ thị tri thức giúp duyệt toàn bộ các vụ việc liên quan đến một chất mà không bị giới hạn bởi ngưỡng `top_k` của vector retrieval.
   - Chi phí dựng đồ thị lúc đầu ($0.00589 và ~3 phút) là chi phí một lần rất nhỏ so với giá trị thông tin mang lại cho các hệ thống đòi hỏi độ chính xác pháp lý cao.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.04s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00054. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.  
Người đã chọn cho `kg_my_case.png`: **Ngô Việt Dũng** (Vụ tổ chức sử dụng và tàng trữ trái phép chất ma túy tại Sầm Sơn và Viện Pháp y tâm thần Trung ương - Điều 249 BLHS).

## Vấn đề gặp phải (không tính điểm)

- Ban đầu cấu hình mặc định dùng `gemini-2.5-flash-lite` bị báo lỗi 404 do phía Google đã dừng hỗ trợ người dùng mới; tôi đã điều chỉnh sang `gemini-3.5-flash-lite` và mô hình chạy rất mượt, ổn định.
- Docker Desktop trên Linux đôi lúc bị nghẽn socket forwarding khi máy vào trạng thái sleep hoặc khởi động lại; tôi đã khắc phục bằng cách cấu hình cơ chế retry kèm exponential backoff khi gọi kết nối mạng.
