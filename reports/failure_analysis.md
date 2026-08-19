# Failure Analysis — Flat RAG vs GraphRAG

**Phạm vi:** 25 câu Golden chính thức `G5000-26`–`G5000-50`, 5.000 raw rows, FAISS trên 2.105 chunk và graph 195 node/117 edge.

## Ca 1 — Flat RAG thiếu liên kết cross-document

- **Question:** `G5000-29` — sự mở rộng thành viên White House AI commitments từ tháng 7 đến tháng 9/2023.
- **Kết quả:** Flat `3.667`, Graph `4.667`, delta `+1.0`.
- **Triệu chứng:** Flat top-k dễ lấy một bài có similarity cao nhưng không bao quát đủ hai participant set và hai mốc thời gian.
- **Root cause:** Vector retrieval xếp hạng từng chunk độc lập, không biểu diễn rõ liên kết entity–event–time giữa tài liệu.
- **Vì sao Hybrid tốt hơn:** Seed/entity context giúp nối các công ty và sự kiện, trong khi vector chunks bổ sung nội dung mà ontology chưa biểu diễn.
- **Khắc phục lâu dài:** thêm `PolicyCommitment`/`COMMITTED_TO`, temporal constraints và diversity-aware retrieval theo document.

## Ca 2 — GraphRAG mất thông tin do coverage/schema

- **Question:** `G5000-33` — phân biệt AP–OpenAI collaboration và voluntary governance commitment.
- **Kết quả:** Flat `5.0`, Graph `3.0`, delta `-2.0`.
- **Triệu chứng:** Graph context thiếu quan hệ governance cần thiết; vector context lại lấy đúng các đoạn nguồn.
- **Root cause:** graph chỉ extraction 400 chunk và allowlist không có relation `COMMITTED_TO`; seed/traversal không thể tìm cạnh chưa từng được tạo.
- **Khắc phục:** extraction toàn corpus hoặc theo evidence coverage, mở rộng ontology có version, đo graph coverage trước generation và chuyển flat-only khi graph không đủ bằng chứng.

## Ca 3 — Multi-entity comparison vượt ontology

- **Question:** `G5000-50` — so sánh tín hiệu AI chip của NVIDIA, AMD và Intel.
- **Kết quả:** Flat `2.333`, Graph `1.0`.
- **Root cause:** cần tổng hợp ba công ty từ nhiều tài liệu, nhưng schema thiếu `CONSIDERING` và các quan hệ market-positioning; graph context ngắn trở thành nhiễu thay vì bổ sung.
- **Khắc phục:** relation taxonomy theo event, query decomposition cho từng entity, temporal rerank, sau đó aggregate có provenance.

## Kết luận

Failure không chỉ nằm ở generation. Chuỗi nguyên nhân là corpus coverage → coreference/extraction → ontology → entity resolution → seed/traversal → context. Vì vậy hệ thống production phải log provenance, no-seed rate, graph coverage, retrieval route và score theo từng nhóm câu hỏi.
