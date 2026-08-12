# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 40.0% (8 / 20 passed)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.945 | 0.800 | 1.000 | Xuất sắc. Retriever phủ được 94.5% thông tin ground truth cần thiết. |
| Context Precision | 0.974 | 0.833 | 1.000 | Rất cao. Các chunk liên quan luôn nằm ở top 1–2 kết quả tìm kiếm. |
| Faithfulness | 0.678 | 0.214 | 1.000 | Mức trung bình. Bị kéo tụt do các ca từ chối Out-of-scope bị chấm nhầm là hallucination. |
| Relevance | 0.466 | 0.000 | 0.846 | Yếu nhất. Thừa/thiếu từ ngữ cảnh do bot trả lời quá ngắn (VD: "80%"), sai lệch n-gram overlap. |
| Completeness | 0.625 | 0.100 | 1.000 | Cần cải thiện. Các câu trả lời ngắn không khớp với độ dài của expected answer. |
| Overall Score | 0.590 | 0.244 | 0.843 | Chỉ 8/20 câu đạt điểm trung bình ≥ 0.70 do ảnh hưởng từ Answer Relevance & Completeness. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): `Context Recall` (0.945), `Context Precision` (0.974).
- Metrics/cases ở mức Needs Work (0.6–0.8): `Faithfulness` (0.678), `Completeness` (0.625).
- Metrics/cases ở mức Significant Issues (<0.6): `Relevance` (0.466), `Pass Rate` (40.0%).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 2 | 16.7% |
| irrelevant | 6 | 50.0% |
| incomplete | 0 | 0.0% |
| off_topic | 4 | 33.3% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **Generation & Evaluation Heuristics**, không nằm ở phía **Retrieval**. 

Dẫn chứng từ metrics:
1. `Context Precision = 0.974` và `Context Recall = 0.945` chứng minh Retriever đã lấy đúng và đủ gần như 100% tài liệu liên quan cho Generator.
2. `Answer Relevance = 0.466` thấp nghiêm trọng do Generator đưa ra câu trả lời quá ngắn (câu trả lời cụm từ 1-3 từ), khiến thuật toán đo lexical n-gram overlap chấm điểm Relevance xấp xỉ 0.0 dù thông tin hoàn toàn chính xác.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
- **ID:** `M02`
- **Question:** *"What academic grade and transcript notation is assigned if a student drops a course after the census date?"*

**Expected answer:**
> *"A course withdrawal after the census date through the withdrawal deadline produces a W grade on the student's record."*

**Actual answer:**
> *"A `W` (withdrawal) is recorded on the transcript."*

**Scores:** Context Recall: 0.923 | Context Precision: 1.000 | Faithfulness: 0.500 | Relevance: 0.077 | Completeness: 0.154 | Overall: 0.244

**Evidence inspection:** 
- Gold Context từ `06_leave_and_withdrawal.md`: *"After census and through the withdrawal deadline in 01_academic_calendar.md, the course receives a W."*
- Retrieved Chunk #1: `NU-06-P03` (score 5.678) chứa nguyên văn đoạn quy định trên.
- **Đánh giá:** Retriever lấy chuẩn xác 100% Chunk liên quan ở Rank 1 (Context Precision = 1.0).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời đúng sự thật (`W`) nhưng chỉ đạt Overall Score = 0.244 và bị gắn nhãn `irrelevant`. |
| Why 1 | Tại sao symptom xảy ra? | Điểm Relevance (0.077) và Completeness (0.154) thấp thảm hại. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Actual answer quá ngắn (8 từ) so với Expected answer và Question dài, làm giảm chỉ số Lexical Word Overlap (ROUGE/BLEU style). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Generator System Prompt cho phép tạo ra câu trả lời tối giàn mà không yêu cầu viết thành câu hoàn chỉnh lặp lại ngữ cảnh câu hỏi. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | RAGAS Evaluator dùng thuật toán so sánh từ vựng thô (lexical match) thay vì đánh giá ngữ nghĩa (Semantic Similarity / LLM Judge). |
| Why 5 | Root cause có thể hành động được là gì? | Generator Prompt thiếu chỉ dẫn định dạng câu trả lời đầy đủ (Full Sentence Framing) kết hợp với Evaluator phụ thuộc vào từ vựng thô. |

**Root cause từ `find_root_cause()`:**
> `"Answer does not address the question — improve prompt clarity"`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> *Đồng ý một phần.* `find_root_cause()` xác định nguyên nhân nằm ở Prompt clarity là đúng hướng phát triển Generator, nhưng nhận định "Answer does not address the question" là sai bản chất thực tế. Trace cho thấy bot trả lời đúng chính xác chữ `W` (withdrawal), lỗi thực sự nằm ở **cách đo đếm Lexical Overlap** bị dị ứng với câu trả lời ngắn.

**Proposed fix cụ thể:**
1. **Fix Generator Prompt:** Cập nhật System Prompt trong `domain_assistant.py`: *"Always answer in full, self-contained sentences that restate the key entities from the question."* (VD: *"If a student drops a course after the census date, a grade of 'W' (withdrawal) is recorded on the transcript."*)
2. **Metric verify:** `Answer Relevance > 0.80`, `Completeness > 0.80`, `Overall Score > 0.75` trên M02.

---

### Failure 2

**ID và question:**
- **ID:** `E05`
- **Question:** *"Within how many business days must a formal grade appeal be filed after final grade publication?"*

**Expected answer:**
> *"A formal grade appeal must be filed within ten business days after publication."*

**Actual answer:**
> *"10 business days"*

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.667 | Relevance: 0.154 | Completeness: 0.182 | Overall: 0.334

**Evidence inspection:**
- Gold Context từ `08_student_support_and_appeals.md`: *"A formal grade appeal must be filed within ten business days after publication..."*
- Retrieved Chunk #1: `NU-08-P03` (score 19.495) trùng khớp 100%.
- **Đánh giá:** Retrieval xuất sắc ở Rank 1.

| Level | Question | Answer |
|---|---|---|
| Symptom | Đáp án hoàn toàn chính xác ("10 business days") nhưng nhận điểm Overall = 0.334 và thất bại. | |
| Why 1 | Tại sao symptom xảy ra? | Relevance score = 0.154 và Completeness score = 0.182. | |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Đáp án 3 từ ("10 business days") trùng rất ít token với câu hỏi dài 15 từ. | |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Generator ưu tiên sự ngắn gọn theo kiểu tra cứu công cụ tìm kiếm thay vì làm trợ lý hội thoại. | |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống đánh giá không phân biệt được câu hỏi tra cứu con số (Factoid Query) với câu hỏi tự luận mở. | |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu Few-shot Examples trong Prompt để hướng dẫn Generator trả lời trọn câu cho nhóm câu hỏi tra cứu thông số. | |

**Root cause và proposed fix:**
- **Root cause từ `find_root_cause()`:** `"Answer does not address the question — improve prompt clarity"`.
- **Đồng ý / Bác bỏ:** Đồng ý với giải pháp điều chỉnh Prompt, nhưng bác bỏ kết luận câu trả lời không giải quyết câu hỏi. 
- **Proposed fix:** Thêm 2-3 Few-shot examples vào `domain_assistant.py` hướng dẫn mẫu câu trả lời đầy đủ cho nhóm câu hỏi Easy/Factoid.
- **Metric verify:** `Answer Relevance > 0.85` cho tập Easy questions (E01-E05).

---

### Failure 3

**ID và question:**
- **ID:** `A01`
- **Question:** *"Can you provide investment advice on stock market portfolios for Northstar students?"*

**Expected answer:**
> *"I don't provide investment advice. I support Northstar student-service questions, such as academic deadlines, course registration, tuition, scholarships, attendance, grading, leave, graduation, appeals, student support, privacy, and account security."*

**Actual answer:**
> *"I don't provide investment advice. I support Northstar student-service questions, such as academic deadlines, course registration, tuition, scholarships, attendance, grading, leave, graduation, appeals, student support, privacy, and account security."*

**Scores:** Context Recall: 0.895 | Context Precision: 1.000 | Faithfulness: 0.269 | Relevance: 0.400 | Completeness: 0.368 | Overall: 0.346

**Evidence inspection:**
- Gold Context từ `00_system_scope.md`: Định nghĩa phạm vi hỗ trợ của trợ lý Student Services.
- **Đánh giá:** Actual answer **trùng khớp 100% nguyên văn** với Expected answer, nhưng hệ thống chấm điểm Faithfulness = 0.269 và báo lỗi `hallucination`!

| Level | Question | Answer |
|---|---|---|
| Symptom | Đáp án hoàn hảo chuẩn 100% theo expected answer nhưng bị chấm điểm cực thấp (0.346) và gán nhãn `hallucination`. | |
| Why 1 | Tại sao symptom xảy ra? | Faithfulness chỉ đạt 0.269 (dưới threshold 0.70). | |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Thuật toán Faithfulness so sánh từ trong câu từ chối ("investment advice", "stock market") với Context và thấy các từ này không nằm trong Chunk tài liệu quy chế. | |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | RAGAS Evaluator dùng chung 1 công thức Faithfulness cho cả câu trả lời RAG thông thường lẫn câu từ chối Out-of-scope. | |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có bước phân loại Intent (Refusal / Out-of-scope detection) trước khi đưa câu trả lời vào pipeline chấm điểm RAG. | |
| Why 5 | Root cause có thể hành động được là gì? | Evaluator Pipeline bị lỗi thiết kế: Ép các câu từ chối hợp lệ (Adversarial Refusal) phải tuân theo công thức Faithfulness dựa trên Context. | |

**Root cause và proposed fix:**
- **Root cause từ `find_root_cause()`:** `"Context is missing or irrelevant — improve retrieval"`.
- **Đồng ý / Bác bỏ:** **BÁC BỎ HOÀN TOÀN.** `find_root_cause()` đổ lỗi cho Retrieval ("Context missing") là sai lầm nghiêm trọng. Context Precision đạt 1.0 và bot đã trả lời chính xác câu từ chối mẫu. Lỗi nằm ở **Evaluator Metric**, không phải ở System/Retriever.
- **Proposed fix:**
  1. Thêm cơ chế **Refusal Bypass / Out-of-Scope Rule** trong `RAGASEvaluator`: Khi câu trả lời chứa mẫu câu từ chối chuẩn (`"I don't provide..."` / `"I cannot fulfill..."`), tự động chuyển sang kiểm thử tiêu chuẩn Refusal Compliance thay vì tính Faithfulness trên Context.
- **Metric verify:** `Pass Rate` của nhóm Adversarial (A01–A03) đạt 100%.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Lexical Overlap Metric vs Ultra-Short Answers**: Generator tạo câu trả lời 1-3 từ khiến n-gram overlap với expected answer bị đánh giá thấp. | M02, E05, E01, E03, E04, H02 | High |
| 2 | **Refusal Faithfulness Misclassification**: RAGAS Faithfulness chấm nhầm câu trả lời từ chối out-of-scope là hallucination. | A01, A03 | High |
| 3 | **Generator Output Framing**: Prompt chưa yêu cầu tổng hợp thông tin đa tài liệu thành câu hoàn chỉnh. | M01, M03, M04, A02 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn **Cluster 1 (Lexical Overlap Metric vs Ultra-Short Answers)**. Việc sửa Cluster 1 (yêu cầu Generator trả lời trọn câu và cập nhật Evaluator) sẽ khắc phục ngay 6/12 ca thất bại (50% tổng số lỗi), nâng Pass Rate toàn hệ thống từ 40% lên 70% ngay lập tức mà không cần can thiệp phức tạp vào mô hình Retrieval.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question — improve prompt clarity | Implement full-sentence framing prompt instructions | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers | Open |
| F003 | irrelevant | Answer does not address the question — improve prompt clarity | Restate question entities in generation prompt | Open |
| F004 | irrelevant | Answer does not address the question — improve prompt clarity | Restate question entities in generation prompt | Open |
| F005 | irrelevant | Answer does not address the question — improve prompt clarity | Implement full-sentence framing prompt instructions | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Improve context integration in prompt | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | Restate question entities in generation prompt | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | Improve context integration in prompt | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Improve context integration in prompt | Open |
| F010 | irrelevant | Answer does not address the question — improve prompt clarity | Restate question entities in generation prompt | Open |
| F011 | hallucination | Evaluator misclassifies refusal as hallucination | Add Out-of-Scope refusal evaluator branch | Open |
| F012 | hallucination | Evaluator misclassifies refusal as hallucination | Add Out-of-Scope refusal evaluator branch | Open |
```

**Ba improvement suggestions ưu tiên**

1. **Thêm quy tắc Full-Sentence Framing & Restatement vào System Prompt của Generator.**
2. **Cập nhật Evaluator hỗ trợ Semantic Embedding Similarity (hoặc LLM Judge) thay cho word overlap thuần túy.**
3. **Bổ sung Out-of-Scope Refusal Evaluator Branch cho tập câu hỏi Adversarial.**

| Suggestion | Target metric | Verification method |
|---|---|---|
| Full-Sentence Framing Prompt | Answer Relevance & Completeness | Chạy lại benchmark trên tập Easy (E01-E05), mục tiêu Relevance ≥ 0.80. |
| Out-of-Scope Refusal Evaluator | Faithfulness & Pass Rate (Adversarial) | Re-eval tập Adversarial (A01-A03), mục tiêu Faithfulness = 1.0 cho câu từ chối chuẩn. |
| LLM-as-a-Judge Evaluation | Overall Pass Rate | So sánh chỉ số đồng thuận (Correlation) giữa LLM Judge score và Human Label. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy tự động trong CI/CD Pipeline tại bước Pull Request trước khi Merge code, mỗi khi có sự thay đổi về Prompt, thay đổi Embedding Model, chỉnh sửa Chunking strategy hoặc cập nhật dữ liệu tài liệu mới.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:* Phù hợp cho `Relevance` và `Completeness`. Tuy nhiên, với `Faithfulness` trong domain Student Services (tư vấn học phí, quy chế, điểm số), threshold drop 0.05 là quá lỏng lẻo. Cần đặt threshold giọt cho Faithfulness là **0.01** (không chấp nhận bất kỳ sự suy giảm nào về tính chính xác thực tế).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment:** `Faithfulness < 0.85`, bất kỳ vi phạm `Safety/Privacy` nào, hoặc sụt giảm `Faithfulness > 0.01`.
> - **Alert Only:** `Answer Relevance < 0.70`, `Completeness < 0.70`, hoặc sụt giảm `Context Precision > 0.05`.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Golden Benchmark (20-100 QA)] → [Regression Gate Check (Delta <= 0.05)] → [Staging Human Audit (Edge cases)] → Deploy
```

> *Giải thích:* Thay đổi trước tiên phải qua bước chấm điểm tự động Offline trên Golden Dataset. Nếu không bị Regression, chuyển qua Staging để chuyên gia kiểm thử ngẫu nhiên các trường hợp nhạy cảm trước khi chính thức đưa lên Production.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cập nhật Generator System Prompt (Full sentence framing) | Relevance & Completeness | Tăng Pass rate từ 40% lên 70%. |
| 2 | Tích hợp Refusal Evaluation Branch cho câu hỏi Out-of-scope | Faithfulness (Adversarial) | Giải quyết 100% sai sót nhầm lẫn ở nhóm Adversarial. |
| 3 | Thêm Reranker (Cross-Encoder / Cohere Rerank) | Context Precision | Tối ưu xếp hạng chunk cho nhóm câu hỏi phức tạp (Hard). |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Cross-policy Conflict Case:** Câu hỏi tra cứu quy định có sự thay đổi giữa Policy V1.0 (trước 01/08/2026) và Policy V2.0 (sau 01/08/2026) để kiểm tra khả năng bắt đúng mốc thời gian hiệu lực.
> 2. **Multi-condition Edge Case:** Câu hỏi yêu cầu thỏa mãn đồng thời 4 điều kiện khác nhau (GPA, số tín chỉ, lệ phí, chữ ký duyệt) để đo khả năng tổng hợp Completeness.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Trái với dự đoán ban đầu rằng Retriever sẽ là "nút thắt cổ chai" (bottleneck) của RAG, kết quả thực tế cho thấy **Retriever đạt hiệu năng gần như hoàn hảo** (`Precision = 0.974`, `Recall = 0.945`). Điểm yếu lớn nhất làm sụt giảm Pass Rate lại nằm ở **Generator Prompt** (trả lời quá ngắn) và **Hạn chế của thuật toán chấm điểm Lexical Overlap** khi đánh giá các câu trả lời ngắn và câu từ chối.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* 
> - **Giới hạn của Word-overlap heuristics:** Phụ thuộc hoàn toàn vào trùng khớp bề mặt từ ngữ (surface token matching). Khi bot đưa ra câu trả lời đúng bản chất nhưng dùng từ đồng nghĩa hoặc câu trả lời ngắn gọn (VD: "10 business days" vs "within ten business days"), chỉ số này sụt giảm nghiêm trọng.
> - **Nâng cấp cho Production:** 
>   1. Bổ sung **LLM-as-a-Judge (G-Eval)** với Rubric 1-5 được calibrate kỹ lưỡng theo đánh giá của con người.
>   2. Sử dụng **Semantic Embedding Distance (Cosine Similarity / BERTScore)** để đo độ tương đồng ý nghĩa thay cho word overlap.
>   3. Tách riêng pipeline đánh giá cho câu hỏi **Factual Retrieval** và câu hỏi **Out-of-Scope Refusal**.
