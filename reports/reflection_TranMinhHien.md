# Reflection và Action Plan — Trần Minh Hiền

## Mapping bài giảng vào code

| Khái niệm | Hàm/khối code | Quan sát thực tế |
|---|---|---|
| Conservative coreference | `resolve_coref_batch()`, `run_coref()` | 137/400 chunk cần LLM; có checkpoint và unresolved log. |
| Schema guard | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Precision tốt nhưng ontology hẹp làm giảm recall. |
| Bulk Cypher | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` idempotent; 195 node/117 edge. |
| Entity resolution | `build_resolution_map()`, `UF`, `merge_guard()` | ANN top-k và lexical guard tránh false merge. |
| Super-node cap | `retrieve_graph_context()` | Degree, edge và context cap bảo vệ latency/token. |
| LLM Judge | `judge_answer()`, `run_evaluation()` | Đủ 25/25 câu, checkpoint và rationale. |

## Debugging và bài học

Lỗi khó nhất là đồng bộ corpus với Golden và xử lý quota/model API. Golden upstream dùng 5.000 dòng đầu, trong khi lần chạy đầu dùng 4.000 rồi sample 1.500 bài. Tôi tải lại đúng 5.000 dòng, bỏ sample ngẫu nhiên, bắt buộc đọc Golden upstream và rebuild FAISS. Evaluation dùng checkpoint/resume; parser chuyển sang `JSONDecoder.raw_decode` để nhận JSON kèm reasoning text.

Bài học quan trọng là benchmark chỉ có ý nghĩa khi cố định corpus scope, ontology, model/prompt version và provenance. GraphRAG không mặc nhiên thắng Flat RAG; coverage của graph quyết định chất lượng nhiều hơn độ phức tạp của traversal.

## Action Plan

**Dự án:** trợ lý tra cứu tin công nghệ và theo dõi quan hệ doanh nghiệp. Tôi chọn Hybrid GraphRAG: Flat cho nội dung dài/factoid, graph cho M&A, đầu tư, nhân sự và chuỗi nhiều bước.

- **Nodes:** `Company`, `Person`, `Technology`, thêm `Event`, `PolicyCommitment`.
- **Relations:** tập hiện tại cộng `CONSIDERING`, `COMMITTED_TO`, `PROVIDES_ACCESS_TO`, `HOSTS_MODEL_FROM`.
- **Entity resolution:** aliases + lexical/type blocking + ANN + human review vùng similarity 0.85–0.92.
- **Super-node:** lọc time/relation trước, relevance rerank, context budget và vector fallback khi graph coverage thấp.
- **Monitoring:** extraction precision, provenance validity, no-seed rate, quality theo question group, latency/token và indexing cost.

## Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4 | Hiểu pipeline, provenance và giới hạn coverage. |
| Kiểm soát AI Coding Agent | 4 | Kiểm chứng upstream và không dùng Golden tự tạo. |
| Chất lượng graph | 3 | Precision tốt, coverage còn hạn chế. |
| Phân tích/debug | 4 | Có checkpoint, quota fallback và root-cause analysis. |
