---
author: "telion"
title: "Các bước để trả lời được phỏng vấn trong buổi System Design Interviews"
description: "Các bước thực hiện để phân tích thiết kế hệ thống"
tags: ["system design interviews"]
categories: ["System Design"]
series: ["Grokking system design interviews"]
ShowToc: true
ShowBreadCrumbs: true
---

Rất nhiều kỹ sư phần mềm cảm thấy quá sức trong các cuộc phỏng vấn system design interviews (SDIs) bởi vì 3 lys do chính sau:

- Do các câu hỏi thường không có cấu trúc chặt chễ  trong các buổi phỏng vấn SDIs, khi các câu hỏi đa phần là hỏi về giải pháp thiết kế để giải quyết các vấn đề từ đầu tới cuối và thường không bó hẹp bởi 1 câu trả lời nào đó.
- Việc thiếu kinh nghiệm của chính bản thân họ trong việc phát triển xây dựng hệ thống large scale systems.
- Họ thường thiếu chuẩn bị cho các cuộc phỏng vấn SDIs.

Không giống như các cuộc phỏng vấn coding interviews, các ứng viên thường không có chủ định chuẩn bị cho các cuộc phỏng vấn SDIs, đa số họ thường xử lý không tốt trong các cuộc phỏng vấn này đặc biệt ở các công ty như Google, Facebook, Amazon, Microsoft, etc. Ở những công ty này, các ứng viên thường xử lý dưới mức trung bình để đạt được cơ hội hiếm có là nhận được 1 offer. Ngược lại, những ứng viên xử lý tốt trong cuộc phỏng vấn sẽ luôn nhận được kết qủa tốt với mức offer tốt hơn (vị trí và mức thu nhập cao hơn) do cuộc phỏng vấn sẽ chỉ ra khả năng của ứng viên trong việc xây dựng 1 hệ thống phức tạp.

Trong bài học này, chúng ta sẽ tuân theo các bước tiếp cận sau để giải quyết các vấn đề về thiết kế. Bước đầu tiên, ta cần đi qua các bước sau:

## Bước 1: làm rõ các yêu cầu Requirements

Luôn là 1 ý tưởng tốt khi ta đặt câu hỏi về chính xác phạm vi scope của vấn đề mà ta đang giải quyết. Các câu hỏi về thiết kế thường là các câu hỏi giải quyết 1 vấn đề từ đầu tới cuối, và chúng thường không chỉ có 1 câu trả lời đúng. Đây là lý do tại sao việc cần làm rõ các yêu cầu mơ hồ từ sớm trong buổi phỏng vấn là điều hết sức quan trọng. Các ứng viên nên dành đủ thời gian để định nghĩa ra mục tiêu cuối của hệ thống sẽ có cơ hội tố t hơn để thành công trong cuộc phỏng vấn. Do chúng ta chỉ có tầm 35-40 phút để thiết kế 1 hệ thống large system, nên chúng ta phải làm rõ những thành phần cấu thành nên hệ thống mà ta đang tập trung vào thiết kế.

Chúng ta sẽ đi sâu vào vấn đề này với 1 ví dụ cụ thể là thiết kế 1 Twitter-like service. Đây là 1 vài câu hỏi dành cho việc thiết kế  Twitter cần được trả lời trước khi ta chuyển sang bước tiếp theo:

- Người dùng users của service có khả năng post tweets và theo dõi follow những người khác?
- Hệ thống có tạo và hiện thị ra timeline của user?
- tweets có chứa photos và videos?
- Chỉ tập trung thiết kế backend, có cần thiết kế front-end?
- Người dùng users có thể tìm kiếm tweets?
- Hệ thống có hiện thị ra các hot trending topics?
- Có tính năng push notification cho các tweets mới hay là quan trọng?

Tất cả câu hỏi này sẽ định hình các ta thiết kế hệ thống.

## Bước 2: Ước lượng/định lượng hệ thống

Cũng luôn là 1 ý tưởng tốt là việc định lượng tính mở rộng của hệ thống mà chúng ta đang thiết kế. Điều này sẽ giúp chúng ta tập chung vào việc mở rộng scaling, phân tán partitioning, cân bằng tải load balancing, caching:

- Tính mở rộng mà hệ thống kỳ vọng (ví dụ: số  lượng các new tweets, số lượt view tweet views, số lượng tạo timeline generations trên mỗi giây)?
- Số lượng lưu trữ storage mà hệ thống cần? Chúng ta có các yêu cầu về lưu trữ storage requirements khác nhau nếu người dùng users có thể có photos và videos trong tweets của họ.
- Lưu lượng băng thông network bandwidth mà hệ thống kỳ vọng? Đây sẽ là điểm quan trọng trong việc ta quyết định quản lý traffic và cân bằng tải balance load giữa các servers.

## Bước 3: Các định nghĩa System interface definition

Định nghĩa những APIs sẽ có trong hệ thống. Điều này sẽ xác lập ra chính xác các chức năng hệ thống và đảm bảo rằng chúng ta không làm sai bất kỳ yêu cầu nào của hệ thống. 1 vài ví dụ về APIs dành cho Twitter-like service như sau:

```bash
postTweet(user_id, tweet_data, tweet_location, user_location, timestamp, …)  
```

```bash
generateTimeline(user_id, current_time, user_location, …) 
```

```bash
markTweetFavorite(user_id, tweet_id, timestamp, …)  
```

## Bước 4: Định nghĩa data model

Việc định nghĩa data model cũng thuộc các bước đầu tiên trong cuộc phỏng vấn, cũng cần làm rõ các luồng data flow giữa các thành phần khác nhau của hệ thống. Sau khi làm rõ bước này, thì nó sẽ định hướng cho ta cách quản lý và phân tán dữ liệu. Các ứng viên nên định nghĩa rõ ra các thành phần hệ thống khác nhau, cách chúng tương tác với thành phần khác, và các khía cạnh khác nhau của việc quản lý dữ liệu như lưu trữ storage, luân chuyển transportation, mã hóa encryption. Đây là 1 vài thành phần khác nhau dành cho Twitter-like service:

- User: UserID, Name, Email, DoB, CreationDate, LastLogin, etc.
- Tweet: TweetID, Content, TweetLocation, NumberOfLikes, TimeStamp, etc.
- UserFollow: UserID1, UserID2
- FavoriteTweets: UserID, TweetID, TimeStamp

Loại database nào mà chúng ta nên sử dụng? Có phải NoSQL như Cassandra phù hợp với yêu cầu mà ta cần, hay ta sử dụng 1 giải pháp dựa trên MYSQL? Loại lưu trữ block storage nào mà ta nên sử dụng để lưu trữ photos và videos?

## Bước 5: High-level design

Thực hiện vẽ ra các biểu đồ diagram để biểu diễn ra các thành phần core của hệ thống. Chúng ta nên xác định đủ các thành phần components cần có để giải quyết các vấn đề thực sự end to end.

Đối với Twitter, ở mức high level, ta sẽ cần nhiều application servers để phục vụ các read/write requests với cân bằng tải load balancers ở trước chúng để phân phối traffic. Giả sử rằng chúng ta sẽ có nhiều read traffic hơn so với write, ta có thể quyết định phân tách servers riêng rẽ xử lý trong ngữ cảnh này. Ở phía back-end, ta cần có 1 database hiệu quả có thể  lưu trữ toàn bộ tweets và hỗ trợ được lượng lớn large number of reads. Ta cũng sẽ cần 1 hệ thống lưu trữ file phân tán distributed file storage system cho việc lưu trữ photos và videos.

![High-level design diagram](/grokking-system-design-interviews/6070836998438912.svg "High-level design diagram")

## Bước 6: Detailed design

Phân tích sâu hơn vào 2 hay 3 thành phần chính của hệ thống; dựa vào phản hồi của người phỏng vấn phải luôn hướng dẫn chúng ta đến những phần nào của hệ thống cần được thảo luận thêm. Chúng ta nên trình bày các hướng tiếp cận khác nhau, đưa ra ưu và nhược điểm của chúng, và giải thích tại sao ta lại lựa chọn cách tiếp cận này hơn cái khác. Nhớ rằng, không có 1 câu trả lời đúng duy nhất, luôn có 1 thứ quan trọng phải xét đến là sự cân bằng tradeoffs giữa các lựa chọn khác nhau liên quan tới các ràng buộc của hệ thống:

- Do chúng ta sẽ lưu trữ 1 lượng lớn dữ liệu, nên ta sẽ thực hiện phân tán dữ liệu phân tám ra nhiều databases? ta nên lưu trữ toàn bộ dữ liệu của 1 user trên cùng 1 database? Vấn đề issue gì có thể xảy ra?
- Cách ta xử lý những hot users khi họ tweet rất nhiều hay follow rất nhiều người khác?
- Do timeline của user sẽ chứa các tweets gần đây, ta có nên thử lưu trữ dữ liệu này để chúng tối ưu hóa cho việc lấy ra latest tweets?
- Bao nhiêu và những layer nào mà chúng ta dùng để cache để tăng tốc?
- Những components nào cần được load balancing?

## Bước 7: Cách xác định và giải quyết các nút cổ chai bottlenecks

Thực hiện đề cập tới nhiều các nút cổ chai bottlenecks nhiều nhất có thể và các cách tiếp cận khác nhau để giảm thiểu chúng:

- Hệ thống có tồn tại điểm single point of failure nào không? Chúng ta sẽ làm gì để gỉam thiểu được chúng?
- Chúng ta có đủ số replicas của dữ liệu và hệ thống vẫn chạy đáp ứng người dùng nếu có 1 vài servers gặp lỗi?
- Tương tự, chúng ta có đủ số copies của các services khác nhau đang chạy mà 1 vài trong số chúng gặp lỗi thì không gây ra shutdown toàn bộ hệ thống?
- Cách chúng ta giám sát hiệu năng của các services? Chúng ta đã có những cảnh báo bất cứ khi nào các hệ thống quan trọng gặp lỗi hay gặp vấn đề về hiệu năng?

## Tổng kết

Ngắn gọn, việc chuẩn bị có chủ định khi thực hiện phỏng vấn sẽ là chìa khóa để  giúp bạn thành công trong buổi phỏng vấn system design interviews. Các bước ở trên sẽ hướng dẫn bạn đi đúng hướng và bao quát được các cách tiếp cận khác nhau khi thiết kế hệ thống.
