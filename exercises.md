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

| Metric            | Acceptable Low Score Scenario                                                                                                                                                                              | Critical Low Score Scenario                                                                                                                                                                                             | Action Required                                                                                                                                                     |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness      | Câu trả lời thêm một cảnh báo an toàn nhỏ, ghi rõ nguồn ("vui lòng xác nhận lại với bộ phận hỗ trợ") không có nguyên văn trong context nhưng không sai lệch bất kỳ fact nào. | Câu trả lời nêu sai thời hạn hoàn tiền, thời hạn bảo hành, hoặc giá/chính sách không được context hỗ trợ — một claim bị hallucinate mà khách hàng có thể hành động theo.               | Rà soát prompt sinh câu trả lời/yêu cầu grounding, giảm temperature, bắt buộc trích dẫn nguồn, và chặn deploy cho tới khi test lại các case fail. |
| Answer Relevance  | Câu hỏi thực sự mơ hồ và assistant hỏi lại để làm rõ thay vì đoán bừa — đây là hành vi đúng nhưng heuristic đo relevance theo keyword/embedding có thể chấm thấp.            | Câu trả lời lạc đề so với câu hỏi (vd. khách hỏi về bảo hành, câu trả lời lại nói về giao hàng) — cho thấy lỗi retrieval hoặc nhận diện intent.                                              | Rà soát cách hình thành query/xử lý intent, kiểm tra các chunk đã retrieve, cân nhắc viết lại query hoặc siết chặt system prompt.                 |
| Context Recall    | Câu hỏi adversarial/out-of-scope nên vốn không có gold evidence liên quan, hoặc gold answer chỉ cần một phần trong tập evidence dư thừa.                                                    | Evidence cần thiết (vd. điều khoản ngoại lệ hoặc ngày hiệu lực) bị thiếu trong các chunk đã retrieve ở câu hỏi Easy/Medium/Hard trong phạm vi hệ thống, khiến model phải đoán hoặc bỏ sót. | Kiểm tra retriever (BM25 index, top-k), tăng top-k, hoặc cải thiện cách chia chunk để evidence bị thiếu có thể được retrieve.                        |
| Context Precision | Top-k chứa một đoạn liền kề liên quan nhẹ, gây nhiễu nhưng không làm sai lệch generation; câu hỏi Hard hợp lý cần nhiều chunk nên rank bị trải rộng.                               | Chunk liên quan duy nhất bị chôn sau nhiều chunk không liên quan, khiến generator bỏ sót hoặc đánh giá sai trọng số — thường gặp ở case Hard/Adversarial.                                          | Tinh chỉnh retriever scoring/reranking, giảm kích thước hoặc overlap của chunk, hoặc thêm reranker để đẩy chunk liên quan lên trên.                 |
| Completeness      | Một chi tiết bổ sung không quan trọng bị thiếu trong khi mọi claim/điều kiện bắt buộc vẫn đầy đủ và chính xác.                                                                        | Thiếu một điều kiện, ngoại lệ, số tiền, hoặc quy tắc điều kiện quan trọng trong expected answer (vd. thiếu phí restocking hoặc điều kiện ngày hiệu lực chính sách).                           | Chỉnh lại prompt sinh câu trả lời để liệt kê rõ mọi điều kiện có trong context; thêm checklist/regression test cho claim bị bỏ sót.              |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy một tập N cặp câu trả lời (Answer A vs Answer B) cho cùng câu hỏi, đưa cho judge chấm pairwise preference.
>
> - Condition 1: trình bày (A trước, B sau).
> - Condition 2: cùng nội dung nhưng đảo thứ tự (B trước, A sau).
>
> Chạy judge trên cả hai conditions với cùng bộ cặp (50–100 cặp), rồi đo tỉ lệ judge đổi lựa chọn khi đảo vị trí (consistency rate). Nếu judge chọn "câu ở vị trí đầu" nhiều hơn đáng kể so với 50% (kiểm định binomial), đó là bằng chứng position bias. Có thể thêm Condition 3 với hai câu trả lời giống hệt nhau (duplicate) để đo baseline: nếu judge vẫn thiên vị "vị trí đầu" khi nội dung y hệt, đó là tín hiệu position bias thuần túy, không lẫn với chênh lệch chất lượng thật.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Thiết kế rubric chấm theo claim rời rạc (checklist các fact/condition bắt buộc phải có) thay vì đánh giá cảm tính toàn đoạn — độ dài không còn là tín hiệu để "ăn điểm". Ghi rõ trong hướng dẫn cho judge: nội dung dài thêm không mang thông tin bắt buộc thì không được cộng điểm, và câu trả lời dài dòng lặp lại thông tin phải bị trừ ở tiêu chí clarity/conciseness. Có thể định nghĩa độ dài kỳ vọng theo loại câu hỏi và phạt khi vượt quá nhiều mà không thêm claim mới. Khi calibrate, dùng cặp câu trả lời cùng nội dung nhưng độ dài khác nhau (ngắn vs dài, cùng đúng) để kiểm tra judge có chấm lệch theo độ dài không, nếu có thì tinh chỉnh lại rubric/prompt.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge có các bias hệ thống (position, verbosity, self-preference) và không nắm chính xác các quy định domain-specific của OrbitTech (số ngày bảo hành, điều kiện hoàn tiền...) như một SME thật. Human labels là ground truth để đo agreement (vd. Cohen's kappa, accuracy) giữa judge và người chấm, định lượng hướng và mức độ lệch của judge, từ đó chọn/hiệu chỉnh rubric hoặc prompt cho tới khi agreement đạt ngưỡng chấp nhận được trước khi tin dùng judge trong CI/CD. Nếu không calibrate, team có thể triển khai thay đổi dựa trên một judge sai lệch một cách hệ thống, khiến chất lượng thực tế suy giảm âm thầm mà benchmark tự động không phát hiện ra.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric           | Threshold | Lý do                                                                                                                                                                                                                                  |
| ---------------- | --------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness     |      0.85 | Hallucination trong customer support (sai chính sách hoàn tiền, sai số tiền/ngày bảo hành) gây rủi ro pháp lý và mất niềm tin khách hàng trực tiếp, nên phải strict, sát mức "Good".                            |
| Answer Relevance |      0.75 | Cần trả lời đúng ý khách hỏi một cách đáng tin cậy, nhưng vẫn cho phép sai lệch nhỏ về cách diễn đạt/phạm vi câu hỏi phụ.                                                                                    |
| Completeness     |      0.75 | Thiếu một điều kiện/exception quan trọng có thể gây hiểu nhầm, nhưng câu trả lời đúng cốt lõi và hướng dẫn liên hệ hỗ trợ thêm vẫn có thể chấp nhận được ở mức threshold thấp hơn Faithfulness. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> - **Offline evaluation** (golden dataset, chạy trong CI ở mỗi PR/thay đổi prompt hoặc model): dùng làm regression gate tự động, nhanh, trước khi merge/deploy; phát hiện các failure mode đã biết trước khi ảnh hưởng người dùng thật.
> - **Online evaluation** (theo dõi traffic thật sau khi deploy, sample câu hỏi thực tế, chấm bằng automated judge hoặc theo dõi metric liên tục): dùng sau khi deploy để phát hiện drift, failure pattern mới không có trong golden set, và thay đổi phân bố sử dụng thực tế; thường chạy async, có dashboard và alert threshold.
> - **Human review**: dùng định kỳ (vd. audit hàng tuần), cho các case có score sát ngưỡng/độ tin cậy thấp mà judge tự động gắn cờ, cho case adversarial/edge case, và bất kỳ câu trả lời nào liên quan tiền/pháp lý/an toàn nơi automated metric không đủ tin cậy; đồng thời dùng để calibrate lại LLM judge và bổ sung case mới vào golden dataset từ các failure thực tế.

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

| Hạng mục                         | Kết quả   |
| ---------------------------------- | ----------- |
| Tổng số records                  | ____ / 20   |
| Easy                               | ____ / 5    |
| Medium                             | ____ / 7    |
| Hard                               | ____ / 5    |
| Adversarial                        | ____ / 3    |
| Source documents được sử dụng | ____ / 10   |
| Validator status                   | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
| -- | ---------- | ------------------ | --------------------------------------------------- |
|    |            |                    |                                                     |
|    |            |                    |                                                     |
|    |            |                    |                                                     |

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

| ID  | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
| --- | ---------------- | ---------: | ------------: | -----------: | --------: | -----------: | ------: | ------- | ------------ |
| E01 |                  |            |               |              |           |              |         |         |              |
| E02 |                  |            |               |              |           |              |         |         |              |
| E03 |                  |            |               |              |           |              |         |         |              |
| E04 |                  |            |               |              |           |              |         |         |              |
| E05 |                  |            |               |              |           |              |         |         |              |
| M01 |                  |            |               |              |           |              |         |         |              |
| M02 |                  |            |               |              |           |              |         |         |              |
| M03 |                  |            |               |              |           |              |         |         |              |
| M04 |                  |            |               |              |           |              |         |         |              |
| M05 |                  |            |               |              |           |              |         |         |              |
| M06 |                  |            |               |              |           |              |         |         |              |
| M07 |                  |            |               |              |           |              |         |         |              |
| H01 |                  |            |               |              |           |              |         |         |              |
| H02 |                  |            |               |              |           |              |         |         |              |
| H03 |                  |            |               |              |           |              |         |         |              |
| H04 |                  |            |               |              |           |              |         |         |              |
| H05 |                  |            |               |              |           |              |         |         |              |
| A01 |                  |            |               |              |           |              |         |         |              |
| A02 |                  |            |               |              |           |              |         |         |              |
| A03 |                  |            |               |              |           |              |         |         |              |

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
| ----: | -------------------------- | ---------------- |
|     5 |                            |                  |
|     4 |                            |                  |
|     3 |                            |                  |
|     2 |                            |                  |
|     1 |                            |                  |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
| --------- | -------------------- | ------------------------- |
|           |                      |                           |
|           |                      |                           |
|           |                      |                           |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí                    | Framework 1: ____ | Framework 2: ____ |
| ----------------------------- | ----------------- | ----------------- |
| Setup complexity              |                   |                   |
| Metrics available             |                   |                   |
| CI/CD integration             |                   |                   |
| Kết quả trên cùng dataset |                   |                   |
| Insight rút ra               |                   |                   |

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

| ID            | Recall before | Recall after | Precision before | Precision after | Delta Precision |
| ------------- | ------------: | -----------: | ---------------: | --------------: | --------------: |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
| **Avg** |               |              |                  |                 |                 |

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
