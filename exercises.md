# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi chitchat/xã giao, câu hỏi ngoài phạm vi (out-of-scope) cần từ chối lịch sự, hoặc câu trả lời tóm tắt không cần trích dẫn nguyên văn context. | Khách hàng hỏi thông tin kỹ thuật, chính sách hoàn tiền, bảo hành, giá cả nhưng model tự bịa đặt dữ liệu (hallucination) gây sai lệch thông tin. | Cải thiện prompt hướng dẫn grounding (chỉ trả lời từ context), kiểm tra context injection, thêm hallucination detection guardrail. |
| Answer Relevance | Câu hỏi mơ hồ/thiếu thông tin khiến bot phải hỏi lại để làm rõ (clarification), hoặc câu hỏi injection/tấn công khiến bot từ chối trả lời (refusal). | Câu hỏi khách hàng rõ ràng nhưng bot trả lời lan man, vòng vo, lạc đề hoặc né tránh vấn đề chính. | Tinh chỉnh prompt để tập trung trả lời đúng trọng tâm câu hỏi, tối ưu intent classification/query routing, loại bỏ filler phrases. |
| Context Recall | Câu hỏi đơn giản (tra cứu 1 thông tin đơn lẻ) mà chỉ cần 1 phần context nhỏ đã đủ trả lời, không cần thu hồi toàn bộ tài liệu liên quan dài dòng. | Câu hỏi phức tạp yêu cầu kết hợp nhiều điều kiện/chính sách từ nhiều tài liệu (multi-document) nhưng retriever bỏ sót tài liệu cốt lõi. | Tăng top-k retrieval, cải thiện chiến lược chunking (kích thước chunk & overlap), áp dụng hybrid search (Dense + BM25) hoặc query expansion. |
| Context Precision | Top-k trả về rất nhỏ (ví dụ k=2 hoặc 3) và LLM generator có context window lớn, không bị ảnh hưởng bởi thứ tự xuất hiện của chunk. | Top-k lớn và chunk chứa thông tin quan trọng nhất bị xếp ở cuối danh sách (lost-in-the-middle) hoặc lẫn giữa nhiều chunk nhiễu. | Áp dụng Reranker (Cross-Encoder / Reranking model) sau bước retrieval thô, tối ưu hóa thuật toán scoring của retriever. |
| Completeness | Người dùng yêu cầu tóm tắt ngắn gọn (executive summary) hoặc câu hỏi mở không có expected answer cố định mà chỉ cần ý chính. | Câu hỏi yêu cầu đầy đủ các bước thực hiện/điều kiện bắt buộc (quy trình đổi trả, điều kiện bảo hành) nhưng bot bỏ sót bước/điều kiện quan trọng. | Bổ sung checklist/guideline vào generation prompt yêu cầu liệt kê đầy đủ điều kiện, kiểm tra lại context recall xem retrieval có bị thiếu không. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Mục tiêu:** Kiểm tra xem LLM Judge có xu hướng ưu tiên câu trả lời đứng trước (vị trí Candidate A) trong so sánh cặp (pairwise evaluation) hay không.
> - **Thiết kế experiment:**
>   - Chuẩn bị tập test gồm các cặp câu trả lời $(Response_1, Response_2)$ cho cùng một câu hỏi.
>   - **Condition 1 (Thứ tự gốc):** Đưa vào prompt chấm điểm theo thứ tự `Candidate A = Response_1`, `Candidate B = Response_2`. Ghi nhận kết quả lựa chọn của Judge.
>   - **Condition 2 (Thứ tự đảo ngược):** Hoán đổi vị trí trong prompt: `Candidate A = Response_2`, `Candidate B = Response_1` với cùng system prompt và rubric.
> - **Tiêu chí phát hiện:** So sánh tỷ lệ thắng của vị trí A ở cả 2 condition. Nếu tỷ lệ chọn Candidate A cao vượt trội ở cả 2 lượt (ưu tiên vị trí A bất kể nội dung là Response 1 hay Response 2), ta kết luận LLM Judge bị position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - **Định nghĩa tiêu chí theo mức độ truyền tải thông tin (Information Density):** Rubric cần tập trung chấm điểm dựa trên số lượng key facts/ý đúng thay vì độ dài văn bản.
> - **Phạt câu trả lời dài dòng, thừa thãi (Penalize Verbosity):** Quy định rõ trong rubric: câu trả lời dài nhưng chứa thông tin lan man, lặp ý hoặc không liên quan sẽ bị trừ điểm.
> - **Khuyến khích tính súc tích (Reward Conciseness):** Quy định mức điểm tối đa (5/5) cho câu trả lời ngắn gọn, đầy đủ ý chính, rõ ràng và trực tiếp; không coi câu trả lời dài là "chi tiết hơn" nếu không bổ sung giá trị thông tin.
> - **Sử dụng Reference-based / Checklist-based Rubric:** Chấm điểm dựa trên việc đối chiếu từng ý trong checklist mong đợi thay vì cho điểm tổng thể theo cảm tính.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - **Đảm bảo tính căn chỉnh với tiêu chuẩn con người (Alignment):** LLM Judge có thể có các bias nội tại (position bias, verbosity bias, leniency/severity bias). Calibration giúp đảm bảo điểm số của LLM Judge phản ánh đúng nhận định của chuyên gia/con người trong domain cụ thể.
> - **Đo lường độ tin cậy:** Giúp tính toán độ tương quan (Spearman/Pearson correlation, Cohen's Kappa) giữa điểm của LLM Judge và Human annotations để biết khi nào có thể tin cậy tự động hóa.
> - **Phát hiện và hiệu chỉnh độ lệch:** Giúp tinh chỉnh lại prompt, few-shot examples và rubric của LLM Judge nếu phát hiện judge chấm quá lỏng tay (leniency) hoặc quá khắt khe (severity).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | >= 0.85 | Domain chăm sóc khách hàng yêu cầu thông tin chính xác tuyệt đối; hallucination về chính sách hoàn tiền, giá hay bảo hành sẽ gây tổn thất tài chính và rủi ro pháp lý lớn. |
| Answer Relevance | >= 0.80 | Đảm bảo câu trả lời trực tiếp giải quyết đúng vấn đề của khách hàng, tránh trả lời lạc đề, vòng vo gây ức chế cho người dùng. |
| Completeness | >= 0.75 | Đảm bảo cung cấp đầy đủ các điều kiện và bước thực hiện cốt lõi để khách hàng thao tác đúng quy trình, chấp nhận khác biệt nhỏ về phong cách diễn đạt. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation:** Sử dụng trong giai đoạn phát triển (development) và CI/CD pipeline trước khi release mỗi khi có thay đổi code, prompt, model hoặc retrieval pipeline. Chạy trên Golden Dataset cố định để kiểm tra regression và làm quality gate chặn bản build lỗi.
> - **Online evaluation:** Sử dụng liên tục trong môi trường production trên traffic thật của người dùng. Giúp giám sát hiệu năng theo thời gian thực (monitoring), phát hiện drift, đo lường user feedback, độ trễ và phát hiện các edge cases mới phát sinh trong thực tế.
> - **Human review:** Sử dụng cho các trường hợp rủi ro cao (high-stakes decisions), các failure cases nghiêm trọng từ production, khi xây dựng/audit Golden Dataset ban đầu, và định kỳ kiểm định mẫu (spot-check) để calibrate LLM Judge.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
