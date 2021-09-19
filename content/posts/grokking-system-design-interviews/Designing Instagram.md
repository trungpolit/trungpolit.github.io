---
author: "telion"
title: "Bài 4: Cách thiết kế  Instagram"
description: "Chúng ta sẽ cùng nhau thiết kế 1 dịch vụ chia sẻ photo giống như Instagram, nơi mà người dùng users có thể tải lên photos và chia sẻ chúng với những người khác"
tags: ["system design interviews","Design photo-sharing service","Design Instagram"]
categories: ["System Design"]
series: ["Grokking system design interviews"]
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

Ở mức high-level, chúng ta cần support 2 ngữ cảnh, 1 là thực hiện upload photos, 2 là khi thực hiện view/search photos. Do đó dịch vụ chúng ta thiết kế sẽ cần các object storage servers để lưu tữ photos và database servers để lưu trữ các thông tin metadata của photos.

![High-level design diagram](/grokking-system-design-interviews/5506715794014208.svg "High-level design diagram")

## 6. Database Schema

> Việc định nghiã DB schema ở những giai đoạn đầu tiên của cuộc phỏng vấn sẽ giúp mọi người hiểu về luồng data flow giữa các thành phần components khác nhau trong hệ thống và giúp định hướng cách phân vùng dữ liệu data partitioning sau này.

Chúng ta cần lưu trữ dữ liệu về users, các photos được upload lên bởi họ, những người họ đang theo dõi follow. bảng Photo sẽ chứa tất cả các dữ liệu liên quan tới ảnh photo; chúng ta cần đánh index trên cặp (PhotoID, CreationDate) do ta cần lấy ra những photo được upload lên gần đây trước tiên.

![Database Schema](/grokking-system-design-interviews/4843868239953920.svg "Database Schema")

1 cách tiếp cận hết sức hiển nhiên cho việc lưu trữ giống như database schema ở trên là sử dụng 1 RDBMS như MySQL do ta có sử dụng đến joins. Nhưng databases quan hệ cũng đi cùng những thách thức của chúng, đặc biệt là khi ta cần mở rộng scale chúng.

Chúng ta có thể lưu trữ photos ở những hệ thống lưu trữ phân tán file distributed file storage giống như HDFS hay S3.

Chúng ta có thể lưu trữ dữ liệu ở schema trên dưới dạng lưu trữ phân tán distributed key-value để tận dựng được các ưu điểm của NoSQL. Tất cả các thông tin metadata liên quan tới photos lưu trữ trong 1 table mà có key là 'PhotoID' và value sẽ la 1 object chứa các thông tin PhotoLocation, UserLocation, CreationTimestamp, etc.

Nếu chúng ta chọn giải pháp sử dụng 1 NoSQL database, chúng ta cần thêm 1 table nữa để lưu trữ quan hệ relationships giữa users và photos để biết được ai đang sở hữu photo nào. Chúng ta sẽ gọi bảng này là 'UserPhoto'. Chúng ta cũng cần lưu trữ danh sách các người mà 1 user đang theo dõi follow. Chúng ta gọi bảng này là 'UserFollow'. Với những bảng table dạng này, ta có thể sử dụng 1 kiểu wide-column datastore giống như Cassandra. Ví với bảng UserPhoto, với key là UserID và value là danh sách các PhotoIDs mà user sở hữu, được lưu trữ ở các cột columns khác nhau. Chúng ta sẽ có schema tương tự cho bảng UserFollow.

Cassandra hay các kiểu key-value stores, thông thường luôn duy trì 1 số lượng các bản sao replicas để đảm bảo tính sẵn sàng của hệ thống. Và cũng ở trong những data stores dạng này, việc xóa deletes sẽ không được áp dụng ngay lập tức; dữ liệu data sẽ vẫn còn được lưu trữ trong vài ngày (hỗ trợ trường hợp muốn undeleting) trước khi bị xóa vĩnh viễn khỏi hệ thống.

## 7. Ước lượng Data Size

Chúng ta sẽ ước lượng cách mà dữ liệu data sẽ gia tăng ở từng bảng table và tổng số lưu trữ storage mà ta cần lưu trữ trong 10 năm.

**User:** Giả sử rằng mỗi kiểu dữ liệu 'int' và 'dataTime' cần 4 bytes để lưu trữ, mỗi dòng row trong bảng User sẽ cần 68 bytes:

```bash
UserID (4 bytes) + Name (20 bytes) + Email (32 bytes) + DateOfBirth (4 bytes) + CreationDate (4 bytes) + LastLogin (4 bytes) = 68 bytes
```

Nếu chúng ta có 500M users, thì ta sẽ cần 32GB để lưu trữ:

```bash
500 million * 68 ~= 32GB
```

**Photo:** Mỗi row trong bảng Photo sẽ cần 284 bytes:

```bash
PhotoID (4 bytes) + UserID (4 bytes) + PhotoPath (256 bytes) + PhotoLatitude (4 bytes) + PhotoLongitude(4 bytes) + UserLatitude (4 bytes) + UserLongitude (4 bytes) + CreationDate (4 bytes) = 284 bytes
```

Nếu có 2M photos mới được upload mỗi ngày, chúng ta sẽ cần 0.5GB lưu trữ cho mỗi ngày:

```bash
2M * 284 bytes ~= 0.5GB per day
```

Với 10 năm, ta sẽ cần 1.88TB lưu trữ.

**UserFollow:** Mỗi row trong UserFollow sẽ cần 8 bytes. Nếu ta có 500M users và trung bình mỗi user sẽ thực hiện theo dõi follow 500 người khác. Chúng ta sẽ cần 1.82TB lưu trữ cho UserFollow:

```bash
500 million users * 500 followers * 8 bytes ~= 1.82TB
```

Tổng dung lượng cần để lưu trữ tất cả các tables trong vòng 10 năm sẽ là 3.7TB:

```bash
32GB + 1.88TB + 1.82TB ~= 3.7TB
```

## 8. Component Design

Việc photo được upload (hay write) có thể trở nên chậm nếu chúng được ghi trực tiếp vào ổ disk, trong khi việc đọc reads sẽ nhanh hơn, đặc biệt nếu chúng được đọc từ cache.

Những người dùng thực hiện upload có thể sử dụng hết các connections hiện có của hệ thống, do việc thực hiện uploading được xét là 1 slow process. Điều này có nghĩa là reads có thể bị ảnh hưởng nếu hệ thống bị quá tải với các write request. Chúng ta cũng cần lưu ý rằng 1 web server luôn có 1 ngưỡng giới hạn connection limit. Ta giả sử rằng web server có 1 ngưỡng giới hạn là 500 connections tại 1 thời điểm, thì nó không thể có hơn 500 uploads hay reads cùng lúc. Để khắc phục nghẽn cổ chai này, ta có thể phân chia reads và writes vào các serivces độc lập. Chúng ta sẽ có các servers dành riêng cho việc reads và các server khác dành riêng cho việc writes để đảm bảo việc upload không ảnh hưởng tới cả hệ thống.

Việc tách biệt read và write requests cũng cho phép ta mở rộng và tối ưu từng loại xử lý 1 cách độc lập:

![Component Design](/grokking-system-design-interviews/4843868239953920.svg "Component Design")

## 9. Độ tin cậy Reliability và tính dư thừa Redundancy

Việc để mất mát các files không được phép xảy ra ở trong hệ thống. Do đó, ta cần lưu trữ nhiều bản copies của từng file và do đó nếu storage server dies thì ta vẫn có thể lấy ra được photo từ các bản copy của nó ở những storage server khác.

Định luật này cũng sẽ được áp dụng cho các thành phần components khác trong hệ thống. Nếu ta muốn hệ thống có tính sẵn sàng cao, chúng ta cần có nhiều bản sao chép replicas của các services đang chạy trong hệ thống do nếu có 1 vài services die thì hệ thống vẫn chạy và đảm bảo sẵn sàng. Tính dư thừa Redundancy sẽ xóa bỏ điểm yếu single point of failure của hệ thống.

Việc tạo ra dư thừa redundancy trong 1 hệ thống có thể xóa bỏ rủi ro single points of failure và cung cấp khả năng backup hay dự phòng nếu cần khi hệ thống gặp rủi ro. Ví dụ, nếu có 2 instances của cùng 1 service đang chạy trên production và khi 1 cái fail, thì hệ thống có thể chuyển đổi dự phòng failover tới bản copy khỏe mạnh. Failover có thể xảy ra 1 cách tự động và không đòi hỏi bất kỳ sự can thiệp thủ công nào.

![Reliability and Redundancy](/grokking-system-design-interviews/5399792583180288.svg "Reliability and Redundancy")

## 10. Data Sharding

Giờ chúng ta sẽ bàng tới các kiểu schemes khác nhau phục vụ cho metadata sharding:

**a. Partitioning dựa vào UserID** Ta gỉa sử rằng chúng ta sẽ shard dựa trên 'UserID' do đó ta có thể giữ mọi photos của 1 user nằm trong cùng 1 shard. Nếu 1 DB shard là 1 TB, chúng ta sẽ cần 4 shards để lưu trữ 3.7TB dữ liệu. Chúng ta sẽ giả sử rằng, vì lí do performace và scalability tốt hơn, ta sẽ cần 10 shards.

Do vậy ta sẽ tính được shard number bằng cách lấy số dư UserID chia cho 10 và sẽ lưu trữ dữ liệu data trên shard number đó. Để xác định được tính unique của photo trong hệ thống, ta cần thêm thông tin shard number vào sau mỗi PhotoID.

**Cách mà ta sẽ tạo ra PhotoIDs?** Mỗi DB shard sẽ có riêng cho mình 1 chuỗi tự tăng auto-increment sequence cho PhotoIDs, và do đó ta sẽ thêm ShardID vào sau mỗi PhotoID, ta sẽ khiến nó unique trên toàn hệ thống.

**Những vấn đề  sẽ phát sinh với kiểu phân tán partitioning scheme này?**

1. Cách mà ta sẽ xử lý đối với hot users? rất nhiều người theo dõi follow những hot users này, và cũng rất nhiều người khác xem những photo mà những user này upload lên.
2. 1 vài users sẽ có rất nhiều photos so với những người khác, do vậy nó sẽ tạo ra sự phấn tán lưu trữ không đồng đều.
3. Chúng ta phải làm gì nếu ta không lưu trữ được tất cả các ảnh của 1 người dùng trên cùng 1 shard? Nếu ta phân tán photos của 1 user ra trên nhiều shards, thì nó có gây ra độ trễ  lớn?
4. Việc lưu trữ tất cả các photos của 1 user trên cùng 1 shard có thể gây ra các vấn đề như độ sẵn sàng của tất cả các dữ liệu của user sẽ không truy cập được nếu shard down và độ trễ cao nếu shard nó đang cao tải.

**b. Partitioning dựa vào PhotoID** Nếu ta có thể tạo ra unique PhotoIDs trước và sau đó xác định shard number dựa vào “PhotoID % 10”, các vấn đề đặt ra ở trên sẽ được giải quyết. Ta cũng không cần thêm thông tin ShardID vào sau PhotoID trong trường hợp này, do PhotoID bản thân nó đã unique trên toàn bộ hệ thống.

**Cách mà ta tạo ra PhotoIDs?** theo cách này, ta không cần 1 chuỗi tự tăng auto-incrementing sequence ở mỗi shard để định nghĩa PhotoID bởi vì ta cần biết về PhotoID trước để xác định ra shard number mà ta dùng để lưu trữ dữ liệu. 1 giải pháp khác là ta có thể danh riêng 1 database instance để tạo ra các IDs tự tăng auto-incrementing IDs. Nếu PhotoID có thể chứa đủ trong 64 bits, ta có thể định nghĩa 1 table chỉ chứa 64 bit ID field. Do vậy bất cứ khi nào ta thêm 1 photo vào hệ thống, ta có thể thêm mới 1 row mới vào table này và lấy giá trị ID đó làm PhotoID của 1 photo mới.

**hệ thống key generating DB này có phải là 1 single point of failure?** Đúng vậy. 1 cách giải quyết cho vấn đề này là ta có thể sử dụng 2 databases, 1 cái chỉ tạo ra các IDs lẻ, và cái còn lại tạo ra IDs chẵn. Ví dụ với MySQL, ta có thể sử dụng scripts sau để định nghĩa chuỗi sequences:

```bash
KeyGeneratingServer1:
auto-increment-increment = 2
auto-increment-offset = 1

KeyGeneratingServer2:
auto-increment-increment = 2
auto-increment-offset = 2
```

Ta có thể đặt 1 cân bằng tải load balancer ở trước những databases này để cân bằng tải round-robin giữa chúng và cũng để giải quyết vấn đề  downtime. Khi cả 2 servers bị rơi khỏi sự đồng bộ này, thì sẽ naỷ sinh ra trường hợp server này sẽ tạo ra nhiều keys hơn so với cái còn lại, nhưng điều này không gây ra vấn đề gì cho hệ thống. Chúng ta có thể mở rộng cách thiết kế này bằng cách định nghĩa ra các bảng ID dành riêng cho Users, Photo-Comments, hay bất cứ các đố tượng object nào khác mà có trong hệ thống.

**Cách chúng ta hoạch định cho việc phát triển của hệ thống trong tương lai?** Chúng ta có thể có 1 lượng lớn số  phân vùng dữ liệu logical partitions để đáp ứng cho sự phát triển của dữ liệu trong tương lai, giống như thời điểm bắt đầu, có thể có nhiều multiple logical partitions nằm trên cùng 1 database server vật lý. Do mỗi database server có thể có nhiều database instances chạy trên nó, chúng ta có thể  có nhiều databases tách biệt nhau trên cùng 1 phân vùng logical partition ở bất cứ server nào. Do đó, bất cứ khi nào ta cảm thấy 1 database server nào đó có nhiều dữ liệu, ta có thể  migrate 1 vài phân vùng logical partitions từ nó sang server khác. Ta có thể  dựa trên 1 config file (hay config theo từng database riêng biệt) để map phân vùng logical partitions của ta vào các database servers; điều này cho phép ta di chuyển các phân vùng partitions 1 cách dễ dàng. Bất cứ khi nào ta muốn di chuyển 1 phân vùng partition, ta chỉ phải updat lại config file này để  tạo ra sự thay đổi.

## 11. Cách tạo hệ thống Ranking và News Feed

Để tạo ra News Feed cho bất cứ user nào đó, ta cần lấy ra các photos mới nhất, phổ biến nhất, và photos của những người mà user đó đang theo dõi follow.

Để đơn giản, ta có thể giả định rằng ta cần lấy ra 100 photos cho 1 News Feed của 1 user. application server của ta cần lấy ra 1 danh sách các người mà user đang theo dõi và lấy ra thông tin metadata info của 100 photos mới nhất của những người này. Bước cuối, server sẽ submit toàn bộ những photos này tới thuật toán ranking của ta, nó sẽ xác định top 100 photos (dựa trên thời điểm xuất hiện, số lượng like) và trả về chúng cho người dùng. 1 vấn đề ta phải đối mặt là độ trễ cao do ta phải truy vấn nhiều tables và thực hiện sorting/merging/ranking trên tập kết quả. Để cải thiện hiệu năng, ta có thể tạo trước sẵn các News Feed và chứa chúng trong 1 bảng table dành riêng.

**Cách tạo trước News Feed:** Chúng ta có thể dành riêng các servers để liên tục tạo ra các News Feeds của users và sắp xếp chúng trong bảng UserNewsFeed. Do vậy, bất cứ khi nào user cần các photo mới nhất trong News-Feed, chúng ta sẽ đơn giản truy vấn vào bảng table này để trả về kết quả đó cho user.

Bất cứ khi nào các servers này cần tạo ra News Feed cho 1 user, chúng sẽ truy vấn trong bảng UserNewsFeed trước và tìm ra thời điểm tạo gần nhất của News Feed, dựa vào đó News-Feed mới sẽ được tạo từ thời điểm đó trở lại đây.

**Có những cách nào để gửi các News Feed mới cho các users?**

1. **Pull:** Clients có thể pull các News-Feed từ server theo 1 khoảng thời gian định kỳ hay thủ công bất cứ khi nào họ cần. Các vấn đề phát sinh với các tiếp cận này: Dữ liệu mới có thể không được hiện thị ra cho users cho tới khi client thực hiện pull request, Đa số thời gian, pull requests sẽ trả về kết quả rỗng nếu chưa có dữ liệu mới.
2. **Push:** Server có thể push dữ liệu mới xuống users sớm nhất khi chúng vừa xuất hiện. Để quản lý việc này hiệu quả, users phải duy trì 1 Long Poll request tới server để nhận về updates. Vấn đề phát sinh với cách tiếp cận này: 1 user có thể theo dõi rất nhiều người hay 1 user nổi tiếng có thể có hàng triệu người theo dõi; trong trường hợp này, server phải push updates 1 cách liên tục.
3. **Hybrid:** Chúng ta có thể sử dụng 1 cách lai hybrid. Chúng ta có thể chuyển tất cả các users mà có số lượng lớn người theo dõi thành kiểu pull-based model và chỉ push data với những user chỉ có 1 vài trăm hay 1 vài ngàn người theo dõi. Cách tiếp cận khắc là có thể để server push updates tới tất cả users mà không hoạt động quá thường xuyên và cho phép các user với nhiều updates pull data 1 cách thường xuyên.

## 12. Cách tạo News Feed với các Sharded Data

1 trong các yêu cầu quan trọng nhất là việc tạo ra News Feed cho bất cứ user nào đó để lấy về các photos mới nhất từ tất cả những ngừời mà user theo dõi. Để làm được điều này, ta cần có 1 cơ chế để sort photos theo thời gian tạo. Để làm việc này 1 cách hiệu quả, ta có thể thêm thời gian tạo photo là 1 phần của PhotoID. Do đó ta chỉ cần đánh khóa chính vào PhotoID, là có thể dễ dàng nhanh chóng tìm ra được các PhotoIDs mới nhất.

Ta có thể sử dụng epoch time cho việc này. Do vậy PhotoID sẽ gồm có 2 phần: phần đầu tiên sẽ biểu diễn epoch time, phần thứ 2 sẽ là 1 chuỗi tự tăng auto-incrementing sequence. Do vậy khi tạo ra 1 PhotoID mới, ta cần lấy ra epoch time hiện tại và thêm nó vào 1 chuỗi tự tăng auto-incrementing ID từ key-generating DB. Ta có thể  dễ dàng xác định shard number từ PhotoID ( PhotoID % 10) này và lưu photo này vào shard number đó.

**Kích thước PhotoID là bao nhiêu?** Nếu epoch time bắt đầu từ thời điểm hiện tại; và ta sẽ cần bao nhiêu bits để lưu trữ số giây cho 50 năm tới?

```bash
86400 sec/day * 365 (days a year) * 50 (years) => 1.6 billion seconds
```

Chúng ta cần 31 bits để lưu trữ số này. Do, ta đang kỳ vọng trung bình có 23 photos mới mỗi giây, ta có thể phải cấp phát thêm 9 bits để lưu trữ chuỗi tự tăng auto-incremented sequence nữa. Do mỗi giây, ta có cần lưu trũ 2^9 = 512 photos mới. Chúng ta đang cấp pháy 9 bits cho chuỗi tự tăng là nhiều hơn những gì mà ta yêu cầu; chúng ta làm việc này bằng cách lấy ra 1 số full byte number (as 40 bits = 5 bytes40bits=5bytes). Ta có thể reset lại chuỗi tự tăng auto-incrementing sequence vào mỗi giây.

## 13. Cache và Load balancing

Chúng ta cần 1 hệ thống phân phối ảnh có khả năng mở rộng lớn để đáp ứng các users nằm phân tán trên toàn cầu. Hệ thống chỉ nên push các bội dung content gần với user bằng cách sử dụng lượng lớn các photo cache servers được phân tán theo vùng địa lý và sử dụng CDNs.

Chúng ta có thể  thêm vào 1 cơ chế cache cho metadata servers để cache lại các hot database rows. Chúng ta có thể sử dụng Memcache để cache data, và Application servers trước khi hit vào database có thể nhanh chóng kiểm tra được cache có chứa dữ liệu mong muốn. Chính sách LRU có thể được áp dụng để xóa cache trong hệ thống của ta. Khi dùng chính sách này, ta sẽ loại bỏ được các row ít được xem trước tiên.

**Cách mà ta xây dựng 1 hệ thống cache thông minh hơn?** Nếu ta tuân theo định luật 80/20, thì 20% tổng sản lượng read hàng ngày sẽ tạo ra 80% traffic, điều này có nghĩa là 1 vài photos nhất định sẽ trở nên phổ biến và được nhiều người đọc chúng. Điều này cho thấy rằng chúng ta có thể thử lưu vào bộ nhớ đệm 20% khối lượng ảnh và metadata đọc hàng ngày.
