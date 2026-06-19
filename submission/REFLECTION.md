# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

1. Lỗi âm thầm nhất là flatten trace hoặc gán nhãn `split` sai: nếu rơi mất một span con hoặc đánh nhầm `split='eval'`, dữ liệu vẫn sinh ra được nhưng sẽ bị rò rỉ train/eval. Mình sẽ phát hiện bằng kiểm tra số dòng theo `trace_id`, đối chiếu độ phủ trace, và audit mẫu Bronze so với JSON trace gốc.

2. Bỏ decontamination sẽ khiến mô hình học luôn cả prompt mà sau đó mình dùng để chấm. Metric offline nhìn cao hơn thực tế, nhưng lên dữ liệu mới thì không giữ được vì mô hình chỉ học thuộc mẫu trả lời thay vì tổng quát hoá.

3. Một feature nguy hiểm là tổng chi tiêu, số dư, hoặc điểm rủi ro gian lận của khách hàng. Nếu không có point-in-time guard, dòng train có thể lấy giá trị cập nhật sau thời điểm dự đoán, làm accuracy offline cao giả tạo.

4. Graph trả lời tốt các câu hỏi nhiều bước như “kho nào giao phụ kiện của widget?”. Còn truy hồi vector là đủ cho việc tra cứu fact đơn giản, nên dùng graph ở đó là quá tay.
