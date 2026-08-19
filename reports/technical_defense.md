# Thuyết minh kỹ thuật — Lab 19

**Học viên:** Trần Minh Hiền · **MSSV:** 2A202601812

## 1. Coreference Resolution

Pipeline chỉ gửi 137/400 chunk có dấu hiệu đại từ cho LLM, còn 263 chunk được giữ nguyên. Không có batch lỗi. Ở chunk `405cc9b946478641117d::c0000`, “everyone including Apple Disney and ESPN decides they want...” bị đổi thành “the companies want...”. Cách sửa không gán nhầm một công ty cụ thể nhưng làm mất sắc thái “mọi bên”, có thể tạo chủ thể mơ hồ. Do đó prompt phải conservative, giữ `unresolved_mentions` và không cưỡng ép khi antecedent không rõ.

## 2. Entity Resolution

Ngưỡng cosine là `0.90`, tìm top-5 ứng viên bằng FAISS rồi kiểm tra lexical guard. `Google Store` và `Google Play Store` có similarity `0.882936` nhưng bị từ chối vì `Play` là qualifier có nghĩa; đây là hai dịch vụ khác nhau. Audit có 24 dòng: 22 `REJECT_THRESHOLD`, 1 `REJECT_GUARD`, 1 `MERGE_VECTOR`. False merge nguy hiểm hơn false split vì lỗi sẽ lan qua mọi đường graph.

## 3. Super-node Mitigation

Top degree thực tế:

| Hạng | Entity | Type | Degree |
|---:|---|---|---:|
| 1 | Google | Company | 4 |
| 2 | Blue Solutions | Company | 3 |
| 3 | PSG | Company | 3 |

Graph mẫu không có node vượt ngưỡng 100. Policy vẫn tự giới hạn super-node còn 50 cạnh mới nhất, toàn truy vấn 250 cạnh và graph context 14.000 ký tự. Ưu điểm là tránh context explosion; rủi ro là mất sự kiện cũ. Cải tiến là lọc theo thời gian câu hỏi và rerank relevance kết hợp recency.

## 4. Edge Provenance

Mỗi edge lưu `source_chunk_id`, `published_date`, `evidence`, `confidence`. Sanity check đạt `invalid_provenance_edges = 0`. Provenance cho phép dẫn nguồn, phân biệt sự kiện theo thời gian và rebuild đúng cạnh khi nguồn thay đổi; triple không provenance không thể audit đáng tin cậy.

## 5. Schema Allowlist

Node chỉ gồm `Company`, `Person`, `Technology`; relation gồm `ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`. Allowlist tăng precision nhưng thiếu các quan hệ Golden như `COMMITTED_TO`, `CONSIDERING`, `PROVIDES_ACCESS_TO`, `HOSTS_MODEL_FROM`, làm giảm recall. Nên mở rộng ontology có version, không cho relation free-form.

## 6. Benchmark

| Metric trung bình | Flat RAG | GraphRAG | Graph − Flat |
|---|---:|---:|---:|
| Comprehensiveness | 2.720 | 2.600 | -0.120 |
| Faithfulness | 4.640 | 4.520 | -0.120 |
| Multi-hop reasoning | 2.800 | 2.640 | -0.160 |
| Latency (s) | 3.837 | 2.967 | -0.870 |
| Token usage | 830.960 | 768.720 | -62.240 |

Graph không luôn tốt hơn: graph chỉ extraction 400 chunk và ontology hẹp nên Flat có coverage nhỉnh hơn. Graph context ngắn làm query dùng ít token và nhanh hơn trong sample, nhưng bảng chưa tính indexing cost.

## 7. Failure mode của Flat RAG

Ở `G5000-29`, câu hỏi nối White House AI commitments tháng 7 và tháng 9, Flat đạt `3.667`, Graph `4.667`. Vector top-k dễ ưu tiên một tài liệu; hybrid context giúp nối hai mốc và hai participant set. Tuy nhiên kết quả vẫn cần vector fallback vì schema chưa có `COMMITTED_TO`.

## 8. Failure mode của GraphRAG

Ở `G5000-33`, Flat đạt `5.0`, Graph `3.0`. Graph thiếu coverage và relation governance nên không phân biệt đủ AP–OpenAI collaboration với voluntary commitment. `G5000-50` giảm từ `2.333` xuống `1.0` vì thiếu quan hệ positioning/`CONSIDERING`. Cần extraction rộng hơn, ontology có kiểm soát và fallback flat-only khi `NO_SEED` hoặc coverage thấp.

## 9. Trade-off và kiểm soát AI Agent

Flat index rẻ, cập nhật dễ, mạnh ở factoid nhưng không đảm bảo nối tài liệu. Graph tốn coreference, NER/RE, resolution và ingestion, đổi lại có traversal và audit. Hybrid cân bằng hai phía. Tôi từ chối tiếp tục dùng Golden tự tạo và subset 4.000/1.500 ngẫu nhiên sau khi xác nhận upstream có Golden chính thức cho 5.000 dòng, vì benchmark lệch scope không thể kiểm chứng.

## 10. Scale lên 350 MB

Bottleneck đầu tiên là token/API call extraction, sau đó entity resolution và ingestion. Production cần streaming partition, async rate-limited queue, checkpoint theo content hash, dead-letter queue, cache theo model/prompt version và incremental `UNWIND`. Entity resolution dùng lexical/type blocking rồi ANN/HNSW, tránh O(N²). Retrieval cần community partition, temporal filter, degree cap và monitoring cost/quality.
