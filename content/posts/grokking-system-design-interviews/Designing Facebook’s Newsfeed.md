---
author: "telion"
title: "Bài 13: Cách thiết kế  Facebook Newsfeed"
description: "Chúng ta sẽ cùng nhau thiết kế  Facebook Newsfeed"
tags: ["system design interviews","Designing Facebook’s Newsfeed","Designing Twitter Newsfeed","Designing Instagram Newsfeed","Designing Quora Newsfeed"]
categories: ["System Design"]
series: ["Grokking system design interviews"]
ShowToc: true
ShowBreadCrumbs: true
---

## 1. Facebook newsfeed là gì?

1 Newsfeed là 1 danh sách các bài stories được cập nhật liên tục nằm chính giữa trang Facebook’s homepage. Nó bao gồm các cập nhật liên quan tới trạng thái status, photos, videos, links, app activity hay các likes từ những người, trang pages, hay groups mà người dùng user theo dõi follow trên Facebook.

Với bất cứ hệ thống mạng xã hội nào mà bạn thiết kế như Twitter, Instagram, hay Facebook thì bạn sẽ cần 1 hệ thống kiểu Newsfeed để hiện thị các thông tin cập nhật từ những người bạn và những người mà bạn theo dõi.

## 2. Yêu cầu Requirements và mục tiêu Goals của hệ thống

Chúng ta sẽ thiết kế 1 hệ thống newsfeed cho Facebook theo các yêu cầu sau:
**Yêu cầu về  mặt tính năng:**

1. Newsfeed cần được tạo ra dựa trên các bài posts từ những người, trang pages, groups mà người dùng user theo dõi follow.
2. 1 người dùng user có thể có rất nhiều bạn và theo dõi follow 1 lượng lớn các pages/groups.
3. Feeds có thể chứ images, videos, hay chỉ chứa text.
4. Hệ thống của chúng ta cần hỗ trợ thêm mới các bài posts mới nhanh nhất có thể khi chúng vừa mới xuất hiện cho tất cả các users đang hoạt động.

**Yêu cầu không liên quan tới tính năng:**

1. Hệ thống có khả năng tạo ra newsfeed cho bất cứ user nào 1 cách real-time và độ trễ tối đa là 2s.
2. 1 bài post không cần tới 5s để nó xuất hiện trong feed của 1 user giả định là bài post vừa được đăng.

## 3. Ước lượng trữ lượng và các ràng buộc

Ta giả định 1 user trung bình sẽ có 300 bạn và theo dõi 200 pages.

**Ước lượng về traffic:** Giả sử ta có 300M người dùng hoạt động hàng ngày và trung bình mỗi người dùng sẽ làm mới timeline của họ 5 lần 1 ngày. Kết quả là sẽ có 1.5B newsfeed request 1 ngày và xấp xỉ 17,500 request mỗi giây.

**Ước lượng về storage:** Ở mức trung bình, ta giả định chúng ta có khoảng 500 posts trên mọi newsfeed của mỗi user và chúng ta sẽ giữ chúng trong bộ nhớ memory để lấy chúng ra 1 cách nhanh nhất. Chúng ta cũng giả định rằng trung bình mỗi bài post có kích thước khoảng 1KB. Điều này có nghĩa rằng chúng ta cần lưu trữ khoảng 500KB dữ liệu data trên mỗi user. Để lưu trữ được tất cả data cho tất cả các users đang active chúng ta cần 150TB memory. Nếu 1 server có thể đáp ứng được 100GB thì chúng ta cần khoảng 1500 servers để giữ được 500 bài posts mới nhất trong memory cho tất cả các users đang active.

## 4. System APIs

> Khi ta đã chốt được các yêu cầu requirements, thì ý tưởng tốt là định nghĩa các system APIs. Điều này sẽ phản ánh đúng những gì mà hệ thống kì vọng.

Chúng ta cần có SOAP hay REST APIs để expose ra các tính năng của các service. Đoạn ví dụ dưới dùng để định nghĩa API dùng để  lấy ra newsfeed:

```bash
getUserFeed(api_dev_key, user_id, since_id, count, max_id, exclude_replies)
```

**Các tham số:**

- **api_dev_key (string):** Api developer key cần phải được đăng ký để sử dụng và dùng để  throttle users dựa trên hạn mức quota mà họ được cấp.
- **user_id (number):** ID của user mà hệ thống dùng để tạo newsfeed.
- **since_id (number):** Không bắt buộc, trả về kết quả với 1 ID cao hơn (mới hơn) ID cho trước.
- **count (number):** Không bắt buộc, định nghĩa số lượng các feed items cần lấy ra, giới hạn max là 200.
- **max_id (number):** Không bắt buộc, trả về kết quả với 1 ID nhỏ hơn hoặc bằng (cũ hơn) ID cho trước.
- **exclude_replies(boolean):** Không bắt buộc, Tham số này sẽ ngăn chặn việc trả về các replies xuất hiện trong kết quả timeline được trả về.

**Kết quả trả về:** (JSON) trả về 1 JSON object chứa 1 danh sách các feed items.

## 5. Database Design

Ta sẽ có 3 đối tượng objects chính: User, Entity (e.g. page, group, etc.), and FeedItem (or Post). Dưới đây là 1 vài quan sát về quan hệ relationships giữa các đối tượng này:

- 1 user có thể follow các đối tượng entities khác và có thể là bạn với users khác.
- Cả users và các đối tượng entities có thể  post FeedItems chứa text, images hay videos.
- Mỗi FeedItem sẽ có 1 UserID dùng để xác định User tạo ra nó. Cho đơn giản, ta giả định chỉ có users mới tạo được feed items mặc dù Facebook Pages cũng có thể tạo ra.
- Mỗi FeedItem có thể có 1 EntityID dùng để  trỏ tới trang page hay group mà bài posts được tạo ra.

Nếu ta đang sử dụng 1 relational database, chúng ta sẽ cần mô hình hóa 2 mối quan hệ relations: User-Entity relation và FeedItem-Media relation. Do mỗi user có thể là bạn với nhiều người khác và theo dõi follow rất nhiều đối tượng entities, ta có thể lưu trữ mối quan hệ relation ra thành bảng table riêng. cột "Type" trong bảng "UserFollow" dùng để xác định đối tượng entity đang được follow là 1 User hay Entity. Tương tự ta cũng cần có 1 bảng table cho FeedMedia relation.

![Database Design](/grokking-system-design-interviews/4506227120275456.svg "Database Design")

## 6. High Level System Design

Ở mức high level, vấn đề của chúng ta chia làm 2 phần:

**Feed generation:** Newsfeed được tạo từ các bài posts (hay feed items) từ những users hay entities (pages hay groups) mà 1 user theo dõi. Do đó, bất cứ khi nào hệ thống nhận được yêu cầu tạo feed cho 1 user (ví dụ là Jane), thì chúng ta sẽ thực hiện các bước sau:

1. Lấy về IDs của tất cả các users và entities mà Jane theo dõi.
2. Lấy về các bài posts mới nhất, phổ biến nhất, phù hợp nhất cho những IDs này. Đây là posts tiềm năng mà ta có thể hiện thị chúng trong newsfeed của Jane.
3. Đánh giá các bài posts này dựa trên sự phù hợp với Jane. Điều này phản ánh trực tiếp với feed của Jane hiện tại.
4. Lưu trữ feed này vào trong cache và trả về top các bài posts (tầm 20) để hiện thị trong Jane feed.
5. Ở phía front-end, khi Jane chạm tới cuối feed hiện tại của cô ấy, thì cô ấy có thể lấy về tiếp 20 posts tiếp theo từ server và cứ tiếp tục như vậy.

**Feed publishing:** Bất cứ khi nào Jane load trang newsfeed page của cô ấy, cô ấy phải thực hiện request và pull các feed items từ server. Bất cứ khi nào cô ấy chạm vào đáy của feed hiện tại, thì cô ấy có thể pull data mới từ server. Với những phần tử items mới hơn thì server có thể thông báo cho Jane và cô ấy có thể pull hay server có thể push những posts mới này xuống. Chúng ta sẽ bàn về các lựa chọn này sau.

Ở mức high level, chúng ta chỉ cần các components sau trong Newsfeed service:

1. **Web servers:** Cần duy trì 1 connection với user. Connection này có thể được sử dụng để chuyển dữ liệu data giữa user và server.
2. **Application server:** Thực hiện xử lý các luồng nghiệp vụ workflows dùng để lưu trữ các posts mới trong database servers. Chúng ta sẽ cần 1 vài application servers để lấy về và để push newsfeed xuống end user.
3. **Metadata database and cache:** Dùng để lưu trữ các thông tin metadata liên quan tới Users, Pages, Groups.
4. **Posts database and cache:** Dùng để lưu trữ các thông tin metadata liên quan tới posts và các nội dung contents của họ.
5. **Video and photo storage, and cache:** Dạng blob storage, dùng để lưu trữ tất cả các media có trong bài posts.
6. **Newsfeed generation service:** Dùng để thu thập và xếp hạng tất cả các posts phù hợp với 1 user để tạo ra newsfeed và lưu nó trong cache. Service này cũng sẽ nhận các cập nhật mới nhất và sẽ thêm các feed items mới này vào timeline của user bất kỳ.
7. **Feed notification service:**  Thông báo cho user các feed items mới đang có sẵn cho newsfeed của họ.

Dưới đây là biểu đồ kiến trúc ở mức high-level, User B và User C đang theo dõi User A.

![Facebook Newsfeed Architecture](/grokking-system-design-interviews/6485520234840064.svg "Facebook Newsfeed Architecture")

## 7. Detailed Component Design

Ta sẽ bàn chi tiết về 2 components khác nhau.

**a. Feed generation**
Giờ ta xét 1 ví dụ đơn giản là newsfeed generation service sẽ lấy về các posts mới nhất từ tất cả các users và entities mà Jane theo dõi; câu truy vấn sẽ giống như sau:

```sql
(SELECT FeedItemID FROM FeedItem WHERE UserID in (
    SELECT EntityOrFriendID FROM UserFollow WHERE UserID = <current_user_id> and type = 0(user))
)
UNION
(SELECT FeedItemID FROM FeedItem WHERE EntityID in (
    SELECT EntityOrFriendID FROM UserFollow WHERE UserID = <current_user_id> and type = 1(entity))
)
ORDER BY CreationDate DESC 
LIMIT 100
```

Các issues xảy ra với cách thiết kế này dành cho newsfeed generation service:

1. Sẽ rất chậm đối với các users có nhiều bạn friends hay người theo dõi follows do ta phải thực hiện sorting/merging/ranking trên 1 lượng lớn các posts.
2. Chúng ta tạo ra timeline khi 1 user thực hiện load trang page của họ. Điều này sẽ rất chậm và có độ trễ cao.
3. Với những live updates, mỗi status update sẽ được cập nhật vào trong các feed updates của tất cả các followers. Điều này gây ra sự tồn đọng high backlogs trong Newsfeed Generation Service.
4. Với những live updates, server push (hay thông báo) các posts mới tới những users có thể gây ra vấn đề nặng tải, đặc biệt là đối với những người, hay pages có lượng lớn followers. Để cải thiện hiệu năng, ta có thể tạo trước timeline và lưu nó trong 1 memory.

**Offline generation for newsfeed:** Chúng ta cần có các servers dành riêng cho việc liên tục tạo ra các newsfeed của users và lưu trữ chúng trong memory. Do đó bất cứ khi nào 1 user request lấy về các posts mới cho feed của họ, chúng ta có thể đơn giản lấy các feed này từ nơi lưu trữ đã tạo trước. Bằng cách sử dụng cơ chế này, thì newsfeed của user không cần phải tạo khi có yêu cầu, và do đó có thể trả về cho users bất cứ khi nào có yêu cầu.

Bất cứ khi các servers này cần tạo feed cho 1 user, chúng sẽ truy vấn để lấy về thời điểm tạo feed gần nhất được tạo với user này đầu tiên. Sau đó, các feed data mới sẽ được tạo từ thời điểm tạo này trở về sau. Chúng ta có thể lưu trữ thông tin data này trong 1 hash table, với key là UserID và value là 1 cấu trúc STRUCT giống như sau:

```java
Struct {
    LinkedHashMap<FeedItemID, FeedItem> feedItems;
    DateTime lastGenerated;
}
```

Chúng ta có thể lưu trữ FeedItemIDs trong 1 cấu trúc data structure giống như Linked HashMap hay TreeMap, chúng có thể cho phép ta không chỉ nhảy tới bất cứ feed item nào mà còn có thể lặp qua chúng thông qua map 1 cách dễ dàng. Bất cứ khi nào users muốn lấy về nhiều feed items hơn, chúng ta có thể gửi last FeedItemID mà họ đang nhìn thấy trong newsfeed của họ, và sau đó ta có thể nhảy tới FeedItemID đó trong hash-map và trả về batch/page của feed items tiếp theo từ đó.

**Bao nhiêu feed items mà ta nên lưu trữ trong memory cho feed của 1 user?** Thời điểm khởi đầu, chúng ta có thể quyết định lưu 500 feed items cho mỗi user, nhưng con số này có thể được điều chỉnh sau đó dựa trên mức sử dụng. Ví dụ, nếu ta giả định 1 page của 1 user feed sẽ có 20 bài posts và đa số users không bao giờ duyệt trên 10 pages feed của họ, chúng ta có thể quyết định chỉ lưu trữ 200 posts trên mỗi user. Với bất cứ user người mà muốn xem nhiều bài posts hơn (nhiều hơn những gì lưu trong memory), chúng ta có thể truy vấn chúng từ backend servers.

**Chúng ta nên tạo (hay giữ trong memory) newsfeed cho tất cả users?** Có rất nhiều users không đăng nhập 1 cách thường xuyên. Đây là 1 vài thứ mà ta có thể làm để xử lý điều này; 1) 1 cách tiếp cận hiển nhiên nhất là sử dụng 1 LRU cache để xóa users khỏi memory do họ không truy cập newsfeed của họ trong 1 khoảng thời gian dài. 2) 1 cách tiếp cận thông minh hơn là tìm ra kiểu đăng nhập của users để tạo ra các newsfeed của họ trước, ví dụ thời điểm nào trong ngày 1 user hoạt động và những ngày nào trong tuần mà user có thể truy cập newsfeed của họ?