# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Võ Đức Trí  **MSSV:** 2A202602603  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000     12.5
graph       196     34619     6128   0.00591    137.2

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       77   0.00010     6.64
graph       0.94   1.83     5044      167   0.00057     6.25
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00591 | N/A |
| Indexing giây | 12.5 | 137.2 | ×10.98 |
| Mỗi câu: USD | 0.00010 | 0.00057 | ×5.70 |
| Mỗi câu: giây | 6.64 | 6.25 | ×0.94 |
| Mỗi câu: in_tok | 696 | 5044 | ×7.25 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Ở giai đoạn Indexing, chi phí tăng thêm từ việc dùng LLM trích xuất cấu trúc thực thể/quan hệ có schema JSON từ 20 bài báo (196 calls, 34.6k in_tok, 6.1k out_tok tốn $0.00591 và 137.2s), trong khi Flat RAG chỉ tính embedding thuần túy. Ở giai đoạn Querying, GraphRAG có chi phí USD mỗi câu cao hơn ~5.7 lần ($0.00057 so với $0.00010) do đồ thị con trích xuất từ Cypher (chứa đầy đủ thông tin Person, Case, Crime, Article, Clause kèm penalty) làm `in_tok` tăng 7.25 lần (5044 tokens so với 696 tokens); tuy nhiên thời gian phản hồi là tương đương (6.25s so với 6.64s).

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều trích xuất chính xác định nghĩa tiền chất từ khoản 4 Điều 2 Luật Phòng, chống ma túy. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều tìm được đúng 2 bị cáo nhận án tử hình (Trần Thanh Tuấn, Trần Minh Tâm) từ văn bản bài báo đơn lẻ. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG đứt mạch giữa tin tức và luật hình sự, còn GraphRAG kết nối qua `CHARGED_WITH -> Crime <- DEFINES` lấy trọn vẹn Điều 251 BLHS và khung 2–7 năm. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | Flat RAG không xác định được điều luật áp dụng cho hành vi của Hoàng Nato, trong khi GraphRAG định danh chính xác Điều 255 BLHS và khung khoản 1. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | GraphRAG kết nối thành công chuỗi 4 bước (Cái Quang Huy → Vụ án → MDMA >9.6kg → điểm b khoản 4 Điều 250 BLHS → án tử hình), còn Flat RAG thiếu liên kết định lượng và điều luật. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | Graph | GraphRAG tổng hợp toàn diện các vụ án (Nội Bài, Hà Nội, Viện Pháp y...) và các điều luật liên quan đến MDMA qua duyệt đồ thị, còn Flat RAG chỉ trích dẫn mẩu tin vụn vặt không có tên vụ cụ thể. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (lọc bỏ sót khoản có mức phạt tối đa ở câu Q4)

- **Hiện tượng:** Ở câu Q4 (hỏi về mức phạt tù tối đa cho hành vi tổ chức sử dụng của giang hồ Hoàng Nato), GraphRAG chỉ trả lời được mức phạt cơ bản của khoản 1 Điều 255 BLHS (2–7 năm tù) mà không đưa ra được mức phạt tù tối đa của điều luật (20 năm hoặc tù chung thân), khiến `recall = 0.67` và `judge = 1`.
- **Bằng chứng:**
Trích nguyên văn câu trả lời Q4 từ `ket_qua_benchmark_kg.txt`:
```
--- Q4 [cross-kb] graph recall=0.67 judge=1 5.95s
Dựa trên ngữ cảnh và dữ kiện knowledge graph:
- Hành vi bị bắt: Giang hồ "Hoàng Nato" (tên thật là Dương Minh Tuấn) bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy.
- Mức phạt tù tối đa: Theo [Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy], hành vi này có thể bị phạt tù tối đa tùy theo các khoản của Điều luật (mức án cụ thể tối đa cho từng khoản không được nêu trọn vẹn trong ngữ cảnh, nhưng ngữ cảnh có viện dẫn Điều 255 khoản 1 quy định phạt tù từ 02 năm đến 07 năm).
```
Kiểm tra cấu trúc quan hệ `MENTIONS` của Điều 255 trong Neo4j:
```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(c:Clause)-[:MENTIONS]->(s)
RETURN c.id, s.name;
```
```
(0 records)
```
Trong khi các khoản của Điều 255 thực tế có mức phạt tăng nặng:
```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(c:Clause)
RETURN c.id, c.penalty ORDER BY c.id;
```
```
- Điều 255 BLHS khoản 1: phạt tù từ 02 năm đến 07 năm
- Điều 255 BLHS khoản 2: phạt tù từ 07 năm đến 15 năm
- Điều 255 BLHS khoản 3: phạt tù từ 15 năm đến 20 năm
- Điều 255 BLHS khoản 4: phạt tù 20 năm hoặc tù chung thân
- Điều 255 BLHS khoản 5: phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng...
```

- **Nguyên nhân:** Nằm ở câu lệnh Cypher lọc khoản tại hàm `context()` trong `src/graph.py`. Câu truy vấn quy định: chỉ lấy khoản 1 (cơ bản) và các khoản có `(c)-[:MENTIONS]->(sub:Substance)` khớp với tang vật của vụ án. Tuy nhiên, Điều 255 ("Tội tổ chức sử dụng trái phép chất ma túy") cấu thành định khung tăng nặng theo hành vi (tổ chức nhiều người, đối với người chưa thành niên, gây hậu quả chết người...) chứ không liệt kê tên chất cụ thể trong luật. Vì vậy không có khoản nào của Điều 255 nối tới `Substance`, dẫn đến Cypher chỉ lấy được duy nhất `khoản 1` và bỏ sót hoàn toàn `khoản 4` (nơi có mức án tối đa là tù chung thân).
- **Đề xuất sửa:** Trong hàm `context()` tại `src/graph.py`, đối với các câu hỏi có ý định tìm mức phạt "tối đa" / "cao nhất", bổ sung logic truy xuất thêm khoản có số thứ tự cao nhất (`ORDER BY c.id DESC LIMIT 1`) hoặc lấy toàn bộ danh sách tóm tắt `penalty` của tất cả các khoản thuộc Điều luật đó thay vì chỉ phụ thuộc vào quan hệ `MENTIONS` chất.
  - *Đánh đổi:* Ngữ cảnh prompt gửi cho LLM sẽ dài thêm khoảng 150–250 tokens cho mỗi điều luật, chi phí token tăng nhẹ (~3–5%).

### Lỗi E4: Phép đo sai (Mâu thuẫn giữa lexical recall và semantic judge ở câu Q6)

- **Hiện tượng:** Ở câu Q6 (aggregation về các vụ việc liên quan đến ma túy MDMA), Flat RAG bị chấm `recall = 0.00` nhưng lại đạt điểm tối đa `judge = 2` từ LLM judge.
- **Bằng chứng:**
Trích nguyên văn câu trả lời Q6 của Flat RAG từ `ket_qua_benchmark_kg.txt`:
```
--- Q6 [aggregation] flat recall=0.00 judge=2 6.34s
Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:
1. Vụ việc thứ nhất: Lực lượng chức năng phát hiện các viên nén màu xanh bên trong thùng hàng là MDMA (Ngữ cảnh [1]).
2. Vụ việc thứ hai: Công an bắt quả tang Thành mang 5 viên ma túy đến điểm hẹn để bán, kết luận giám định xác định đây là ma túy MDMA (Ngữ cảnh [2]).
3. Vụ việc thứ ba: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA (Ngữ cảnh [3]).
```
Kiểm tra cấu hình benchmark cho Q6 trong `data/benchmark_kg.json`:
```json
"must_include": [
  "Đức",
  "Cái Quang Huy",
  "Hà Nội",
  "Lê Minh Thành",
  "Đông"
]
```

- **Nguyên nhân:** Nằm ở sự lệch pha giữa phép đo từ khóa tĩnh (`recall`) và đánh giá ngữ nghĩa (`judge`):
  1. Hàm tính `recall` trong `bench_kg.py` kiểm tra chuỗi con cứng: chuỗi trả lời phải chứa chính xác các thực thể như "Lê Minh Thành", "Cái Quang Huy", "Đức". Tuy nhiên, Flat RAG do bị giới hạn bởi chunk retrieval (chỉ retrieve được 3 đoạn nhỏ) nên nó tóm tắt dạng khái quát: "Thành mang 5 viên ma túy..." (thiếu họ "Lê Minh"), "kiện hàng", "thùng hàng" (không nêu tên Cái Quang Huy hay nước Đức). Kết quả là không trúng keyword nào trong danh sách `must_include` dẫn đến `recall = 0.00`.
  2. Ngược lại, LLM judge (sử dụng rubric ngữ nghĩa trong prompt judge) thấy câu trả lời đã nhận diện được 3 vụ việc có chứa ma túy MDMA theo đúng các mẩu tin được cung cấp, trả lời mạch lạc nên đã cho điểm tối đa `judge = 2`.
- **Đề xuất sửa:**
  1. Trong benchmark (`data/benchmark_kg.json` và `bench_kg.py`): Cho phép `must_include` hỗ trợ nhóm alias (ví dụ `["Lê Minh Thành", "Thành"]`, `["Đức", "Cái Quang Huy", "Nội Bài"]`) hoặc sử dụng LLM-based entity extraction để tính recall thay cho regex chuỗi đơn giản.
  2. Bổ sung prompt chỉ dẫn trong câu hỏi: "Hãy nêu rõ tên bị can hoặc địa danh cụ thể của từng vụ việc" để định hướng mô hình xuất ra các thực thể định danh đầy đủ.
  - *Đánh đổi:* Phép đo cần viết regex phức tạp hơn hoặc tốn thêm 1 lượt gọi LLM judge để trích xuất thực thể, tăng nhẹ thời gian chạy benchmark.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào Flat RAG là đủ:** Đối với các tác vụ tra cứu thông tin đơn chặng (single-hop) như tìm kiếm định nghĩa pháp lý (Q1: Flat và Graph đều đạt recall 1.00 / judge 2) hoặc trích xuất thông tin bị cáo từ một bài báo độc lập (Q2: cả hai đều đạt recall 1.00 / judge 2). Flat RAG tiết kiệm hơn 10 lần thời gian index (12.5s so với 137.2s), chi phí truy vấn rẻ hơn gần 6 lần ($0.00010 so với $0.00057) và vận hành đơn giản vì không đòi hỏi cơ sở dữ liệu đồ thị Neo4j.
> - **Khi nào bắt buộc dùng GraphRAG:** Khi hệ thống cần giải quyết các bài toán xuyên miền tri thức (cross-KB) và suy luận đa chặng / tổng hợp (multi-hop reasoning, aggregation). Số liệu benchmark chứng minh ưu thế áp đảo của GraphRAG: ở Q3 recall tăng từ 0.33 lên 1.00; ở Q5 recall tăng từ 0.40 lên 1.00; ở Q6 tổng hợp vụ án recall tăng từ 0.00 lên 1.00 (recall trung bình toàn bộ câu hỏi đạt 0.94 so với 0.51 của Flat RAG; điểm judge trung bình đạt 1.83 so với 1.50). Đồ thị tri thức đóng vai trò cầu nối cấu trúc giữa tình tiết vụ án thực tế với các điều khoản luật hình sự, giải quyết triệt để điểm mù "đứt gãy ngữ cảnh" mà Flat RAG không thể vượt qua.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.12s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
      [1/1] Bài báo: Góp 14 triệu đồng mua ma túy rồi nói đã ‘rút  (2 vụ)
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 25 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Trong quá trình khởi tạo môi trường ban đầu, Docker Desktop Service trên Windows gặp tình trạng treo pipe socket (`\\.\pipe\dockerDesktopLinuxEngine`) khiến tiến trình kết nối container Neo4j bị tắc nghẽn. Sau khi đóng và khởi động lại Docker Desktop bằng quyền Administrator, container Neo4j đã hoạt động ổn định trên cổng 7474 và 7687. Bên cạnh đó, mô hình `gemini-3.8-flash` bị hạ mức quota miễn phí xuống 20 RPD dẫn đến lỗi HTTP 429; nhóm đã cấu hình sang mô hình `gemini-3.5-flash-lite` (hạn mức 1,500 RPD) kết hợp cơ chế Exponential Backoff retry và cache embedding, giúp toàn bộ pipeline chạy mượt mà, vượt qua 48/48 test và hoàn thành benchmark với kết quả vượt trội.
