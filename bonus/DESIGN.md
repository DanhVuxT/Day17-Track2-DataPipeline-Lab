# Bonus Design - Data Pipeline cho Chatbot CSKH tiếng Việt

## Bài toán

Sản phẩm tôi chọn là một chatbot CSKH cho một sàn thương mại điện tử Việt Nam.
Chatbot phải trả lời các câu hỏi về đổi trả, bảo hành, trạng thái đơn hàng,
phí vận chuyển, khiếu nại và chính sách theo ngành hàng. Dữ liệu đầu vào khá
lộn xộn: file PDF chính sách do đội vận hành cập nhật, trang help center, bảng
đơn hàng từ OLTP, ticket CSKH, và trace hội thoại thật của chatbot. Người dùng
hỏi bằng tiếng Việt có dấu, không dấu, viết tắt, teencode, trộn tiếng Anh, và
nhiều câu chứa thông tin cá nhân như số điện thoại hoặc mã đơn hàng.

Ràng buộc chính là chatbot không được trả lời sai chính sách, không được lộ dữ
liệu cá nhân, và phải cải thiện từ chính traffic thật mà không tự đầu độc eval.
Độ trễ trả lời nên dưới 2 giây cho câu hỏi thường, còn câu cần kiểm tra đơn hàng
có thể chậm hơn một chút. Ngân sách cũng quan trọng: nếu mọi tài liệu đều gọi LLM
để trích xuất lại mỗi ngày thì chi phí sẽ tăng rất nhanh.

```text
PDF/help center/ticket logs        orders/returns DB          chatbot traces
          |                              |                         |
          v                              v                         v
   raw document lake              CDC/order snapshots        bronze spans
          |                              |                         |
          v                              v                         v
 chunk + embed store        point-in-time feature store   eval + DPO candidate
          |                              |                         |
          +-------------> retrieval gateway <----------------------+
                              |
                              v
                 answer with citations + policy checks
                              |
                              v
                      human review + monitoring
```

## 1. Nguồn và hình dạng dữ liệu

Quyết định của tôi là giữ cả raw zone và curated zone. Raw zone lưu nguyên PDF,
HTML, ticket và trace để có thể replay khi parser sai. Curated zone chứa chunk,
embedding, triple graph, bảng policy version và bảng trace đã flatten.

Tradeoff là raw-first tốn thêm storage so với chỉ lưu dữ liệu đã parse, nhưng
storage rẻ hơn rất nhiều so với mất khả năng audit. Với tài liệu CSKH, lỗi parse
có thể không làm pipeline chết ngay mà làm chatbot trả lời sai âm thầm. Vì vậy
khả năng replay và so sánh parser version quan trọng hơn tiết kiệm vài GB.

Schema sẽ không ổn định. Chính sách vận hành thường đổi tên cột, thêm exception,
hoặc chèn bảng nhiều cột trong PDF. Tôi sẽ validate ở mức contract tối thiểu:
mỗi tài liệu phải có `doc_id`, `source`, `version`, `effective_from`, `text`,
`content_hash`. Nếu thiếu các field này, tài liệu đi vào quarantine thay vì vào
index. Các field nghiệp vụ chi tiết sẽ được parse sau và có test riêng.

## 2. Batch hay streaming

Tôi chọn hybrid: batch cho tài liệu chính sách, streaming nhẹ cho trace và
feedback. Chính sách đổi trả hoặc bảo hành không cần cập nhật từng giây; chạy
batch mỗi 1-3 giờ là đủ. Ngược lại, trace chatbot và feedback xấu cần vào hệ
thống nhanh để phát hiện regression trong ngày.

Tradeoff ở đây là streaming toàn bộ nghe hiện đại hơn nhưng phức tạp, khó
backfill, và tốn công vận hành. Batch toàn bộ thì đơn giản nhưng phát hiện lỗi
quá chậm. Hybrid giữ chi phí thấp mà vẫn có tín hiệu nhanh từ production. Khi
trace lỗi tăng đột biến, hệ thống có thể cảnh báo ngay mà chưa cần rebuild toàn
bộ vector store.

## 3. Hợp đồng dữ liệu và chất lượng

Trước khi dữ liệu vào model, tôi sẽ validate ba lớp. Lớp một là cấu trúc: file
đọc được, có version, có ngày hiệu lực, không rỗng. Lớp hai là an toàn: PII như
số điện thoại, email, địa chỉ cụ thể được mask trong trace dùng cho training.
Lớp ba là chất lượng retrieval: chunk không quá ngắn, không quá dài, không bị
lặp boilerplate như footer hoặc menu.

Tradeoff là validate chặt có thể quarantine nhiều dữ liệu hợp lệ nhưng hơi xấu,
còn validate lỏng sẽ để rác vào index và làm chatbot kém đi. Tôi chọn chặt ở
biên giới trước model, nhưng không làm mất dữ liệu: dòng xấu vào quarantine kèm
lý do, owner, source và sample. Nếu tỷ lệ quarantine tăng hơn baseline, Slack
alert cho đội data và đội vận hành tài liệu.

## 4. Train/serve parity và point-in-time

Với câu hỏi có mã đơn hàng, feature nguy hiểm nhất là trạng thái mới nhất của
đơn. Nếu train một câu hỏi ngày 1 nhưng join trạng thái ngày 5, model sẽ học
tín hiệu tương lai. Tôi chọn point-in-time join theo `event_ts`: mọi feature
đơn hàng, lịch sử hoàn tiền, số lần khiếu nại chỉ được lấy tại hoặc trước thời
điểm người dùng hỏi.

Tradeoff là point-in-time store khó làm hơn bảng snapshot latest. Snapshot latest
dễ query và rẻ hơn, nhưng metric offline sẽ nói dối. Trong CSKH, lời nói dối đó
rất nguy hiểm: chatbot có vẻ dự đoán đúng trạng thái đơn trong eval, nhưng khi
serve thật nó không có thông tin tương lai. Vì vậy tôi chấp nhận chi phí lưu
snapshot theo thời gian và dùng ASOF join cho training set.

## 5. RAG hay Knowledge Graph

Tôi chọn kết hợp vector RAG và knowledge graph. Vector RAG phù hợp câu hỏi trực
tiếp như "gadget bảo hành bao lâu" hoặc "đổi trả trong mấy ngày". Knowledge graph
phù hợp câu hỏi nhiều bước như "sản phẩm thuộc ngành hàng nào và kho nào xử lý
đổi trả" vì thông tin có thể nằm ở hai tài liệu khác nhau.

Tradeoff là KG cho precision và truy vấn multi-hop tốt hơn, nhưng cần entity
resolution và bảo trì ontology. Vector store rẻ hơn để bắt đầu, nhưng dễ bỏ lỡ
quan hệ tách rời giữa chính sách, ngành hàng và kho. Tôi sẽ dùng vector làm
default retrieval, còn KG cho các entity ổn định như product category, warehouse,
policy type, warranty duration và return eligibility.

## 6. Flywheel từ trace production

Trace production sẽ được flatten thành Bronze span giống lab: mỗi lượt có prompt,
retrieval context, answer, latency, cost, outcome và human feedback nếu có.
Từ đó tạo eval set bằng các lượt đã được human xác nhận đúng, chia holdout theo
thời gian và theo intent. DPO pair lấy từ cùng một intent: câu trả lời được
duyệt là chosen, câu trả lời sai hoặc bị sửa là rejected.

Tradeoff lớn nhất là dùng nhiều trace để train nhanh so với giữ eval sạch. Tôi
chọn decontamination nghiêm: prompt hoặc paraphrase gần eval sẽ bị loại khỏi
training. Nếu không, dashboard sẽ đẹp lên nhưng sản phẩm thật không tốt hơn.
Với tiếng Việt, tôi sẽ normalize dấu, lowercase, bỏ khoảng trắng thừa và dùng
n-gram similarity để bắt cả câu hỏi không dấu hoặc viết lại nhẹ.

## Phương án bị loại

Tôi loại phương án "đưa toàn bộ tài liệu và toàn bộ ticket vào một vector store,
không versioning, không KG, không decontamination". Cách này nhanh để demo nhưng
không phù hợp production. Nó không biết chính sách nào đang có hiệu lực, khó
debug khi câu trả lời sai, dễ train trên eval, và không xử lý tốt câu hỏi
multi-hop. Nó cũng rủi ro với dữ liệu Việt Nam vì ticket thật chứa PII, lỗi gõ
không dấu, và nhiều biến thể tên sản phẩm.

## Vận hành và chi phí

80% chi phí dự kiến nằm ở embedding lại tài liệu và gọi LLM để trích xuất KG.
Để giảm chi phí, tôi dùng `content_hash` để chỉ re-embed chunk thay đổi. Với KG,
chỉ tài liệu chính sách chính thức mới gọi extractor đắt tiền; ticket và trace
chỉ dùng cho eval/fine-tune sau khi mask PII. Backfill phải idempotent: mỗi run
ghi theo `run_id`, `source_version`, `content_hash`, và upsert bằng khóa ổn định
để chạy lại không nhân đôi chunk hay DPO pair.

Thiết kế này ưu tiên correctness trước tốc độ phát triển. Lý do là chatbot CSKH
không chỉ cần trả lời nghe hay; nó cần trả lời đúng chính sách, đúng thời điểm,
có thể audit, và cải thiện thật sau mỗi vòng trace -> eval -> training.
