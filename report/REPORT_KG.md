# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Nam Anh  **MSSV:** 2A202602901  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu khớp 100% với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.
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
> - **Lúc Indexing:** Flat RAG chỉ tốn thời gian embedding 176 chunks qua mô hình embedding. GraphRAG phải gọi thêm 20 lượt gọi LLM để trích xuất cấu trúc thực thể/quan hệ từ 20 bài báo tin tức (tiêu tốn 34,619 input tokens và 6,063 output tokens), làm tăng thời gian xây dựng thêm 79.2 giây và phát sinh thêm 0.00589 USD.
> - **Lúc Querying:** Flat RAG chỉ đưa top-k chunk vector vào prompt (trung bình 696 input tokens). GraphRAG ghép thêm toàn bộ facts duyệt multi-hop từ đồ thị vào prompt, khiến context dài hơn gấp 7.41 lần (5,159 input tokens), dẫn đến chi phí mỗi câu hỏi tăng 5.7 lần (từ $0.00010 lên $0.00057) và độ trễ tăng nhẹ từ 2.15s lên 3.16s (+47%).
> - **Giai đoạn Indexing:** Flat RAG chỉ chạy embedding cho 176 chunks nên không phát sinh chi phí LLM chat. Ngược lại, GraphRAG phải gọi thêm 20 lượt LLM để trích xuất cấu trúc thực thể từ 20 bài báo tin tức (tốn 34,619 input tokens và 6,063 output tokens), kéo dài thời gian dựng thêm ~79.2 giây và tốn 0.00589 USD.
> - **Giai đoạn Querying:** Flat RAG chỉ đưa top-k chunk vector vào ngữ cảnh (trung bình 696 input tokens). Trong khi đó, GraphRAG nối thêm danh sách facts mở rộng từ Neo4j vào prompt, đẩy input prompt trung bình lên 5,159 tokens (gấp 7.41 lần), khiến chi phí mỗi câu truy vấn tăng 5.7 lần (từ $0.00010 lên $0.00057) và độ trễ tăng từ 2.15s lên 3.16s (+47%).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một điều luật của Luật PCMT 2021, cả hai pipeline đều tìm được đoạn văn phù hợp. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin các bị cáo bị tuyên tử hình nằm trọn trong bài báo về vụ 36kg ma túy, vector search lấy đủ ngữ cảnh. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG thiếu liên kết luật nên không trả lời được số Điều và khung phạt cơ bản; GraphRAG đi từ Thành $\rightarrow$ Vụ án $\rightarrow$ Tội danh $\rightarrow$ Điều 251 $\rightarrow$ Khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ biết hành vi của Hoàng Nato mà không tra được khung phạt tối đa; GraphRAG nối sang Điều 255 và lấy được khung chung thân ở khoản 4. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG không biết khoản luật và khung phạt; GraphRAG liên kết tang vật 9,6kg MDMA đối chiếu chính xác khoản 4 Điều 250 (khung tử hình). |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 2 | **Graph** | Flat RAG chỉ lấy được 3 đoạn nhỏ nên bỏ sót các vụ án khác; GraphRAG gom đủ 5 vụ việc liên quan đến MDMA qua node Substance và được judge chấm 2/2. |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một điều luật của Luật PCMT 2021 nên cả hai pipeline đều tìm trúng đoạn văn và trả lời chuẩn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Danh sách bị cáo nhận án tử hình nằm trong cùng bài báo về vụ 36kg ma túy, vector search lấy đủ chunk cần thiết. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG bị đứt liên kết nên không biết số Điều luật và khung hình phạt cơ bản, còn GraphRAG đi trọn vẹn từ Thành $\rightarrow$ Vụ án $\rightarrow$ Tội danh $\rightarrow$ Điều 251 $\rightarrow$ Khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ tìm được hành vi của Hoàng Nato mà không tra được khung phạt tối đa trong luật; GraphRAG lần ra Điều 255 và lấy đúng khung tù chung thân ở khoản 4. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG thiếu hẳn khung hình phạt của luật, trong khi GraphRAG đối chiếu được tang vật 9,6kg MDMA với ngưỡng định khung $\ge$ 100g ở khoản 4 Điều 250 (khung tử hình). |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 2 | **Graph** | Flat RAG bị nghẽn ở top-k chunks nên bỏ sót nhiều bài báo; GraphRAG gom đủ 5 vụ việc liên quan đến MDMA qua node Substance và được LLM judge chấm điểm tối đa 2/2. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Case không có quan hệ CHARGED_WITH sang Crime)

- **Hiện tượng:** Có 6 vụ án (`Case`) trong đồ thị không có quan hệ `[:CHARGED_WITH]` nối sang bất kỳ tội danh (`Crime`) nào, khiến đường đi sang KB Luật bị đứt hoàn toàn.
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

- **Nguyên nhân:** Khi đọc bài báo gốc theo `doc_id`, đây là các bài báo đưa tin về giai đoạn phát hiện tang vật trôi dạt bờ biển hoặc chuyên án mới bắt giữ sơ bộ. Tại thời điểm viết báo, cơ quan điều tra chưa có quyết định khởi tố bị can hoặc chưa khởi tố tội danh cụ thể theo Bộ luật Hình sự. Do đó LLM không tìm thấy tội danh hợp lệ trong danh mục chuẩn để gán vào trường `charges`.
- **Đề xuất sửa:** Bổ sung cơ chế suy luận tội danh dự kiến (inferred/provisional crime) dựa trên từ khóa hành vi trong bài báo (ví dụ: phát hiện tang vật $\rightarrow$ tạm nối với tội tàng trữ hoặc vận chuyển với trọng số thấp), hoặc tạo thêm cầu nối phụ trực tiếp từ `Substance` của `Case` sang `Substance` của `Clause` mà không cần đi qua `Crime`.
- **Nguyên nhân:** Đọc lại các bài báo gốc thì thấy đây là các bài đưa tin nhanh về giai đoạn phát hiện tang vật trôi dạt bờ biển hoặc vụ việc xảy ra ở nước ngoài. Lúc này cơ quan công an mới chỉ thu giữ tang vật chứ chưa có quyết định khởi tố vụ án, khởi tố bị can hay định danh tội danh cụ thể. Do bài báo không có từ khóa tội danh chính thức trong Bộ luật Hình sự, prompt trích xuất và hàm `link_entity` không thể ép vào tội danh nào nên trường `charges` bị rỗng.
- **Đề xuất sửa:** Cho phép hệ thống suy đoán một tội danh dự kiến (provisional charge) dựa trên hành vi khách quan (ví dụ: phát hiện tang vật $\rightarrow$ liên kết tạm với tội tàng trữ hoặc vận chuyển với nhãn cảnh báo), hoặc bổ sung đường nối trực tiếp giữa `Substance` của vụ án với `Substance` trong `Clause` mà không nhất thiết phải bắt buộc đi qua node trung gian `Crime`.

---

### Lỗi E3: Trùng thực thể (Thực thể Case và Substance bị phân mảnh nhiều node)
### Lỗi E3: Trùng thực thể (Thực thể Case và Substance bị phân mảnh thành nhiều node)

- **Hiện tượng:** Cùng một vụ án ngoài đời thực hoặc cùng một chất ma túy nhưng bị tạo thành nhiều node riêng rẽ trong đồ thị thay vì hợp nhất.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các `Case` liên quan đến cùng một đối tượng (`Dương Minh Tuấn` - Hoàng Nato):
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
Ngoài ra, danh sách `Substance` có thêm node chung chung là `"ma túy"` đứng cạnh các node chất cụ thể như `Heroine`, `MDMA`.
Ngoài ra, danh sách `Substance` xuất hiện thêm một node tên chung chung là `"ma túy"` đứng song song với các chất cụ thể như `MDMA`, `Heroine`.

- **Nguyên nhân:** Node `Case` hiện tại đặt khóa định danh `MERGE` theo thuộc tính `name` do LLM tự sinh ra trong JSON. Khi nhiều bài báo viết về cùng một chuyên án lớn ở các thời điểm hoặc góc độ khác nhau, LLM đặt tên vụ án khác nhau trong mỗi bài báo, khiến câu lệnh `MERGE (k:Case {name: $name})` tạo ra 4 node vụ án khác nhau thay vì gộp lại.
- **Đề xuất sửa:** Thay đổi khóa định danh của `Case` không dựa thuần túy vào tên tự do, mà sử dụng mã chuyên án (nếu có) hoặc bộ khóa kết hợp: `(primary_suspect, location, month_year)`. Thêm bước Entity Resolution (so khớp độ tương đồng vector của summary vụ án) trước khi thực hiện `MERGE`.
- **Nguyên nhân:** Khóa định danh của `Case` hiện đang `MERGE` hoàn toàn theo chuỗi `name` do LLM tự do đặt tên trong JSON. Mỗi bài báo phản ánh chuyên án ở một khía cạnh hoặc thời điểm khác nhau (lúc bắt giữ, lúc khởi tố, lúc điều tra mở rộng) khiến LLM đặt 4 cái tên khác nhau, dẫn tới việc Neo4j tạo ra 4 node `Case` riêng biệt. Tương tự, một số bài báo chỉ viết từ "ma túy" thay vì tên chất cụ thể, khiến LLM tạo ra node chất chung chung.
- **Đề xuất sửa:** Khóa định danh của `Case` không nên dùng tên tự do mà nên kết hợp mã chuyên án hoặc bộ khóa tổng hợp: `(primary_suspect, location, month_year)`. Với `Substance`, cần bổ sung một từ điển chuẩn hóa để lọc bỏ các danh từ chung như "ma túy", "chất gây nghiện" trước khi nạp vào đồ thị.

---

### Lỗi E4: Phép đo sai (Mâu thuẫn giữa Keyword Recall và LLM Judge ở câu Q6)

- **Hiện tượng:** Ở câu Q6 (câu hỏi gom nhóm aggregation), GraphRAG trả lời rất đầy đủ, chính xác và được LLM Judge chấm điểm tuyệt đối 2/2 điểm, nhưng chỉ số `recall` đo bằng từ khóa lại chỉ đạt 0.33 (thấp bất thường).
- **Bằng chứng:** So sánh câu trả lời của GraphRAG và danh sách từ khóa bắt buộc `must_include` trong `data/benchmark_kg.json`:
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

- **Nguyên nhân:** Hàm `keyword_recall` hoạt động theo cơ chế so khớp chuỗi con chính xác (`k.lower() in answer.lower()`). Khi GraphRAG tổng hợp từ đồ thị, model liệt kê danh sách tên các vụ án (ví dụ *"Vụ vận chuyển hơn 10kg ma túy từ Đức về..."* thay vì nhắc trực tiếp tên cá nhân *"Cái Quang Huy"*; *"Vụ mua bán ma túy tổ chức sinh nhật tại Hà Nội"* thay vì ghi tên *"Lê Minh Thành"*). Mặc dù câu trả lời đúng 100% về mặt nội dung, metric từ khóa cứng vẫn phạt điểm nặng (0.33/1.0).
- **Đề xuất sửa:** Không nên chỉ dùng keyword matching đơn thuần cho các câu hỏi tổng hợp dạng aggregation. Nên chuẩn hóa metric đánh giá bằng cách cho phép danh sách `must_include` chứa các cụm từ đồng nghĩa (synonym sets: `["Cái Quang Huy" OR "vận chuyển ma túy từ Đức"]`), hoặc dùng LLM Judge làm thước đo chính.
- **Nguyên nhân:** Hàm `keyword_recall` hoạt động theo cơ chế tìm kiếm xâu con chính xác (`k.lower() in answer.lower()`). Khi GraphRAG tổng hợp từ đồ thị, model đã hành văn tự nhiên bằng cách liệt kê theo tên vụ án (gọi là *"Vụ vận chuyển hơn 10kg ma túy từ Đức..."* thay vì nhắc tên *"Cái Quang Huy"*; gọi là *"Vụ mua bán ma túy tổ chức sinh nhật..."* thay vì nhắc tên *"Lê Minh Thành"*). Mặc dù câu trả lời chính xác và đầy đủ về mặt bản chất, phép đo từ khóa máy móc vẫn chấm rớt điểm (0.33/1.0).
- **Đề xuất sửa:** Không nên dùng keyword matching cứng nhắc cho các câu hỏi tổng hợp kiến thức. Nên đổi sang đánh giá bằng semantic similarity hoặc dùng LLM-as-a-judge làm trọng số chính. Nếu vẫn giữ keyword recall thì danh sách `must_include` cần hỗ trợ tập từ khóa thay thế (synonym sets: `["Cái Quang Huy" OR "vận chuyển ma túy từ Đức"]`).

## 4. Kết luận (5 điểm)

Từ số liệu thực nghiệm đo đạc giữa hai hệ thống:
Từ kết quả benchmark thực nghiệm giữa hai pipeline:
1. **Khi nào Flat RAG là đủ:**
   - Khi các câu hỏi là đơn chặng (`single-hop`), thông tin cần trả lời nằm trọn vẹn trong một đoạn văn bản cục bộ (như Q1 về định nghĩa luật và Q2 về chi tiết bài báo). Trên các câu này, Flat RAG đạt độ chính xác tối đa (`recall = 1.00`, `judge = 2.0`), độ trễ nhanh hơn (2.15s vs 3.16s) và chi phí rẻ hơn gần 6 lần so với GraphRAG ($0.00010 vs $0.00057).
2. **Khi nào nên dùng Knowledge Graph (GraphRAG):**
   - Khi bài toán đòi hỏi suy luận đa chặng (`cross-kb`, `multi-hop`), thông tin nằm rải rác ở nhiều nguồn dữ liệu độc lập (như vụ án ở tin tức và khung hình phạt ở luật tại Q3, Q4, Q5). Ở nhóm câu hỏi này, Flat RAG thất bại vì không thể tự liên kết thông tin giữa các tài liệu (`judge = 1.0`, `recall = 0.33–0.40`), trong khi GraphRAG đạt độ chính xác tuyệt đối (`judge = 2.0`, `recall = 1.00`).
   - Khi thực hiện các câu truy vấn gom nhóm (`aggregation` như Q6) quét trên toàn bộ kho dữ liệu, GraphRAG duyệt qua các node quan hệ nhanh chóng và tránh được điểm nghẽn giới hạn số lượng `top_k` chunk của vector search.
   - Chi phí dựng đồ thị ($0.00589 và ~3 phút) là khoản đầu tư một lần (one-off) rất nhỏ, hoàn toàn xứng đáng với bước nhảy vọt về chất lượng câu trả lời trong các hệ thống đòi hỏi độ chính xác cao như pháp lý, y tế.
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

- Ban đầu model `gemini-2.5-flash-lite` trong cấu hình mặc định bị nhà cung cấp Google deprecate, chuyển sang `gemini-3.5-flash-lite` hoạt động 
- Thiết kế ban đầu của Docker Desktop trên Linux đôi khi gặp hiện tượng port-forward socket bị reset kết nối tạm thời khi gọi liên tục, xử lý bằng cách cấu hình retry kết nối với exponential backoff.
- Ban đầu cấu hình mặc định dùng `gemini-2.5-flash-lite` bị báo lỗi 404 do phía Google đã dừng hỗ trợ người dùng mới; tôi đã điều chỉnh sang `gemini-3.5-flash-lite` và mô hình chạy rất mượt, ổn định.
- Docker Desktop trên Linux đôi lúc bị nghẽn socket forwarding khi máy vào trạng thái sleep hoặc khởi động lại; tôi đã khắc phục bằng cách cấu hình cơ chế retry kèm exponential backoff khi gọi kết nối mạng.
