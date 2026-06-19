# Bonus Design: Flywheel dữ liệu cho chatbot CSKH tiếng Việt

Tôi chọn bài toán xây pipeline dữ liệu cho một chatbot chăm sóc khách hàng tiếng Việt trong lĩnh vực thương mại điện tử và ví điện tử. Người dùng hỏi bằng tiếng Việt có dấu, đôi khi viết tắt, dùng ảnh chụp màn hình, hoặc gửi các câu hỏi rất thực dụng như tra cứu đơn hàng, chính sách hoàn tiền, phí vận chuyển, hoặc trạng thái khiếu nại. Bài toán khó không nằm ở việc “có LLM”, mà ở chỗ dữ liệu đầu vào rất bẩn và luồng dữ liệu phải khép kín: câu hỏi thật của khách hàng, trace của agent, phản hồi của người kiểm duyệt, và các mẫu lỗi cần biến thành dữ liệu huấn luyện mà không rò rỉ sang eval.

Ràng buộc đầu tiên là ngôn ngữ và chất lượng dữ liệu. Tiếng Việt có dấu làm tăng lỗi chuẩn hoá: “hoàn tiền”, “hoan tien”, “hòan tien” có thể là cùng một ý nhưng khác chuỗi. Ngoài ra, chat CSKH thường chứa PII như số điện thoại, mã đơn, email, địa chỉ. Vì vậy pipeline không thể chỉ lưu raw text; phải có bước che PII, chuẩn hoá Unicode, và phân loại intent trước khi dữ liệu đi vào kho huấn luyện. Ràng buộc thứ hai là độ trễ: các câu trả lời cho khách hàng nên gần real-time, nhưng dữ liệu để huấn luyện và đánh giá có thể batch theo ngày. Tôi không chọn streaming end-to-end vì chi phí và độ phức tạp cao hơn nhu cầu thật của phần training data.

Tôi sẽ chốt 5 câu hỏi then chốt.

1. Nguồn và hình dạng dữ liệu: dữ liệu đến từ đâu, có ổn định không? Tôi chọn mô hình “batch cho training, near-real-time cho serving”. Từ hệ thống CSKH, mỗi cuộc hội thoại sinh ra trace và event log; từ CRM có trạng thái đơn hàng, từ hệ thống ticket có kết quả xử lý. Schema không ổn định hoàn toàn, vì các trường payload thường thêm mới theo sản phẩm. Do đó cần Bronze append-only lưu raw event, rồi chuẩn hoá sang Silver với contract rõ ràng. Tôi ưu tiên schema có version thay vì schema cứng tuyệt đối, vì thay đổi nghiệp vụ là điều chắc chắn sẽ xảy ra.

2. Batch hay streaming? Tôi chọn batch hàng ngày cho dataset huấn luyện và eval, nhưng serving dùng truy vấn gần thời gian thực trên kho trạng thái hiện tại. Lý do là phần lớn giá trị nằm ở dữ liệu để fine-tune và đánh giá, không phải ở việc update mô hình từng giây. Nếu làm streaming toàn bộ, chúng ta phải trả giá bằng vận hành Kafka, watermark, backpressure, và debug khó hơn nhiều. Chỉ những tín hiệu thật sự thời gian nhạy cảm như “đơn đang ở trạng thái nào” mới cần đọc online.

3. Contracts và quality: dữ liệu xấu đi đâu? Tôi sẽ tách thành quarantine riêng cho các hội thoại lỗi, thiếu PII mask, hoặc không map được sang intent. Dòng xấu không được xoá âm thầm, vì nó là tín hiệu quan trọng để cải thiện upstream. Khi quarantine tăng vọt, cảnh báo phải đi cho đội vận hành bot và đội tích hợp CRM, không chỉ cho data team. Tôi ưu tiên fail-open ở luồng serving nhưng fail-closed ở luồng training, nghĩa là khách vẫn được phục vụ nhưng dữ liệu bẩn không được đưa vào train.

4. Train/serve parity: future leak nằm ở đâu? Rủi ro lớn nhất là dùng nhãn hậu kiểm hoặc trạng thái ticket sau khi agent đã trả lời để tạo feature training. Ví dụ, nếu một ticket được gán “resolved” sau 2 giờ mà ta lấy nhãn đó làm feature ở thời điểm trả lời đầu tiên, mô hình sẽ học tương lai. Tôi sẽ dùng point-in-time join cho mọi feature từ CRM và ticket history, tương tự ASOF join trong lab. Feature store phải lưu `valid_from`, không chỉ lưu giá trị cuối cùng.

5. Flywheel: làm sao biến trace thành dữ liệu train mà không tự đầu độc mình? Tôi sẽ tách thành 3 tập: eval holdout được curated thủ công, preference pairs từ các lượt trả lời tốt/xấu, và một tập decontaminated để train. Mọi prompt xuất hiện trong eval phải bị loại khỏi train. Nếu không làm vậy, metric offline sẽ đẹp giả tạo vì mô hình đã thấy cùng một câu hỏi trong cả train lẫn eval. Tôi chọn decontamination chính xác theo prompt chuẩn hoá trước; sau đó, nếu sản phẩm lớn hơn, mới mở rộng sang fuzzy matching hoặc n-gram overlap.

Một phương án tôi loại bỏ là “chỉ dùng vector RAG, không cần trace/data flywheel”. Cách đó rẻ hơn lúc đầu, nhưng không giải quyết được việc bot dần dần học sai hành vi từ log thật. Với CSKH, điều quan trọng không chỉ là tìm đúng tài liệu, mà còn là học được cách trả lời, cách từ chối, và cách escalate. Vector RAG chỉ trả lời câu hỏi tại thời điểm hiện tại; nó không tạo ra vòng lặp cải tiến dài hạn.

Kiến trúc tôi đề xuất:

```text
CRM/Ticket/Chat Logs
        |
        v
Bronze raw events  ---> PII mask + schema check ---> Quarantine
        |
        v
Silver normalized conversations
        |
        +--> Eval set (holdout curated)
        |
        +--> Preference pairs (good vs bad answers)
        |
        +--> Point-in-time features from CRM/Ticket history
        |
        v
Training datasets (decontaminated)
        |
        v
Model fine-tune + offline eval
        |
        v
Serving bot + new traces -> back to Bronze
```

Tôi cũng ưu tiên bối cảnh Việt Nam trong thiết kế. Vì tiếng Việt có dấu và nhiều biến thể gõ sai, bước chuẩn hoá phải giữ cả bản gốc lẫn bản canonical để không mất ý định. Vì băng thông và hạ tầng thường không “rộng tay” như các công ty ở Mỹ, tôi chọn batch định kỳ cho phần dữ liệu huấn luyện thay vì streaming end-to-end. Và vì dữ liệu CSKH có thể chứa thông tin cá nhân, pipeline phải coi masking và audit log là yêu cầu lõi, không phải tiện ích phụ.

Nếu phải tối giản để triển khai sớm, tôi sẽ cắt bớt graph/KG và chỉ giữ trace flywheel + decontamination + point-in-time features. Ba phần đó đủ để tạo ra giá trị thật và cũng là ba chỗ dễ sai nhất.
