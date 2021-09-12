---
author: telion
title: "Bài 4: Cách thiết kế  Instagram"
description: Chúng ta sẽ cùng nhau thiết kế 1 dịch vụ chia sẻ photo giống như
  Instagram, nơi mà người dùng users có thể tải lên photos và chia sẻ chúng với
  những người khác
series:
  - Grokking system design interviews
ShowToc: true
ShowBreadCrumbs: true
---
## 1. Instagram là gì?

Instagram là 1 dịch vụ mạng xã hội cho phép người dùng users upload và chia sẻ các hình ảnh photos và videos của họ với những người khác. Người dùng Instagram có thể lựa chọn chia sẻ thông thông qua cả 2 hình thức công khai publicly và riêng tư privately. Bất cứ thứ gì được chia sẻ công khai có thể được nhìn thấy bởi người dùng khác, mặt khác những nội dung chia sẻ riêng tư sẽ chỉ được truy cập bởi 1 nhóm người dùng cụ thể. Instagram cho phép người dùng nó chia sẻ thông qua nhiều nền tảng mạng xã hội khác, giống như Facebook, Twitter, Flickr, và Tumblr.

Chúng ta sẽ lên kế hoạch thiết kế 1 phiên bản đơn giản hơn của Instagram vì mục đích thiết kế, nơi mà người dùng user có thể chia sẻ photos và follow những người khác. "News Feed" cho từng user sẽ chỉ bao gồm các top photos của tất cả gười dùng khác mà user đang follows.

## 2. Yêu cầu Requirements và mục tiêu Goals của hệ thống

Chúng ta sẽ tập trung vào các tập yêu cầu như sau khi thực hiện thiết kế  Instagram.

**Các yêu cầu về mặt tính năng:**

1. Users có thể upload/download/view photos.
2. Users có thể thực hiện các tìm kiếm trên các tiêu đề photo/video.
3. Users có thể theo dõi follow người khác.
4. Hệ thống sẽ tạo và hiện thị News Feed của user - Chứa các top photos của tất cả các người dùng mà họ theo dõi.

**Yêu cầu không liên quan tới tính năng:**

1. Hệ thống cần có tính sẵn sàng cao.
2. Độ trễ có thể chấp nhận được của hệ thống là 200ms cho việc tạo ra News Feed.
3. Hệ thống chấp nhận tính nhất quán có thể ảnh hưởng nếu người dùng không nhìn thấy 1 photo trong 1 khoảng thời gian.
4. Hệ thống có độ tin cậy cao, bất cứ photo hay video nào đã được upload thì sẽ không bao giờ bị mất.

**Không nằm trong scope:** Việc theo tags vào photos, tìm kiếm photos dựa trên tag, bình luận vào photos, tagging người dùng user vào photos, etc.

## 3. 1 vài chú ý cần xem xét đến khi thực hiện thiết kế

Hệ thống sẽ nặng tải về read, do vậy ta sẽ tập trung vào xây dựng 1 hệ thống sao cho việc lấy ra các photos 1 cách nhanh nhất.

1. Thực tế, users có thể upload bao nhiêu photos tùy thích, do vậy việc quản lý lưu trữ storage 1 cách hiệu quả sẽ là 1 nhân tố đóng vai trò quan trọng trong thiết kế hệ thống.
2. Độ trễ thấp như kỳ vọng khi thực hiện viewing photos.
3. Data cũng cần 100% tin cậy. Nếu 1 user upload 1 photo thì hệ thống sẽ đảm bảo nó sẽ không bao giờ bị mất.

## 4. Định lượng sức chứa Capacity và các ràng buộc Constraints

- Ta giả sử chúng ra có tổng số  500M users, với 1M user hoạt động hàng ngày.
- 2M photos mới được tạo ra mỗi ngày, 23 new photos mỗi giây.
- Trung bình kích thước 1 photo => 200KB.
- Tổng số  lưu trữ bắt buộc cho tất cả photos trong 1 ngày là: ```2M * 200KB => 400 GB```
- Tổng số  lưu trữ bắt buộc cho tất cả photos trong 10 năm là: ```400GB *365 (days a year)* 10 (years) ~= 1425TB```

## 5. High Level System Design