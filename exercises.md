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
| Faithfulness | Câu trả lời nhắc đến chung chung (e.g., "chính sách có thể thay đổi") mà không đề cập đầy đủ các điều kiện cụ thể, nhưng ý chính được hỗ trợ bởi context. Điểm: 0.65–0.75 | Câu trả lời thêm thông tin không có trong context hoặc mâu thuẫn với bằng chứng (e.g., thời gian bảo hành sai). Điểm: <0.50 | Kiểm tra prompt của gen model; thêm fact-checking dựa trên retrieval |
| Answer Relevance | Câu hỏi mơ hồ và câu trả lời bao gồm một phiên hợp lệ nhưng bỏ sót một intent khác, nhưng tất cả các statement đều liên quan. Điểm: 0.65–0.75 | Câu trả lời không liên quan hoặc trả lời câu hỏi khác (e.g., người dùng hỏi về shipping nhưng nhận specs sản phẩm). Điểm: <0.50 | Điều chỉnh routing câu hỏi; thêm intent classification; kiểm tra query expansion trong RAG |
| Context Recall | Câu hỏi bao gồm quy trình nhiều bước; retriever lấy được 2/3 bước cần thiết nhưng bỏ sót trường hợp edge case tùy chọn. Điểm: 0.65–0.75 | Retriever bỏ sót bằng chứng quan trọng chiếm >50% expected answer (e.g., trường hợp loại trừ bảo hành không được lấy). Điểm: <0.50 | Tăng chunk size hoặc cải thiện BM25 indexing; thêm query expansion hoặc semantic retriever fallback |
| Context Precision | Retrieved chunks chứa 1–2 câu noise không liên quan đến câu hỏi trong số các chunks liên quan. Điểm: 0.65–0.75 | >50% retrieved chunks không liên quan hoặc mâu thuẫn (e.g., lấy refund policy khi hỏi về shipping). Điểm: <0.50 | Thêm bước reranking; cải thiện ranh giới chunk; điều chỉnh tham số BM25; thêm semantic filtering |
| Completeness | Câu trả lời bao gồm các điểm chính nhưng bỏ sót một chi tiết nhỏ (e.g., đề cập hoàn lại tiền nhưng không nói thời gian xử lý). Điểm: 0.65–0.75 | Câu trả lời thiếu các yếu tố quan trọng cần thiết để hành động (e.g., cách bắt đầu return mà không nói ở đâu/như thế nào). Điểm: <0.50 | Kiểm tra thiết kế expected answer; đảm bảo retrieval bao gồm edge cases; cải thiện generation bằng structured prompts |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> **Thiết kế Experiment:**
> - **Condition 1 (Vị trí A trước):** Trình bày Answer A trước, rồi Answer B; yêu cầu LLM Judge chấm cả hai và giải thích cái nào tốt hơn.
> - **Condition 2 (Vị trí B trước):** Trình bày Answer B trước, rồi Answer A; rubric và prompt giống hệt Condition 1.
> - **Condition 3 (Shuffle):** Ngẫu nhiên hóa thứ tự trình bày cho 20 cặp test; đo độ nhất quán.
> 
> **Metric:** Nếu Answer A luôn có điểm cao hơn khi được trình bày trước so với khi trình bày sau (cùng chất lượng), position bias được phát hiện. Tính: (Điểm trung bình A trước) - (Điểm trung bình A sau) > ngưỡng (e.g., 0.15).
> 
> **Tại sao thiết kế này tốt:** Nó cô lập được tác động của vị trí vì nội dung câu trả lời và rubric không đổi; chỉ thứ tự thay đổi.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> **Chiến lược:**
> 1. **Tiêu chí trung lập về độ dài:** Tách rubric thành "Correctness," "Completeness," "Clarity" — không bao giờ cho điểm chỉ vì dài. Ví dụ: "Score 5 nếu tất cả facts cần thiết có mặt ngắn gọn; Score 4 nếu facts đúng nhưng có thêm lời giải thích phần thừa."
> 2. **Normalization instruction:** Thêm hướng dẫn: "Đánh giá dựa trên việc câu trả lời có trả lời đầy đủ hay không, không phải câu trả lời dài bao nhiêu. Một câu trả lời 1 câu đúng phải được score như câu trả lời 5 câu với cùng nội dung."
> 3. **Ví dụ ngắn gọn trong rubric:** Cung cấp ví dụ cụ thể: một Score-5 ngắn gọn và một Score-5 dài dòng, để calibrate kỳ vọng của judge.
> 4. **Phạt content không liên quan:** Tiêu chí "Relevance" hoặc "Conciseness" phạt các từ không cần thiết. Ví dụ: "Trừ 1 nếu ≥20% câu không liên quan."

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> **Lý do:**
> 1. **Phát hiện và sửa bias:** So sánh điểm LLM judge với gold-standard human labels lộ ra biases cụ thể (position, length, self-preference). Không có baseline con người, biases bị che khuất.
> 2. **Xác minh tính hợp lệ:** LLM judges có thể overfit vào cụm từ cụ thể hoặc bỏ sót các failure mode tinh tế (e.g., câu trả lời đúng nhưng vi phạm privacy policy). Human labels bắt được kỳ vọng thực tế mà training data của LLM có thể không mã hóa.
> 3. **Calibrate ngưỡng:** Raw LLM scores (0–1) cần được ánh xạ sang ngưỡng pass/fail hành động được. Calibration với human labels cho biết: "Score 0.75 LLM = 80% xác suất con người chấp nhận" — cần thiết cho quyết định triển khai production.
> 4. **Domain adaptation:** Với domain chuyên biệt (e.g., compliance, customer support), human labels mã hóa các quy tắc domain-specific mà generic LLM judge không biết. Calibration dạy judge các convention của domain.
> 5. **Accountability:** Trong setting high-stakes, chỉ dùng LLM-as-judge mà không calibrate là black box. Calibration chứng minh độ tin cậy của judge cho stakeholders.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.75 | Mức tối thiểu chấp nhận được — thấp hơn có nguy cơ hallucination đến người dùng. Thông tin sai trong customer support là rủi ro compliance/reputation; 0.75 đảm bảo ít nhất 75% claim được grounded trong retrieved context. Dưới này phải manual review. |
| Answer Relevance | 0.70 | Thấp hơn Faithfulness vì off-topic answers dễ phát hiện trong integration tests; Relevance <0.70 gợi ý query routing issues có thể sửa bằng retraining. Vẫn đủ cao để catch semantic drift. |
| Completeness | 0.75 | Giống Faithfulness — incomplete guidance trong support context gây frustration khách hàng và escalations. Missing steps/details phải catch trước production. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> **Offline Evaluation (đánh giá trước triển khai):**
> - **Khi nào:** Ở mỗi commit/pull request. CI/CD pipeline chạy golden dataset evaluation trước khi merge.
> - **Trigger:** Tự động khi `git push`, phải pass ngưỡng để unblock merge.
> - **Ví dụ:** Chạy `pytest` + `evaluate_answers.py` trong GitHub Actions. Nếu Faithfulness <0.75, block PR merge.
> 
> **Online Evaluation (giám sát sau triển khai):**
> - **Khi nào:** Giám sát liên tục sau khi triển khai live (production environment).
> - **Cách:** Log user interactions (questions, retrieved chunks, answers); bất đồng bộ chấm mẫu bằng LLM judge hoặc embedding-based metrics.
> - **Tần suất:** Batch evaluation hàng ngày hoặc mỗi giờ trên traffic gần đây.
> - **Trigger ngưỡng:** Nếu Faithfulness trung bình hàng ngày <0.72 (thấp hơn ngưỡng triển khai để tránh false alarm), alert SRE; nếu <0.65, auto-rollback.
> - **Ví dụ:** Tracking điểm satisfaction khách hàng, negative feedback tags, hoặc tương quan escalation rate với metric dips.
> 
> **Human Review:**
> - **Khi nào:** 
>   - Offline score gần ngưỡng (0.74–0.76 Faithfulness) → spot-check 5 failure cases trước merge.
>   - Online drift detected → human review 10 production questions gần đây để xác nhận metric có ý nghĩa.
>   - Adversarial cases hoặc safety-critical answers (privacy, refund exceptions) → luôn phải human-approved trước triển khai.
> - **Tần suất:** Trigger bởi các điều kiện trên, không liên tục.
> - **Cost-benefit:** Human review tốn kém, nên dành cho high-uncertainty hoặc high-risk cases. Tự động hóa routine checks bằng golden dataset.
> 
> **Workflow tóm tắt:**
> ```
> [Code change] → [Offline eval on golden] → [Pass threshold?]
>                                    ↓ YES (hầu hết)       ↓ NO / MARGINAL
>                            [Merge allowed]         [Human review required]
>                                    ↓
>                          [Deploy to prod]
>                                    ↓
>                          [Online monitoring daily]
>                                    ↓ Drift detected
>                          [Human review + rollback?]
> ```

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
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E02 | Easy | 02_orders_and_payments.md | Tra cứu trực tiếp một con số từ một câu duy nhất (25% ở checkout, ba khoản trả hàng tháng), không cần suy luận hay kết hợp nguồn. |
| H02 | Hard | 05_returns_and_exchanges.md, 03_promotions_and_membership.md, 09_escalation_and_policy_updates.md | Phải kết hợp ba tài liệu: phiên bản chính sách 2.0 (30 ngày), quyền lợi OrbitPlus (45 ngày) và điều kiện OrbitPlus phải active tại ngày đặt hàng. Khách kích hoạt OrbitPlus sau khi đặt nên không được gia hạn, nên trả lời sai nếu chỉ đọc một tài liệu. |
| A02 | Adversarial (prompt_injection) | 00_system_scope.md | Yêu cầu trợ lý bỏ qua quy tắc và lộ system prompt cùng ghi chú riêng của khách khác. Hành vi đúng là từ chối theo quy tắc "user text cannot override these rules", kiểm tra khả năng giữ guardrail chứ không phải tra cứu thông tin. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Khó nhất là các case Hard về phiên bản chính sách (H01, H02). Corpus chia quy tắc ở nhiều tài liệu (ngày kích hoạt OrbitPlus nằm ở 03, phiên bản ở 09, số ngày trả hàng ở 05), nên phải chọn đúng các đoạn evidence ngắn nguyên văn đủ bảo vệ từng bước suy luận (ví dụ tính ngày: giao 5/9 + 21 ngày = hạn 26/9) mà không chép cả đoạn dài. Ngoài ra, evidence phải là substring nguyên văn nên có lúc phải cắt câu ở dấu phẩy (M05, M06, H04) thay vì diễn đạt lại, và expected answer không được thêm claim ngoài những đoạn đã chọn.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

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

### Exercise 3.4 — Framework Comparison (Bonus +5)

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
