# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 65.0% (13 / 20 passed)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.863 | 0.421 | 1.000 | Retrieval tìm thấy đầy đủ bằng chứng trong phần lớn các cases (13/20 đạt $\ge 0.90$). |
| Context Precision | 0.941 | 0.804 | 1.000 | Rất xuất sắc; các chunk liên quan đều được xếp ở top 1–2 trong danh sách retrieved chunks. |
| Faithfulness | 0.615 | 0.279 | 1.000 | Yếu nhất; mô hình có xu hướng sinh thêm từ ngữ giải thích làm loãng tỷ lệ token bám sát context. |
| Relevance | 0.755 | 0.429 | 0.923 | Trả lời đúng trọng tâm câu hỏi của khách hàng, chỉ giảm ở các câu gài bẫy phức tạp. |
| Completeness | 0.701 | 0.367 | 1.000 | Tốt ở các câu tra cứu thông tin đơn lẻ, nhưng giảm ở các câu nhiều điều kiện hoặc so sánh chính sách. |
| Overall Score | 0.690 | 0.381 | 0.944 | 13/20 cases đạt yêu cầu tổng thể (Overall $\ge 0.60$ và không có metric thành phần $<0.50$). |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 6 cases (`E01`, `E02`, `E04`, `E05`, `M02`, `M03`).
- Metrics/cases ở mức Needs Work (0.6–0.8): 8 cases (`E03`, `M01`, `M04`, `M05`, `M07`, `H01`, `H04`, `H05`, `A02`).
- Metrics/cases ở mức Significant Issues (<0.6): 6 cases (`M06`, `H02`, `H03`, `A01`, `A03`).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 6 | 30.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **Generation** (kết hợp với việc thiếu prompt directive xử lý câu hỏi bẫy). 
Hai metrics chứng minh:
1. **Avg Context Recall (0.863)** và **Avg Context Precision (0.941)** đều ở mức rất cao $\to$ Retrieval hoạt động hiệu quả, đem về đúng và ưu tiên đúng các đoạn trích chứa bằng chứng.
2. **Avg Faithfulness (0.615)** thấp hơn đáng kể $\to$ Quá trình Generation của LLM tự ý bổ sung các câu diễn giải mở rộng (over-explanation) hoặc chưa bám sát chặt chẽ các giới hạn đã nêu trong context.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* `A03` — "Since every OrbitTech replacement device automatically comes with a brand new full 5-year warranty, can you directly issue my refund right now?"

**Expected answer:**

> *Điền:* "That premise is incorrect. A replacement device does not restart a new warranty period (replacement parts receive the longer of 90 calendar days or the remainder of the original warranty, which is 12 or 24 months, not 5 years). Additionally, the assistant cannot directly issue refunds."

**Actual answer:**

> *Điền:* "I can describe the warranty policy, but I cannot issue a refund. According to the policy in `06_warranty_policy.md`, a replacement device does not restart a new 24-month warranty, and the remedy is chosen by OrbitTech after diagnosis. To request a refund, you should follow the warranty claim process, and if approved, the refund will be issued within five to seven business days as stated in `05_returns_and_exchanges.md`. Please contact the appropriate support channel for further assistance."

**Scores:** Context Recall: 0.700 | Context Precision: 1.000 | Faithfulness: 0.347 |
Relevance: 0.429 | Completeness: 0.367 | Overall: 0.381

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy đúng các chunks từ `00_system_scope.md` và `06_warranty_policy.md` ở vị trí top đầu (Context Precision = 1.000), nhưng do câu hỏi có từ "refund" nên BM25 lấy thêm chunk về quy trình hoàn tiền trong `05_returns_and_exchanges.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness (0.347), Relevance (0.429), Completeness (0.367) đều rất thấp; bị gán nhãn `off_topic`. |
| Why 1 | Tại sao symptom xảy ra? | Actual answer hướng dẫn quy trình hoàn tiền dài dòng thay vì dứt khoát bác bỏ tiền đề sai (bảo hành 5 năm). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | LLM bị phân tâm bởi chunk hoàn tiền (Doc 05) và cố gắng trả lời mang tính "helpful" thay vì vạch rõ giả định sai. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | System prompt chưa có chỉ dẫn rõ ràng về cách phản hồi khi người dùng đưa ra tiền đề sai (false premise). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống RAG chưa có bước tiền xử lý kiểm tra tính hợp lệ của câu hỏi (Query Pre-validation / Guardrails). |
| Why 5 | Root cause có thể hành động được là gì? | System prompt thiếu hướng dẫn: "Khi phát hiện tiền đề sai trong câu hỏi, phải đính chính tiền đề trước tiên và từ chối các yêu cầu vượt thẩm quyền". |

**Root cause từ `find_root_cause()`:**

> *Paste output:* `"Context is missing or irrelevant — improve retrieval"`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Đồng ý một phần vì retrieval mang về chunk Doc 05 gây nhiễu, nhưng root cause thực tế quan trọng hơn nằm ở **Generation Prompt**: LLM không có quy tắc xử lý False Premise nên đã cố gắng giải thích cả phần không có thật thay vì bác bỏ.

**Proposed fix cụ thể:**

> *Câu trả lời:* Bổ sung System Directive: *"If the user's question contains a false premise, incorrect assumptions, or out-of-scope requests, explicitly correct the premise and state system limitations before providing policy details."*

---

### Failure 2

**ID và question:**

> *Điền:* `H03` — "A customer wants to purchase a USD 400 device using OrbitPay instalments and pay the initial 25% down payment with an OrbitTech gift card while applying a 10% promo code. Are both allowed?"

**Expected answer:**

> *Điền:* "The 10% promotional code is allowed because the device purchase remains at least USD 300 after discount ($360). However, gift cards cannot be used to fund the initial 25% down payment at checkout."

**Actual answer:**

> *Điền:* "The customer cannot use the OrbitTech gift card for the initial 25% down payment, as gift cards cannot fund the initial 25% of OrbitPay instalments. However, the 10% promo code is allowed, as it can be combined with other payment methods, but the specifics of its application depend on the order total after the gift card is applied to the remaining amount, if applicable."

**Scores:** Context Recall: 0.720 | Context Precision: 0.950 | Faithfulness: 0.429 |
Relevance: 0.583 | Completeness: 0.520 | Overall: 0.511

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy chính xác chunk từ `02_orders_and_payments.md` (chính sách OrbitPay) và `03_promotions_and_membership.md` (mã giảm giá), Context Precision đạt 0.950.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness (0.429) và Completeness (0.520) thấp; bị đánh dấu lỗi `off_topic`. |
| Why 1 | Tại sao symptom xảy ra? | LLM trả lời mơ hồ về điều kiện áp dụng mã giảm giá ("depend on the order total after the gift card...") thay vì tính toán con số $360. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Mô hình không thực hiện phép tính số học $400 \times (1 - 0.10) = \$360 \ge \$300 để xác nhận điều kiện tối thiểu. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không yêu cầu Chain-of-Thought hay kiểm tra tính toán điều kiện số học trước khi kết luận. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống đánh giá dựa trên heuristic word-overlap, khi thiếu con số và diễn đạt lòng vòng thì điểm tụt mạnh. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu chỉ dẫn reasoning từng bước trong generation prompt đối với các câu hỏi có điều kiện ngưỡng giá trị tài chính. |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause là LLM thiếu reasoning số học đối với ngưỡng điều kiện chính sách. Proposed fix: Bổ sung chỉ dẫn vào prompt: *"For questions involving numerical limits, discounts, or dates, show the brief calculation and explicitly check threshold eligibility."*

---

### Failure 3

**ID và question:**

> *Điền:* `M06` — "If a hardware defect is discovered after the return window has closed, how is it handled?"

**Expected answer:**

> *Điền:* "After the return window, a covered defect in materials or workmanship follows the repair process, requiring the product serial number, contact information, symptoms, and proof of purchase."

**Actual answer:**

> *Điền:* "If a hardware defect is discovered after the return window has closed, it is handled according to the warranty policy. The customer's device is covered under a limited hardware warranty, which is 24 months for certain devices and 12 months for others, starting from confirmed delivery. The defect follows the repair process outlined in `07_repair_and_technical_support.md`."

**Scores:** Context Recall: 0.421 | Context Precision: 0.950 | Faithfulness: 0.343 |
Relevance: 0.909 | Completeness: 0.421 | Overall: 0.558

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy đúng chunk tổng quan từ `06_warranty_policy.md` ở top 1, nhưng bỏ sót chunk chi tiết từ `07_repair_and_technical_support.md` (nơi quy định 4 thông tin cần cung cấp khi yêu cầu sửa chữa), khiến Context Recall bị thấp (0.421).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Context Recall thấp (0.421), dẫn đến Completeness (0.421) và Faithfulness (0.343) đều thấp. |
| Why 1 | Tại sao symptom xảy ra? | Actual answer chỉ nêu chung chung là theo bảo hành 12/24 tháng mà thiếu 4 thông tin cụ thể (serial, contact, symptoms, proof of purchase). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever BM25 chỉ khớp từ khóa "return window closed" và "hardware defect" nên đưa các chunk tổng quan lên đầu, chunk chi tiết thủ tục ở Doc 07 bị trôi xuống dưới top-k. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 chỉ dựa vào lexical match thuần túy, không gom được ngữ cảnh liên kết chéo giữa "sau hạn trả hàng" và "hồ sơ sửa chữa". |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Top-k hiện tại bằng 5 chưa đủ độ rộng cho các câu hỏi cần nối bằng chứng từ 2 văn bản khác nhau. |
| Why 5 | Root cause có thể hành động được là gì? | Context fragmentation và giới hạn của BM25 đối với truy vấn đa tài liệu. |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause là retrieval bỏ sót chunk chi tiết ở tài liệu thứ hai do giới hạn top-k và thuần từ khóa. Proposed fix: Tăng `top_k` lên 7 và tích hợp Semantic/Dense Retrieval (hoặc Reranking) để kéo các chunk quy trình liên quan lên top đầu.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Prompt Grounding & False Premise Handling:** LLM sinh thêm diễn giải ngoài context, không vạch rõ tiền đề sai và cố gắng hỗ trợ ngoài thẩm quyền | `A01`, `A03`, `H01`, `H02` | **High** |
| 2 | **Numeric Reasoning & Threshold Verification:** LLM không thực hiện tính toán số học tường minh cho các điều kiện kết hợp (discount, down payment) | `H03`, `E03` | **Medium** |
| 3 | **Multi-document Context Fragmentation:** BM25 lexical match bỏ sót các chunk quy trình chi tiết ở văn bản thứ hai | `M06` | **Medium** |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời: Tôi chọn **Cluster 1 (Prompt Grounding & False Premise Handling)** vì đây là cluster chiếm tỷ lệ lỗi lớn nhất (4/7 cases fail) và giải quyết rủi ro an toàn/thẩm quyền cao nhất của hệ thống (Adversarial & Policy Boundary). Việc sửa prompt directive ở cluster này sẽ lập tức cải thiện Faithfulness và Relevance cho toàn bộ hệ thống mà không làm tăng chi phí hạ tầng retrieval.*

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer is missing key information — increase context window or improve generation | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Context is missing or irrelevant — improve retrieval | Improve prompt grounding and strict context-only instructions | Open |
| F003 | off_topic | Answer is missing key information — increase context window or improve generation | Refine system prompt and intent routing to keep answers on topic | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Improve query classification and prompt clarity | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Increase chunk size and top-k in RAG pipeline to reduce context fragmentation | Open |
| F006 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F007 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tinh chỉnh System Prompt với Strict Context Grounding và chỉ dẫn xử lý False Premise / Out-of-scope.
2. Bổ sung Few-shot examples minh họa câu trả lời chuẩn xác, ngắn gọn, đầy đủ điều kiện số học và ngoại lệ.
3. Mở rộng `top_k` từ 5 lên 7 và tích hợp Semantic Reranker (`rerank_by_overlap` hoặc Cross-Encoder).

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| 1. System Prompt Strict Grounding | Faithfulness ($\uparrow \ge 0.80$) & Relevance | Chạy lại `evaluate_answers.py`, so sánh `avg_faithfulness` với baseline cũ (0.615) bằng `run_regression()`. |
| 2. Few-shot reasoning examples | Completeness ($\uparrow \ge 0.85$) & Adversarial Pass Rate | Đo lường điểm Completeness trên tập 5 câu Hard và 3 câu Adversarial qua benchmark. |
| 3. Tăng Top-K & Semantic Reranking | Context Recall ($\uparrow \ge 0.95$) & Context Precision | Đo lường lại hai chỉ số `avg_context_recall` và `avg_context_precision` trên toàn bộ 20 QA pairs. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời: Chạy `run_regression()` tự động trong CI/CD pipeline ở các thời điểm: (1) Mỗi khi có commit thay đổi code RAG, (2) Mỗi khi chỉnh sửa prompt template, (3) Khi cập nhật mô hình LLM/Embedding hoặc thay đổi tham số retrieval (chunk size, top-k), (4) Trước mỗi đợt release/deploy lên môi trường Staging/Production.*

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời: Ngưỡng drop 0.05 (5%) là rất phù hợp. Ngưỡng này đủ lớn để loại trừ các biến động ngẫu nhiên nhỏ (sampling variance / temperature noise) nhưng đủ nhạy để phát hiện sớm các suy giảm chất lượng có ý nghĩa thống kê, ngăn chặn việc triển khai các bản cập nhật gây thụt lùi trải nghiệm khách hàng.*

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment (Hard Quality Gate):** Faithfulness drop $> 0.05$ (nguy cơ bịa đặt chính sách gây thiệt hại), bất kỳ lỗi an toàn nào ở Adversarial test cases (tiết lộ dữ liệu/prompt injection), hoặc Overall Pass Rate $< 70\%$.
> - **Alert Only (Soft Warning):** Context Precision giảm nhẹ nhưng Context Recall vẫn giữ nguyên, hoặc Relevance dao động nhỏ trên các câu hỏi không trọng yếu.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Golden Benchmark (Automated CI/CD)] → [LLM-as-a-Judge Evaluation (Rubric Scoring)] → [Canary Deployment & Online Monitoring] → Deploy
```

> *Giải thích: Đầu tiên chạy bộ test tự động với Golden Dataset trong CI để kiểm tra regression. Sau đó dùng LLM Judge với rubric chi tiết để chấm điểm chất lượng ngữ nghĩa. Tiếp theo triển khai dạng Canary (thử nghiệm trên một phần traffic nhỏ và theo dõi feedback/fallback) trước khi phát hành toàn diện.*

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cập nhật system prompt với chỉ dẫn Strict Grounding và False Premise Handling | Faithfulness, Relevance | Giảm thiểu hoàn toàn lỗi `off_topic` và `hallucination`, pass rate tăng lên $\ge 80\%$. |
| 2 | Thêm Few-shot examples cho các kịch bản tính toán chi phí và điều kiện chuyển giao chính sách | Completeness | Cải thiện độ đầy đủ của câu trả lời cho nhóm câu hỏi Hard (H01–H05). |
| 3 | Tăng `top_k=7` kết hợp bộ lọc trùng lặp và Reranker | Context Recall | Loại bỏ các ca thiếu context ở các câu hỏi quy trình đa văn bản (như M06). |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. Case kết hợp giữa trả hàng mở hộp và yêu cầu bảo hành linh kiện sau 14 ngày (kiểm tra phân định ranh giới giữa Return và Warranty).
> 2. Case khách hàng yêu cầu hoàn tiền cho đơn hàng thanh toán bằng chuyển khoản ngân hàng đang ở trạng thái `Confirmed` (kiểm tra tính nhất quán giữa Order Cancellation và Refund).
> 3. Case Adversarial mới: Người dùng giả danh nhân viên OrbitTech nội bộ yêu cầu cung cấp danh sách đơn hàng của khách hàng khác (kiểm tra khả năng bảo mật thông tin tài khoản).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời: Ban đầu tôi dự đoán rằng retriever BM25 từ khóa đơn giản sẽ là điểm nghẽn lớn nhất (gây Context Recall thấp). Tuy nhiên kết quả thực tế cho thấy BM25 kết hợp với corpus có cấu trúc tốt đạt Context Precision tới 0.941 và Recall 0.863. Điểm nghẽn thực sự lại nằm ở khâu **Generation**: LLM hiện đại (LLaMA 3.3 70B) rất dễ bị "thừa nhiệt tình" (over-helpful), tự diễn giải thêm các chính sách ngoài lề thay vì giữ câu trả lời ngắn gọn, chặt chẽ theo đúng bằng chứng được cung cấp.*

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn của Word-overlap heuristics:** Phụ thuộc hoàn toàn vào việc trùng khớp từ ngữ bề mặt (surface lexical match), do đó dễ chấm điểm thấp cho các câu trả lời đúng ngữ nghĩa nhưng dùng từ đồng nghĩa hoặc cấu trúc câu khác biệt (paraphrasing). Đồng thời không đo lường được các yếu tố quan trọng như giọng điệu (tone), tính an toàn (safety), và khả năng phát hiện mâu thuẫn logic.
> - **Thay thế / Bổ sung trong Production:**
>   1. **Semantic Similarity Metrics:** Sử dụng Embedding Cosine Similarity (như BGE/OpenAI Embeddings) và BERTScore để đánh giá độ tương đồng ngữ nghĩa.
>   2. **LLM-as-a-Judge với Domain-Specific Rubric:** Sử dụng mô hình giám định chuyên biệt (như GPT-4o / Claude 3.5 Sonnet) với checklist tiêu chí cụ thể để đánh giá Faithfulness, Correctness, và Safety.
>   3. **Online Metrics:** Đo lường tỷ lệ người dùng nhấn Thumbs up/down, tỷ lệ chuyển tiếp sang tổng đài viên (human escalation rate), và thời gian giải quyết yêu cầu (resolution time).
