# Báo cáo Lab 19 — GraphRAG vs Flat RAG

**Học viên:** Trần Minh Hiền · **MSSV:** 2A202601812
**Khóa:** Cohort 4 · Track 3 · **Ngày:** 20/08/2026

## Phần 1 — Thuyết minh kỹ thuật và phân tích lỗi

### 1. Coreference Resolution

Pipeline chỉ gửi 137/400 chunk có dấu hiệu đại từ cho LLM; 263 chunk đi thẳng để giảm chi phí và tránh sửa quá mức. Không có batch thất bại. Trường hợp khó là chunk `405cc9b946478641117d::c0000`: “everyone including Apple Disney and ESPN decides they want...” bị đổi thành “the companies want...”. Cách sửa không gán nhầm một công ty cụ thể nhưng làm mất sắc thái “mọi bên”, có thể sinh chủ thể mơ hồ. Vì vậy hệ thống dùng prompt conservative, giữ `unresolved_mentions`, allowlist và evidence; khi không chắc nên giữ nguyên đại từ.

### 2. Entity Resolution Threshold và Lexical Guard

Ngưỡng cosine là `0.90`, chỉ xét top-5 láng giềng FAISS. Cặp `Google Store` và `Google Play Store` có similarity `0.882936` nhưng bị từ chối: token `Play` là qualifier có ý nghĩa và hai tên chỉ hai dịch vụ khác nhau. Audit có 24 cặp: 22 `REJECT_THRESHOLD`, 1 `REJECT_GUARD`, 1 `MERGE_VECTOR`. Ưu tiên precision là hợp lý vì false merge sẽ lan lỗi sang mọi đường graph.

### 3. Đồ thị và Super-node Mitigation

Graph cuối có 195 node, 117 edge, không có edge mất provenance.

| Hạng | Entity | Type | Degree |
|---:|---|---|---:|
| 1 | Google | Company | 4 |
| 2 | Blue Solutions | Company | 3 |
| 3 | PSG | Company | 3 |

Graph mẫu chưa có node vượt ngưỡng 100. Guard vẫn giới hạn mỗi super-node 50 cạnh mới nhất, toàn truy vấn 250 cạnh và graph context 14.000 ký tự. Nó ngăn context explosion và ưu tiên tin mới; rủi ro là mất sự kiện lịch sử. Cải tiến là lọc thời gian theo câu hỏi và rerank bằng relevance kết hợp recency.

### 4. Provenance và kiểm soát schema

Mỗi edge giữ `source_chunk_id`, `published_date`, `evidence`, giúp dẫn nguồn, xử lý mâu thuẫn theo thời gian và rebuild đúng cạnh; sanity check cho `invalid provenance = 0`. Node được giới hạn ở `Company`, `Person`, `Technology`; relation gồm `ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`. Allowlist tăng precision nhưng thiếu các quan hệ Golden như `COMMITTED_TO`, `CONSIDERING`, `PROVIDES_ACCESS_TO`, `HOSTS_MODEL_FROM`, làm giảm recall. Nên mở rộng ontology có version thay vì cho relation free-form.

### 5. Benchmark 25 câu Golden chính thức

| Metric trung bình | Flat RAG | GraphRAG | Graph − Flat |
|---|---:|---:|---:|
| Comprehensiveness | 2.720 | 2.600 | -0.120 |
| Faithfulness | 4.640 | 4.520 | -0.120 |
| Multi-hop reasoning | 2.800 | 2.640 | -0.160 |
| Latency (s) | 3.837 | 2.967 | -0.870 |
| Token usage | 830.960 | 768.720 | -62.240 |

Kết quả không chứng minh GraphRAG luôn tốt hơn. Graph chỉ extraction 400 chunk và ontology hẹp nên Flat có coverage tốt hơn một chút. Graph dùng ít token và nhanh hơn trong sample chủ yếu vì graph context thường ngắn; chi phí indexing chưa nằm trong query latency.

### 6. Failure case — Flat RAG yếu hơn

`G5000-29`, câu hỏi về sự mở rộng các công ty tham gia White House AI commitments từ tháng 7 đến tháng 9: Flat đạt trung bình `3.667`, Graph `4.667` (delta `+1.0`). Flat top-k dễ ưu tiên một bài; hybrid context bổ sung entity/relationship giúp nối hai mốc và hai participant set. Phần cải thiện vẫn dựa cả vector context vì schema chưa có `COMMITTED_TO`.

### 7. Failure case — GraphRAG yếu hơn

`G5000-33`, phân biệt AP–OpenAI collaboration và voluntary governance commitment: Flat `5.0`, Graph `3.0` (delta `-2.0`). Root cause là extraction chỉ phủ 400 chunk và allowlist thiếu governance relation. `G5000-50` cũng giảm từ `2.333` xuống `1.0` vì cần tổng hợp NVIDIA, AMD, Intel nhưng graph thiếu `CONSIDERING` và positioning. Khắc phục: extraction toàn corpus, mở rộng ontology có kiểm soát, temporal/entity filters và fallback flat-only khi `NO_SEED` hoặc coverage thấp.

### 8. Trade-off

Flat RAG index rẻ, cập nhật dễ và mạnh ở factoid nhưng top-k không đảm bảo nối đủ tài liệu. GraphRAG tốn coreference, NER/RE, entity resolution và ingestion trước truy vấn; bù lại có traversal, provenance và audit. Hybrid phù hợp nhất: graph cung cấp quan hệ, vector cung cấp coverage. Tôi không dùng Golden tự tạo và không tiếp tục benchmark trên subset 4.000/1.500 ngẫu nhiên sau khi xác nhận upstream có Golden cho đúng 5.000 dòng, vì kết quả lệch scope sẽ không kiểm chứng được.

### 9. Scale 350 MB

Bottleneck đầu tiên là token/API call cho coreference và extraction, sau đó entity resolution và ingestion. Production cần streaming partition, async queue có rate limiter, checkpoint theo content hash, dead-letter queue, cache theo model/prompt version và incremental `UNWIND`. Entity resolution cần blocking theo type/lexical key rồi ANN/HNSW, tránh O(N²). Graph lớn cần community partition, degree-aware traversal, temporal filter và monitoring chi phí/chất lượng.

### 10. Khi nào chọn Flat, Graph hoặc Hybrid?

Chọn Flat cho câu hỏi mà đáp án nằm trong một đoạn và cần ingest nhanh. Chọn Graph khi quan hệ nhiều bước, tính thời gian và audit quan trọng. Chọn Hybrid khi corpus vừa có nội dung dài vừa có quan hệ; đây là lựa chọn của lab. Router nên dựa trên seed confidence, graph coverage và loại câu hỏi, không buộc mọi truy vấn đi qua graph.

## Phần 2 — Reflection và Action Plan

### Mapping bài giảng vào code

| Khái niệm | Hàm/khối code | Quan sát |
|---|---|---|
| Conservative coreference | `resolve_coref_batch()`, `run_coref()` | 137/400 chunk cần LLM; có checkpoint và unresolved log. |
| Schema guard | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Precision tốt, ontology hẹp làm giảm recall. |
| Bulk Cypher | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` idempotent; 195 node/117 edge. |
| Entity resolution | `build_resolution_map()`, `UF`, `merge_guard()` | ANN top-k và lexical guard tránh false merge. |
| Super-node cap | `retrieve_graph_context()` | Degree, edge và context cap bảo vệ latency. |
| LLM Judge | `judge_answer()`, `run_evaluation()` | Đủ 25/25 câu, checkpoint và rationale. |

### Debugging và bài học

Lỗi khó nhất là đồng bộ corpus với Golden và xử lý quota/model API. Golden upstream dùng 5.000 dòng đầu, trong khi lần chạy đầu dùng 4.000 rồi sample 1.500 bài. Tôi tải lại đúng 5.000 dòng, bỏ sample, bắt buộc đọc Golden upstream và rebuild FAISS. Evaluation được checkpoint/resume; parser đổi sang `JSONDecoder.raw_decode` để nhận JSON kèm reasoning text. Bài học: benchmark chỉ có ý nghĩa khi cố định corpus scope, ontology, model version và checkpoint.

### Action Plan

**Dự án:** trợ lý tra cứu tin công nghệ và theo dõi quan hệ doanh nghiệp. Dùng Hybrid GraphRAG: Flat cho nội dung, graph cho M&A, đầu tư, nhân sự và chuỗi nhiều bước.

- Nodes: `Company`, `Person`, `Technology`, thêm `Event`, `PolicyCommitment`.
- Relations: tập hiện tại cộng `CONSIDERING`, `COMMITTED_TO`, `PROVIDES_ACCESS_TO`, `HOSTS_MODEL_FROM`.
- Resolution: alias + lexical blocking + ANN theo type + human review vùng 0.85–0.92.
- Retrieval: temporal/relation filter, relevance rerank, context budget, fallback vector khi graph coverage thấp.
- Monitoring: extraction precision, provenance validity, no-seed rate, quality theo group, latency/token và indexing cost.

### Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4 | Hiểu pipeline, provenance và giới hạn coverage. |
| Kiểm soát AI Coding Agent | 4 | Kiểm chứng upstream, bỏ Golden tự tạo, không che failure. |
| Chất lượng graph | 3 | Precision tốt, coverage còn hạn chế. |
| Phân tích/debug | 4 | Hoàn tất checkpoint, quota fallback và RCA. |
