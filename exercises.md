# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu trả lời bổ sung thông tin lời chào/định dạng xã giao hoặc chủ động từ chối trả lời do Context không có dữ liệu. | Model bị hallucination, tự bịa đặt thông tin sai lệch so với Context (học phí, quy chế, lịch thi). | Thắt chặt System Prompt (chỉ dùng Context), hạ Temperature về 0.0, kiểm tra ranh giới Retriever/Generator. |
| Answer Relevance | Câu hỏi mơ hồ hoặc câu xã giao ("Chào bạn") làm bot đưa ra phản hồi tổng quát/xác nhận lại ý định. | Câu trả lời hoàn toàn lạc đề, trả lời sang một chủ đề khác không liên quan đến thắc mắc của sinh viên. | Tinh chỉnh Prompt generation đi thẳng vào trọng tâm, cải thiện Query Rewriting/Expansion để bắt đúng Intent. |
| Context Recall | Expected Answer chứa thông tin lề không quan trọng hoặc câu hỏi đơn giản không cần tới toàn bộ tài liệu dài. | Retriever bỏ sót các thông tin/bằng chứng cốt lõi (ground truth evidence) cần thiết để trả lời câu hỏi. | Tăng `top_k`, áp dụng Hybrid Search (BM25 + Dense Embeddings), tối ưu Chunking strategy hoặc cập nhật dữ liệu nguồn. |
| Context Precision | Thu thập ra nhiều Chunks bổ trợ tốt nhưng Chunk chính đứng ở vị trí 2-3 thay vì vị trí 1 (LLM vẫn lọc được). | Relevant Chunks nằm ở vị trí quá thấp hoặc top Chunks retrieved toàn là nhiễu, gây lỗi "Lost in the Middle". | Tích hợp Reranker (Cross-Encoder / Cohere Rerank), fine-tune Embedding model, lọc bớt noise chunks trước Generator. |
| Completeness | Sinh viên chỉ cần câu trả lời tóm tắt ngắn gọn thay vì liệt kê toàn bộ các điều khoản tiểu tiết trong Expected Answer. | Câu trả lời bỏ sót các bước xử lý hoặc điều kiện bắt buộc (ví dụ: thiếu hạn nộp đơn hoãn thi khiến sinh viên trượt môn). | Tinh chỉnh Prompt generation yêu cầu trả lời đầy đủ các ý cốt lõi, áp dụng Structured Output / Multi-step extraction. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition A (Direct Order)**: Đưa hai câu trả lời A và B cho LLM Judge dưới dạng Pairwise Evaluation theo thứ tự `[Answer A, Answer B]` và yêu cầu chọn câu trả lời tốt hơn hoặc chấm điểm từng câu.
> - **Condition B (Swapped Order)**: Đảo vị trí của hai câu trả lời thành `[Answer B, Answer A]`, giữ nguyên Question, Context và System Prompt.
> - **Đánh giá & Metrics**: So sánh tỷ lệ thắng (Win Rate) hoặc điểm trung bình của Answer A ở Condition A vs Condition B. Nếu câu trả lời xuất hiện ở vị trí đầu tiên (Position 1) liên tục nhận điểm cao hơn hoặc thắng nhiều hơn đáng kể bất kể nội dung (p < 0.05), hệ thống có hiện tượng **Position Bias**.
> - **Giải pháp**: Chấm điểm theo cả 2 chiều (Swap Order) rồi lấy điểm trung bình, hoặc chuyển sang Single-Answer Evaluation dựa trên Rubric thay vì Pairwise Comparison.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Bổ sung quy tắc Conciseness & Mật độ thông tin**: Thêm chỉ dẫn trực tiếp vào Prompt của LLM Judge: *"Không cộng điểm cho các câu trả lời dài dòng, hoa mỹ hoặc lặp lại thông tin không cần thiết. Đánh giá dựa trên lượng thông tin đúng và hữu ích."*
> 2. **Chấm điểm theo các Key Information Points (KIPs)**: Định nghĩa tiêu chuẩn chấm 5/5 dựa trên việc trả lời đúng và đủ các ý chính cốt lõi (Key Points), không phụ thuộc vào độ dài câu từ.
> 3. **Phân tách chiều đánh giá (Multi-dimensional Scoring)**: Tách riêng chiều *Correctness/Factuality* và chiều *Conciseness*. Nếu phản hồi quá dài dòng hoặc thừa thông tin, chiều *Conciseness* sẽ bị trừ điểm, giúp ngăn chặn việc "câu dài làm lu mờ lỗi sai factual".

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - **Đo lường khoảng cách Alignment (Human Alignment Gap)**: LLM Judge luôn có nguy cơ mắc các dạng bias (position, verbosity, self-preference) và nhạy cảm với prompt. Calibration giúp tính toán chỉ số tương quan (như Cohen's Kappa, Spearman Correlation) giữa điểm của LLM Judge và đánh giá của chuyên gia (Human Annotators).
> - **Phát hiện lỗi hệ thống (Systematic Errors)**: Nhận diện xem LLM Judge có xu hướng quá dễ dãi (lenient) hay quá khắt khe (strict) ở từng tiêu chí cụ thể nào để điều chỉnh prompt/rubric.
> - **Đảm bảo tính tin cậy trước khi tự động hóa**: Giúp tinh chỉnh Prompt, Rubric và Few-shot Examples cho LLM Judge cho đến khi đạt độ đồng thuận cao (> 0.8) với Human Baseline trước khi tin tưởng sử dụng trong CI/CD pipeline tự động.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Hệ thống Student Services tư vấn quy chế, học phí, lịch thi yêu cầu độ chính xác tuyệt đối dựa trên tài liệu. Score < 0.85 có rủi ro cao gây hallucination, dẫn tới hậu quả sai lệch thông tin nghiêm trọng cho sinh viên. |
| Answer Relevance | 0.80 | Đảm bảo phản hồi giải quyết đúng và trúng thắc mắc của sinh viên. Score < 0.80 thể hiện câu trả lời lan man, trả lời không đúng ý câu hỏi, gây trải nghiệm người dùng kém. |
| Completeness | 0.75 | Đảm bảo cung cấp đủ các bước/điều kiện quan trọng. Chọn threshold 0.75 vì một số câu hỏi chỉ cần câu trả lời ngắn gọn/tóm tắt là đạt yêu cầu thực tế mà không nhất thiết phải liệt kê 100% tiểu tiết. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment / CI/CD Pipeline)**:
>   - *Thời điểm*: Thực hiện tự động trong pipeline CI/CD mỗi khi có Pull Request, thay đổi prompt, đổi embedding model hoặc tinh chỉnh RAG workflow.
>   - *Mục đích*: Chạy trên Golden Dataset cố định (20–100 QA pairs) để phát hiện sớm các lỗi suy giảm chất lượng (Regression Errors) trước khi merge code hoặc deploy lên production.
> - **Online Evaluation (Post-deployment / Production Monitoring)**:
>   - *Thời điểm*: Thực hiện liên tục 24/7 trên luồng hội thoại thực tế của người dùng tại môi trường Production.
>   - *Mục đích*: Theo dõi hiệu năng thực tế qua các tín hiệu từ người dùng (Thumbs Up/Down, CSAT), tín hiệu ngầm (tỷ lệ hỏi lại, drop-off rate, latency, cost per query) và thu thập telemetry để phát hiện các edge cases mới mà Golden Dataset chưa bao phủ.
> - **Human Review (Manual Audit / Expert Evaluation)**:
>   - *Thời điểm*: Thực hiện định kỳ theo chu kỳ (ví dụ audit 5–10% logs hàng tuần), hoặc kích hoạt khẩn cấp khi phát hiện Anomaly (online metrics sụt giảm đột ngột), hoặc khi xây dựng/cập nhật Golden Dataset.
>   - *Mục đích*: Cung cấp Ground Truth chuẩn xác nhất, dùng để Calibrate lại LLM-as-a-Judge, và đánh giá các yếu tố tinh tế như tone giọng (tone of voice), tính an toàn (safety) và quy trình nghiệp vụ thực tế của nhà trường.

---

## Part 2 — Core Coding (09:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

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

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
