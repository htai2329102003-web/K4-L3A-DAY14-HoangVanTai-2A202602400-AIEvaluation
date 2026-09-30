# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 75% (15/20 cases)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.852 | 0.588 | 1.000 | Retrieval thường đưa được evidence cần thiết; A01 là ngoại lệ out-of-scope nên recall thấp. |
| Context Precision | 0.925 | 0.478 | 1.000 | Context được lấy nhìn chung liên quan; H05 thấp nhất, dù top 5 vẫn có các chunk về warranty và an toàn, cho thấy cần kiểm tra nhiễu và trọng số context. |
| Faithfulness | 0.736 | 0.357 | 1.000 | Đây là metric answer-side thấp nhất trung bình; H03 và H05 có điểm thấp, cần đối chiếu câu trả lời với evidence. |
| Relevance | 0.785 | 0.500 | 1.000 | Các câu trả lời nhìn chung bám chủ đề; một số câu hard/adversarial vẫn bị giảm điểm. |
| Completeness | 0.692 | 0.429 | 0.950 | Đây là answer metric thấp nhất trung bình; các câu hỏi nhiều điều kiện dễ thiếu ý. |
| Overall Score | 0.738 | 0.506 | 0.950 | Trung bình chưa đạt 0.8; 7 case Good, 10 Needs Work, 3 Significant Issues. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Overall của 7/20 cases. Trung bình Context Recall (0.852) và Context Precision (0.925) cũng thuộc vùng này.
- Metrics/cases ở mức Needs Work (0.6–0.8): Overall của 10/20 cases; trung bình Faithfulness (0.736), Relevance (0.785), Completeness (0.692) và Overall (0.738).
- Metrics/cases ở mức Significant Issues (<0.6): Overall của 3/20 cases: H02, H05, H03. Min theo metric: Context Recall 0.588, Context Precision 0.478, Faithfulness 0.357, Relevance 0.500, Completeness 0.429.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 0 | 0% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 5 | 25% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề chính nằm ở generation và answer coverage, kèm một lỗi retrieval rõ ở H05. Trung bình Context Recall 0.852 và Context Precision 0.925 khá cao, trong khi Completeness chỉ 0.692 và Faithfulness 0.736. H02 có Recall 0.857/Precision 1.000 nhưng Completeness 0.429; H03 có Precision 1.000 nhưng Faithfulness 0.438 do câu trả lời suy diễn thêm về eligibility. H05 là ngoại lệ có retrieval yếu hơn (Recall 0.692, Precision 0.478) và thiếu chunk về phương án sửa chữa có tính phí.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> H02 — I bought AeroBuds Pro with OrbitPlus active and opened the ear tips 10 days ago. Does the member return extension make them returnable?

**Expected answer:**

> No. OrbitPlus extends only the unopened-device return window; it does not override hygiene exclusions. Opened ear tips and in-ear audio products are non-returnable unless defective. The return policy version and eligibility also depend on the order date.

**Actual answer:**

> Opened ear tips for the AeroBuds Pro are considered hygiene accessories and are non-returnable unless defective, regardless of the return policy version or OrbitPlus membership. Therefore, they cannot be returned.

**Scores:** Context Recall: 0.857 | Context Precision: 1.000 | Faithfulness: 0.591 |
Relevance: 0.500 | Completeness: 0.429 | Overall: 0.506

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Top retrieved chunks: OT-05-P02 (returns), OT-01-P03 (product), OT-09-P04 (policy version), OT-03-P05 (membership), OT-06-P01 (warranty). Context Recall 0.857 và Precision 1.000; cả hai gold contexts nằm trong top 5.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | H02 có Overall 0.506; completeness 0.429 và relevance 0.500, dù cả hai gold context có trong top 5. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời kết luận ear tips đã mở không được trả, nhưng chưa giải thích OrbitPlus chỉ kéo dài hạn trả cho hàng chưa mở và không ghi đè hygiene exclusion. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Câu hỏi có nhiều điều kiện nhưng đầu ra tập trung vào kết luận yes/no về hygiene, bỏ mất phần giải thích quyền lợi thành viên và điều kiện theo policy version/order date. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Generation chưa có checklist tách các ý hỏi thành các policy facts bắt buộc phải trả lời. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Metric phát hiện thiếu completeness sau khi sinh câu trả lời, nhưng không có bước coverage check trước khi trả kết quả hoặc regression case buộc trả lời từng điều kiện. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu cơ chế lập answer plan theo từng sub-question và kiểm tra đủ điều kiện/ngoại lệ; bổ sung checklist coverage và test cho câu hỏi kết hợp membership với hygiene exclusion. |

**Root cause từ `find_root_cause()`:**

> Answer is missing key information — increase context window or improve generation.

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Tôi đồng ý. OT-03-P05 nói OrbitPlus chỉ mở rộng unopened-device window và không override hygiene exclusion; OT-05-P02 xác nhận opened ear tips non-returnable unless defective. Hai gold chunks đều được retrieve (Recall 0.857, Precision 1.000), nhưng Completeness chỉ 0.429. Đây là thiếu coverage trong câu trả lời hơn là thiếu retrieval.

**Proposed fix cụ thể:**

> Tách câu hỏi thành ba ý: phạm vi gia hạn OrbitPlus, hygiene exclusion, và ngoại lệ khi sản phẩm defective. Sinh một câu cho mỗi ý, giữ điều kiện theo ngày đặt hàng khi liên quan, rồi chạy answer-coverage check trước khi trả lời. Thêm biến thể opened ear tips bị lỗi và membership theo hai policy version vào regression set.

### Failure 2

**ID và question:**

> H05 — My PulsePhone X was exposed to liquid and now will not charge. Is this covered by warranty, and what is the safe next step?

**Expected answer:**

> Liquid exposure is excluded from the limited warranty, though accidental damage may still be repairable for a fee. Do not charge or continue using a wet device: power it down when safe, disconnect it from charging, and do not open the device or bypass safety features.

**Actual answer:**

> Liquid exposure is excluded from the warranty for the PulsePhone X, so this issue is not covered. The safe next step is to power down the device when safe, disconnect it from charging, and avoid opening it or bypassing safety features. For further assistance, please contact OrbitTech support.

**Scores:** Context Recall: 0.692 | Context Precision: 0.478 | Faithfulness: 0.357 |
Relevance: 0.750 | Completeness: 0.500 | Overall: 0.536

**Evidence inspection:**

> Top retrieved chunks: OT-06-P01 (warranty term), OT-01-P02 (phone specifications), OT-06-P03 (liquid exclusion), OT-07-P01 (safe troubleshooting), OT-00-P05 (safety scope). Recall 0.692 và Precision 0.478. Có evidence cho liquid exclusion và safe next step, nhưng thiếu gold chunk OT-06-P04 về việc accidental damage có thể sửa tính phí; OT-01-P02 là context nhiễu với câu hỏi này.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | H05 có Overall 0.536; Context Precision 0.478, Faithfulness 0.357 và Completeness 0.500. Answer thiếu lựa chọn sửa tính phí. |
| Why 1 | Tại sao symptom xảy ra? | Top 5 có các chunks về liquid exclusion và safe handling nhưng không có gold chunk OT-06-P04 về accidental damage repairable for a fee; đồng thời có chunk thông số PulsePhone ít liên quan. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Truy xuất theo query ưu tiên thông tin gắn với tên thiết bị/charging và không lấy đủ các policy subtopic: warranty exclusion, repair option và an toàn khi thiết bị ướt. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retrieval không phân rã câu hỏi nhiều ý hoặc mở rộng query sang “accidental damage repair fee”; top-k chứa context nhiễu thay cho chunk về phí sửa chữa. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có assertion theo case yêu cầu cả gold facts về warranty, fee repair và safety phải có trong retrieved context; benchmark chỉ cho thấy điểm thấp sau lần chạy. |
| Why 5 | Root cause có thể hành động được là gì? | Retrieval chưa bảo đảm độ bao phủ các policy intents của câu hỏi. Thêm query expansion/reranking và kiểm tra presence của các evidence chunks quan trọng cho warranty, repair fee và safety. |

**Root cause và proposed fix:**

> find_root_cause(): Context is missing or irrelevant — improve retrieval.

### Failure 3

**ID và question:**

> H03 — My package arrived with visible shipping damage, but I only reported it 60 hours after delivery. What evidence and reporting deadline apply?

**Expected answer:**

> Visible shipping damage must be reported within 48 hours after confirmed delivery, so 60 hours is beyond the stated deadline. The policy asks you to keep the packaging and provide photographs of the label, box, and contents. A concealed defect follows the warranty or return policy instead.

**Actual answer:**

> You reported visible shipping damage 60 hours after delivery, which is beyond the 48-hour reporting deadline. You should retain the packaging and provide photographs of the label, box, and contents, but since the deadline has passed, you may not be eligible for a shipping damage claim. For further assistance, please contact OrbitTech support.

**Scores:** Context Recall: 0.806 | Context Precision: 1.000 | Faithfulness: 0.438 |
Relevance: 0.688 | Completeness: 0.548 | Overall: 0.558

**Evidence inspection:**

> Top retrieved chunks: OT-04-P04 (shipping damage deadline and evidence), OT-05-P03 (return requirements), OT-04-P03 (tracking delays), OT-08-P03 (account fraud), OT-07-P04 (repair quote). The gold shipping evidence is first and Context Precision is 1.000; Faithfulness is 0.438, so inspect the answer's inference about eligibility against the exact policy wording.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | H03 có Overall 0.558 và Faithfulness 0.438. Câu trả lời thêm khả năng khách “may not be eligible” dù policy chunk chỉ nêu deadline và evidence cần cung cấp. |
| Why 1 | Tại sao symptom xảy ra? | Answer biến việc báo trễ hơn 48 giờ thành khả năng không đủ điều kiện claim, một hệ quả không được khẳng định trong OT-04-P04. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generation suy rộng từ deadline sang eligibility thay vì giới hạn kết luận ở điều policy nêu rõ; phần hướng dẫn giữ bao bì và ảnh là phù hợp. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Quy trình sinh câu trả lời chưa buộc kiểm tra từng claim chính sách với evidence tương ứng hoặc nêu rõ khi corpus không xác định hệ quả. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Gold evidence được retrieve đầu tiên (Precision 1.000), nhưng không có claim-entailment check trước khi trả lời. Heuristic faithfulness chỉ đánh giá sau khi sinh, không tự chặn suy luận. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu grounded-claim guard: chỉ nêu hệ quả được tài liệu hỗ trợ; nếu corpus chỉ quy định deadline thì không tự kết luận claim chắc chắn bị chấp nhận hoặc từ chối. |

**Root cause và proposed fix:**

> `find_root_cause()` trả “Multiple issues detected — review full pipeline” vì Faithfulness thấp nhất nhưng retrieval scores đều trên 0.5. Trace cho thấy retrieval đúng; lỗi cụ thể là generation thêm suy luận về eligibility. Ràng buộc mọi claim về quyền lợi vào OT-04-P04 và trả lời rằng policy nêu hạn 48 giờ cùng yêu cầu ảnh/bao bì, không khẳng định chắc kết quả xét claim.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Thiếu coverage check cho câu hỏi nhiều điều kiện; bỏ sót policy condition hoặc ngoại lệ trong answer. | H02, M03, A03 | High |
| 2 | Retrieval top-k có nhiễu và thiếu một policy chunk quan trọng cho query nhiều ý. | H05 | Medium |
| 3 | Không kiểm tra claim entailment nên generation suy diễn hệ quả vượt quá evidence. | H03 | High |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Tôi ưu tiên Cluster 3 vì suy diễn về eligibility có thể khiến khách quyết định dựa trên một điều policy không xác nhận. H03 có Context Precision 1.000, nên thêm retrieval không giải quyết lỗi claim grounding. Cluster 1 có nhiều case hơn và là ưu tiên tiếp theo để cải thiện completeness.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer is missing key information — increase context window or improve generation | Add a grounding check that rejects claims unsupported by retrieved context | Open |
| F002 | off_topic | Answer is missing key information — increase context window or improve generation | Retrieve broader evidence and require the answer to cover each key reference point | Open |
| F003 | off_topic | Context is missing or irrelevant — improve retrieval | Add intent and scope checks before generating an answer | Open |
| F004 | off_topic | Context is missing or irrelevant — improve retrieval | Investigate and address the identified root cause | Open |
| F005 | off_topic | Answer is missing key information — increase context window or improve generation | Investigate and address the identified root cause | Open |
```

**Ba improvement suggestions ưu tiên**

1. Bắt buộc lập coverage checklist theo từng phần câu hỏi và policy condition để tăng answer completeness, nhất là H02/M03/A03.
2. Mở rộng query/rerank các policy chunks theo từng intent; ưu tiên đưa evidence về repair fee vào top-k cho H05.
3. Kiểm tra entailment cho từng claim về eligibility, warranty và safety; không thêm hệ quả mà corpus không nêu như H03.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Coverage checklist cho answer | Completeness, Relevance, pass rate | Chạy full golden benchmark; kiểm tra từng điều kiện trong H02/M03/A03 đã được nêu đúng; so sánh score theo case và aggregate với baseline. |
| Query expansion/reranking | Context Recall và Context Precision, đặc biệt H05 | Xác nhận OT-06-P04 xuất hiện trong top-k H05; tính lại retrieval metrics và bảo đảm các case khác không giảm quá 0.05. |
| Claim-entailment guard | Faithfulness và độ chính xác policy/safety | Đối chiếu từng claim với chunk nguồn; chạy H03 cùng câu hỏi biên về deadline/eligibility và review thủ công các claim chưa được support. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy trong CI sau mọi thay đổi model, prompt, retrieval, chunking/index, preprocessing hoặc policy corpus, trước khi deploy. So sánh output mới với baseline cố định trên golden dataset và các critical safety/privacy cases. Sau deploy, chạy lại khi đổi cấu hình production và theo dõi canary để phát hiện drift.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> 0.05 là ngưỡng khởi đầu hữu ích để phát hiện regression, nhưng không đủ làm gate duy nhất. Benchmark chỉ có 20 câu và aggregate average có thể che lỗi safety/policy ở một case; thêm absolute per-case thresholds, slice theo difficulty/domain và hard block cho unsupported claims. Vì heuristic/judge có nhiễu, dùng cùng versioned dataset và lặp lại/kiểm tra thủ công các biến động sát ngưỡng.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block nếu có lộ dữ liệu, prompt-injection thành công, hướng dẫn nguy hiểm hoặc claim về policy/safety không có evidence; block nếu bất kỳ mandatory case nào có answer metric dưới 0.5 hoặc aggregate regression vượt 0.05 so với baseline. Alert và điều tra khi non-critical average giảm nhẹ, retrieval metric suy giảm nhưng chưa mất evidence cho case quan trọng, hoặc user feedback có xu hướng xấu. Benchmark hiện tại có H02/H03/H05 fail theo metric threshold, nên quality gate chưa cho phép deploy trước khi sửa và rerun.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit/integration tests] → [Golden-set benchmark] → [Regression + safety quality gate] → Deploy
```

> Chạy test suite để xác nhận implementation, benchmark 20 QA để đo answer và retrieval metrics, rồi so với baseline và kiểm tra critical cases trước khi deploy. Lần chạy hiện tại có 42 tests passed và validator PASS, nhưng benchmark còn 5/20 failures nên kiểm tra unit pass chưa đủ để thông qua quality gate.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm coverage checklist/answer planning cho câu hỏi nhiều điều kiện. | Completeness, Relevance, pass rate | H02/M03/A03 trả lời đủ điều kiện; giảm failure do thiếu policy branch. |
| 2 | Query expansion + reranking theo policy intent; bổ sung assertion cho gold evidence. | Context Recall/Precision, đặc biệt H05 | Lấy được OT-06-P04 về sửa tính phí và giảm context nhiễu. |
| 3 | Grounding check cho claim về eligibility, warranty và safety. | Faithfulness, critical-case pass rate | Loại suy diễn như “may not be eligible” ở H03 khi corpus chỉ nêu reporting deadline. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Thêm (1) H02 biến thể ear tips bị lỗi để kiểm tra hygiene exception cùng OrbitPlus; (2) H05 biến thể hỏi rõ sửa chữa ngoài warranty có tính phí không và bước an toàn khi thiết bị ướt; (3) H03 biến thể báo trong/ngoài 48 giờ để kiểm tra assistant không khẳng định kết quả claim nếu policy chỉ mô tả deadline. Các case cần gold chunks riêng và expected answer phân biệt rõ fact với điều corpus không xác nhận.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Kết quả cho thấy lỗi thấp điểm không chỉ đến từ retrieval. H02 có Context Recall 0.857 và Precision 1.000 nhưng Completeness chỉ 0.429; H03 có Precision 1.000 nhưng Faithfulness 0.438 do thêm suy luận về eligibility. Ngược lại, H05 cho thấy retrieval vẫn là vấn đề ở một số query: Precision chỉ 0.478 và thiếu evidence về sửa chữa có tính phí. Vì vậy cần xử lý cả answer coverage/grounding lẫn retrieval theo từng case.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> Word overlap dễ bỏ sót paraphrase/đồng nghĩa, phạt câu trả lời đúng nghĩa nhưng dùng từ khác, và không kiểm tra entailment hay tính hợp lệ của suy luận. Điểm cũng nhạy với cách viết expected answer và độ dài chunk; một claim nguy hiểm có thể bị che bởi average cao. Production nên kết hợp rubric LLM-as-a-Judge đã hiệu chuẩn với human review trên critical cases, groundedness/claim-entailment có evidence, RAGAS Faithfulness/Answer Relevancy/Context Precision/Recall, cùng safety/privacy checks và per-case thresholds. Theo dõi disagreement giữa judge và người chấm để cập nhật rubric/dataset.
