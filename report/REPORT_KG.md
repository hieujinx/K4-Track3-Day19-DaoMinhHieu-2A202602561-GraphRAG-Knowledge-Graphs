# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** …  **MSSV:** …  **Ngày:** …

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     46.7
graph       196     91958     4729   0.00934    120.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00       694       48   0.00013     1.48
graph       0.63   1.33      2934       80   0.00048     1.86
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00934 | 8.34x |
| Indexing giây | 46.7 | 120.3 | 2.58x |
| Mỗi câu: USD | 0.00013 | 0.00048 | 3.69x |
| Mỗi câu: giây | 1.48 | 1.86 | 1.26x |
| Mỗi câu: in_tok | 694 | 2934 | 4.23x |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
GraphRAG tốn thêm chi phí vì phải gọi LLM để trích xuất 20 bài báo khi dựng graph
và đưa thêm dữ kiện graph vào prompt lúc hỏi. Indexing graph tốn 0.00934 USD,
cao hơn flat 0.00112 USD; mỗi câu hỏi graph tốn trung bình 0.00048 USD và
2934 input tokens, so với 0.00013 USD và 694 tokens của Flat RAG.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi chỉ cần định nghĩa trong luật |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Câu trả lời nằm trong một bài báo |
| Q3 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Graph nối được người/vụ với luật nhưng LLM chọn nhầm Điều 249 thay vì 251 |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph | Graph tìm đúng vụ Hoàng Nato nhưng prompt không đủ dữ kiện để trả khung hình phạt tối đa |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | Graph đưa được chất và ngưỡng, nhưng LLM nhầm Điều 251 thay vì Điều 250 |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Có liệt kê vụ nhưng không khớp đầy đủ must_include |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E1: Cầu nối gãy

- **Hiện tượng:** Một vụ tin tức không có đường sang node `Charge`, nên không thể
  đi qua `Crime` tới điều luật.
- **Bằng chứng:**

```cypher
MATCH (k:Case)
WHERE NOT (k)-[:HAS_CHARGE]->()
RETURN k.name, k.doc_id;
```

```
Vụ tông cảnh sát giao thông ở An Giang
news-100260926112415229
```

- **Nguyên nhân:** Bài báo không phải vụ ma túy chính; prompt trích xuất không
  tạo `charges`, nên `build_graph` không tạo `Charge`. Đây là trường hợp không
  nên ép nối vào luật ma túy.
- **Đề xuất sửa:** Giữ khả năng `Case` không có cáo buộc ma túy, nhưng đánh dấu
  `Case.topic`/`extraction_status` để phân biệt “không áp dụng” với “trích xuất
  lỗi”; chỉ báo E1 khi bài có tội danh mà `link_entity` vẫn thất bại.

### Lỗi E2/E5: Thiếu ngữ cảnh và LLM lệch với graph

- **Hiện tượng:** GraphRAG cải thiện recall nhưng vẫn trả sai điều luật trong Q3
  và Q5.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`:

```text
Q3 graph:
... quy định tại Điều 249 Bộ luật Hình sự ...
Đáp án chuẩn: Điều 251 BLHS

Q5 graph:
... khoản 4 của Điều 251 Bộ luật Hình sự ...
Đáp án chuẩn: khoản 4 Điều 250
```

Cypher xác nhận graph có đường đúng:

```cypher
MATCH (k:Case)-[:HAS_CHARGE]->(:Charge)-[:FOR_CRIME]->(c:Crime)
      <-[:DEFINES]-(a:Article)
RETURN k.name, c.name, a.id
ORDER BY k.name;
```

```text
Vụ góp tiền mua ma túy tại Hà Nội
  mua bán trái phép chất ma túy -> Điều 251 BLHS
Vụ vận chuyển ma túy của Cái Quang Huy
  vận chuyển trái phép chất ma túy -> Điều 250 BLHS
```

- **Nguyên nhân:** `context()` lấy khoản 1 và các khoản có chất liên quan của
  nhiều điều, nhưng chưa ràng buộc đủ chặt `Article` theo `Charge`/người đang
  hỏi. Prompt dài có nhiều điều tương tự, khiến LLM chọn nhầm Điều 249/251.
- **Đề xuất sửa:** Trả facts theo từng vụ và chỉ cho phép các `Article` nối qua
  đúng `Crime`; thêm dòng kiểm chứng bắt buộc “tội danh vận chuyển -> Điều 250”
  trước khi sinh câu trả lời. Với ngưỡng, bổ sung `min_value`, `max_value` và
  `unit` trên `Threshold` để chọn khoản bằng Cypher thay vì để LLM suy đoán.

### Lỗi E3: Trùng thực thể Substance

- **Hiện tượng:** Một chất có nhiều node do khác biệt hoa thường hoặc tên gọi.
- **Bằng chứng:**

```cypher
MATCH (s:Substance)
RETURN s.name
ORDER BY toLower(s.name);
```

```text
Ketamine
ketamine
Methamphetamine
methamphetamine
```

- **Nguyên nhân:** `Substance` đang `MERGE` theo spelling LLM trả về; ontology
  chỉ chuẩn hóa tội danh bằng `link_entity`, chưa chuẩn hóa Substance hai phía.
- **Đề xuất sửa:** thêm `normalize_substance` (lowercase, bỏ dấu/alias), dùng
  `substance_id` chuẩn làm khóa `MERGE`, giữ spelling gốc ở `aliases` hoặc
  `display_name`.

### Lỗi E6: Thuộc tính quan hệ bị rỗng

- **Hiện tượng:** `PARTICIPATES_IN.charge` rỗng ở nhiều người liên quan.
- **Bằng chứng:**

```cypher
MATCH (p:Person)-[r:PARTICIPATES_IN]->(k:Case)
WHERE r.charge = '' OR r.charge IS NULL
RETURN p.name, r.role, k.name, r.charge;
```

```text
Trần Văn Trường | người liên quan | Vụ án tại Viện Pháp y tâm thần Trung ương | ''
Lê Văn Đông | bị can | Vụ án tại Viện Pháp y tâm thần Trung ương | ''
Phan Kim Nhi | người liên quan | Vụ triệt phá 8 đường dây ma túy tại TP.HCM | ''
```

- **Nguyên nhân:** Prompt cho phép `charge` là chuỗi rỗng khi bài không gán tội
  danh riêng cho người đó; một phần là hợp lý với “người liên quan”, nhưng
  trường hợp bị can cần kiểm tra lại trích xuất.
- **Đề xuất sửa:** dùng `null` thay vì chuỗi rỗng; phân biệt `role` người liên
  quan với bị can; hậu kiểm rằng mọi bị can có `charge` hoặc có lý do miễn trừ.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
KG đáng tiền khi câu hỏi cần nối người/vụ án trong tin tức với điều luật ở KB
khác: recall cross-KB tăng từ 0.00 lên 0.67 (Q3), từ 0.00 lên 0.33 (Q4),
và từ 0.60 lên 0.80 (Q5). Đổi lại GraphRAG tốn khoảng 3.69 lần chi phí
mỗi câu và 4.23 lần input tokens.
Flat RAG vẫn đủ cho Q1/Q2 dạng single-hop và rẻ hơn.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
48 passed in 0.14s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 316 node / 628 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 14 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy.

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
Đã thử Gemini nhưng hết quota chat/embedding. Benchmark đầy đủ sau đó chạy
thành công bằng OpenAI `gpt-4o-mini` và `text-embedding-3-small`.
