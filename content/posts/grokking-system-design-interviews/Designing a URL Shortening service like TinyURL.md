---
author: "telion"
title: "Bài 2: Cách thiết kế 1 dịch vụ URL Shortening service giống như TinyURL"
description: "Chúng ta sẽ cùng nhau thiết kế 1 dịch vụ URL Shortening service giống như TinyURL. Dịch vụ này sẽ cung cấp các định danh ngắn gọn dùng để điều hướng tới các URLs dài"
tags: ["system design interviews","Design URL Shortening service","Design TinyURL"]
categories: ["System Design"]
series: ["Grokking system design interviews"]
ShowToc: true
ShowBreadCrumbs: true
---

## 1. Tại sao chúng ta cần 1 dịch vụ kiểu làm ngắn URL shortening?

URL shortening được sử dụng để tạo ra các định danh ngắn gọn hơn shorter aliases cho các URLs dài dòng. Chúng ta gọi các định danh được làm ngắn này là các "short links.". Các users được điều hướng tới URL gốc khi họ thực hiện truy cập vào các short links này. Các short links sẽ tiết kiệm nhiều không gian khi hiện thị, in ấn, truyền thông điệp hay tweeted. Thêm nữa, các users thường hiếm khi gõ sai khi thực hiện gõ các URLs được làm ngắn gọn lại.

Ví dụ, nếu chúng ta làm ngắn URL sau thông qua TinyURL:

```bash
https://www.educative.io/courses/grokking-the-system-design-interview/m2ygV4E81AR
```

Chúng ta sẽ nhận được kết quả:

```bash
https://tinyurl.com/rxcsyr3r
```

URL được làm ngắn chỉ bằng 1 phần 3 so với URL gốc.
Hệ thống URL shortening được sử dụng để tối ưu links thông qua nhiều thiết bị devices, có thể track được các links cụ thể để phân tích audience, đo lường hiệu quả của chiến dịch quảng cáo, hay dùng để ẩn đi URLs gốc.

Nếu bạn chưa từng sử dụng tinyurl.com trước đây, bạn nên thử tạo ra 1 shortened URL và bỏ thời gian để sử dụng 1 số options khác nhau do dịch vụ cung cấp. Nó sẽ giúp bạn hiểu về chương này.

## 2. Yêu cầu và mục tiêu của hệ thống

> Bạn phải luôn luôn làm rõ các yêu cầu ngay từ thời điểm bắt đầu của buổi phỏng vấn. Và đảm bảo chắc chắn rằng đặt ra các câu hỏi để tìm ra chính xác phạm vi scope của hệ thống.

Hệ thống URL shortening system sẽ có các yêu cầu như sau:
**Các yêu cầu về mặt tính năng:**

1. Đưa ra 1 URL, dịch vụ của chúng ta sẽ tạo ra 1 định danh ngắn hơn và duy nhất cho nó. Định danh này được gọi là 1 short link. Link này được làm ngắn gọn đủ để dễ dàng khi thực hiện copied và pasted vào trong các ứng dụng applications.
2. Khi users truy cập  1 short link, dịch vụ của chúng ta sẽ điều hướng họ tới link gốc.
3. Các users có thể tùy chọn cấu hình tùy chỉnh custom short link cho URL của họ.
4. Các links sẽ bị hết hạn sau 1 khoảng thời gian. Các users có thể định nghĩa được thời gian hết hạn này.

**Các yêu cầu không liên quan tới tính năng:**

1. Hệ thống phải có tính sẵn sàng cao. Đây là yêu cầu bắt buộc bởi vì khi hệ thống bị down, thì tất cả các chuyển hướng URL sẽ gặp lỗi.
2. Chuyển hướng URL nên xảy ra theo thời gian thực với độ trễ thấp.
3. Các Shortened links nên được tạo ra 1 cách bất định (khó đoán trước).

**Các yêu cầu mở rộng:**

1. Phân tích Analytics, ví dụ 1 url được chuyển hướng bao nhiêu lần?
2. Dịch vụ nên được truy cập được thông qua REST APIs bởi các services khác.

## 3. Định lượng sức chứa và các ràng buộc

Hệ thống của ta sẽ nặng về read. Sẽ có rất nhiều chuyển hướng redirection requests so với các new URL shortenings. Ta giải định tỉ lệ này khoảng 100:1 giữa read và write.

**Định lượng traffic:** Giả sử, chúng ta sẽ có 500M new URL shortenings mới mỗi tháng, với tỉ lệ 100:1 read/write, ta kì vọng có 50B (50 tỷ) chuyển hướng redirections với cùng chu kỳ:

```bash
100*500M => 50B
```

Số lượng truy vấn trên giây Queries Per Second (QPS) là bao nhiêu? New URLs shortenings được làm ngắn trên mỗi giây:

```bash
500 million / (30 days * 24 hours * 3600 seconds) = ~200 URLs/s
```

Xét tới tỉ lệ 100:1 read/write, URLs redirections trên mỗi giây sẽ là:

```bash
100 * 200 URLs/s = 20K/s
```

**Định lượng lưu trữ storage:** Giả sử ta lưu trữ mọi URL shortening request (bao gồm cả shortened link tương ứng với nó) trong vòng 5 năm. Do ta kỳ vọng có 500M new URLs mỗi tháng, thì tổng số objects mà ta kỳ vọng lưu trữ là 30B (30 tỷ):

```bash
500 million * 5 years * 12 months = 30 billion
```

Ta giả định mỗi object được lưu trữ mất xấp xỉ 500 bytes. Ta sẽ cần 15TB lưu trữ storage:

```bash
30 billion * 500 bytes = 15 TB
```

**Định lượng băng thông Bandwidth:** Với các write requests, do chúng ta kỳ vọng 200 new URLs trên mỗi giây, tổng số dữ liệu incoming data sẽ khoảng 100KB trên giây:

```bash
200 * 500 bytes = 100 KB/s
```

Với các read requests, do mỗi giây ra kỳ vọng ~ 20K chuyển hướng URLs redirections, tổng số dữ liệu outgoing data sẽ khoảng 10MB trên giây:

```bash
20K * 500 bytes = ~10 MB/s
```

**Định lượng memory:** Nếu ta muốn cache 1 vài hot URLs mà thường xuyên được truy cập, số lượng memory và ta cần để lưu trữ chúng là bao nhiêu? Nếu ta tuân theo định luật 80-20 rule, có nghĩa là 20% số URLs sẽ tạo ra 80% traffic, Chúng ta sẽ muốn cache lại 20% hot URLs.
Do ta có khoảng 20K requests trên giây, ta sẽ có khoảng 1.7B (tỷ) requests mỗi ngày:

```bash
20K * 3600 seconds * 24 hours = ~1.7 billion
```

Ta sẽ cache lại 20% số requests này, ta sẽ cần 170GB memory:

```bash
0.2 * 1.7 billion * 500 bytes = ~170GB
```

1 điểm nữa ta cần lưu ý ở đây là sẽ có nhiều duplicate requests tới cùng 1 URL, nên số lượng memory mà ta sử dụng sẽ thấp hơn 170GB.
**Định lượng ở mức High-level:** Giả định có 500M new URLs mỗi tháng và tỷ lệ 100:1 read:write, bảng sau đây sẽ chỉ ra định lượng tổng quan của service:
| Chỉ số              | Định lượng |
| ------------------- | ---------- |
| New URLS            | 200/s      |
| URL redirections    | 20k/s      |
| Incoming data       | 100KB/s    |
| Outgoing data       | 10MB/s     |
| Storage for 5 years | 15TB       |
| Memory for cache    | 170GB      |

## 4. System APIs

> Khi chúng ta đã hoàn thành việc làm rõ yêu cầu, sẽ luôn là 1 ý tưởng tốt để định nghĩa ra các system APIs. Do đây đúng là những gì mà hệ thống kỳ vọng

Chúng ta có thể tạo ra SOAP hay REST APIs để lộ ra các tính năng của hệ thống. Đoạn mã dưới đây định nghĩa APIs dùng để tạo mới và xóa URLs:

```bash
createURL(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)
```

**Các tham số:**

- api_dev_key (string): API developer key của 1 account đã đăng ký. Nó sẽ được sử dụng cho nhiều mục đích khác như việc giới hạn gọi api của users dựa trên quota được cấp.
- original_url (string): URL gốc được làm ngắn
- custom_alias (string): custom key tùy chọn cho URL.
- user_name (string): tùy chọn user name được dùng để mã hóa encoding.
- expire_date (string): tùy chọn ngày hết hạn cho shortened URL.

**Return:(string)**
Khi việc thêm mới thành công nó sẽ trả về shortened URL; mặt khác nó sẽ trả về 1 mã lỗi error code.

```bash
deleteURL(api_dev_key, url_key)
```

Với `url_key` là chuỗi biểu thị shortened URL được trả về; Nếu xóa thành công sẽ trả về "URL Removed".
**Cách mà ta phát hiện và ngăn chặn việc lạm quyền?** Để ngăn chặn điều này, ta có thể giới hạn users thông qua `api_dev_key`. Mỗi `api_dev_key` có thể bị giới hạn 1 số lượng tạo URL creations hay chuyển hướng redirections trên 1 khoảng thời gian (Điều này có thể được thiết lập các khoảng thời gian khác nhau trên mỗi developer key).

## 5. Database Design

> Định nghĩa DB schema trong những bước đầu tiên của buổi phỏng vấn sẽ giúp hiểu được về luồng data flow giữa các thành phần components khác nhau trong hệ thống và từ đó nó sẽ hướng chúng ta tới việc quy hoạch phân vùng dữ liệu.

Có 1 vài quan sát về bản chất dữ liệu mà chúng ta sẽ lưu trữ như sau:

1. Chúng ta cần lưu trữ hàng tỷ bản ghi.
2. Mỗi object chúng ta lưu trữ thường rất nhỏ (nhỏ hơn 1K).
3. Không có mối quan hệ relationships giữa các bản ghi records--hơn là việc cần lưu lại user nào tạo ra URL.
4. Service thiên về read.

### Database schema

Chúng ta cần có 2 bảng tables: 1 dùng để lưu trữ thông tin về URL mapping và 1 bảng lưu trữ về thông tin user data người đã tạo ra short link.

![High-level design diagram](/grokking-system-design-interviews/5932826461995008.svg "High-level design diagram")

Loại database nào mà ta nên sử dụng? Do ta ước tính cần lưu trữ hàng tỷ bản ghi và ta không cần sử dụng mối quan hệ relationships giữa các objects - 1 NoSQL giống như DynamoDB, Cassandra hay Riak là lựa chọn phù hợp. Lựa chọn theo hướng NoSQL sẽ giúp dễ dàng mở rộng scale hơn.

## 6. Basic System Design và Algorithm

Vấn đề mà ta muốn giải quyết tại đây là cách tạo ra 1 định danh ngắn gọn và duy nhất cho 1 URL nào đó.
Trong ví dụ về TinyURL ở phần 1,  shortened URL có giá trị là "https://tinyurl.com/rxcsyr3r". 8 kí tự cuối trong chuỗi URL này chính là định danh mà ta muốn tạo. Ta sẽ đưa ra 2 giải pháp tại đây:

### a. Mã hóa URL gốc

Ta có thể tạo ra 1 chuỗi unique hash (MD5 hay SHA256) từ 1 URL đã cho. Chuỗi hash có thể được mã hóa lại cho việc hiện thị. Chuỗi mã hóa encoding này có thể là base36 ([a-z ,0-9]) hoặc base62 ([A-Z, a-z, 0-9]) và nếu ta thêm vào các ksi tự '+' và '/' ta có thể sử dụng base64 encoding. 1 Câu hỏi cần được đặt ra là độ dài về định danh short key là bao nhiêu? 6,8,hay 10 kí tự?

Sử dụng mã hóa base64 encoding, thì với 1 định danh có độ dài là 6 kí tự, sẽ tạo ra được 64^6 = ~68.7 tỷ định danh.
Sử dụng mã hóa base64 encoding, thì với 1 định danh có độ dài là 8 kí tự, sẽ tạo ra được 64^8 = ~281 tỷ tỷ định danh.

Với 68.7B unique strings, chúng ta giả định là định danh có độ dài là 6 kí tự là đủ cho hệ thống mà ta đang xây dựng.

Nếu ta sử dụng thuật toán MD5 algorithm hay các hash function, nó sẽ tạo ra chuỗi hash có giá trị 128-bit. Sau khi mã hóa base64 encoding, ta sẽ nhận được 1 chuỗi string có nhiều hơn 21 kí tự (do mỗi 1 kí tự base64 sẽ mã hóa 6 bits của chuỗi hash). Dù ta lưạ chọn định danh key có độ dài là 6 hay 8, thì ta vẫn gặp phải vấn đề là trùng lặp key; để giải quyết vấn đề này, ta có thể lựa chọn 1 vài kí tự khác nằm ngoài các chuỗi kí tự dùng để mã hóa.

**Những issues nào ta sẽ phải đối mặt?** Chúng ta sẽ đối mặt với 2 vấn đề với giải pháp mã hóa encoding:

1. Nếu có nhiều users cùng nhập vào cùng 1 URL, thì họ sẽ lấy ra cùng 1 shortened URL, điều này là không thể chấp nhận được.
2. Nếu 1 vài phần trong URL là đã được mã hóa URL-encoded thì sao? ví dụ `http://www.educative.io/distributed.php?id=design` và `http://www.educative.io/distributed.php%3Fid%3Ddesign` là giống URL trước ở dạng mã hóa URL encoding.

**Cách giải quyết các issues:** Chúng ta có thể thêm vào 1 chuỗi number tự tăng vào mỗi input URL để nó thành unique và sau đó tạo ra chuỗi hash. Ta không cần lưu trữ chuỗi number tự tăng này vào trong databases. Do đó, các vấn đề có thể gặp phải với cách giải quyết này là chuỗi number tự tăng này ngày càng tăng. Nó có thể tăng mãi mãi? Việc thêm vào 1 chuỗi number tự tăng cũng ảnh hưởng tới hiệu năng của service.

Cách giải quyết khác là thêm vào user id (sẽ là unique) vào input URL. Tuy nhiên, nếu user không đăng nhập, ta sẽ phải yêu cầu user chọn 1 chuỗi key duy nhất. Sau tất cả thì ta vẫn phải đối mặt với 1 mâu thuẫn conflict, ta sẽ phải giữ việc tạo ra định danh key cho tới khi ta tạo ra được 1 chuỗi key unique.

{{< youtube id="ZrnYOStAwsM" title="Request flow for shortening of a URL" >}}

### b. Tạo định danh keys offline

Chúng ta đã có 1 standalone **Key Generation Service (KGS)** dùng để tạo ra trước các chuỗi 6 ký tự ngẫu nhiên và sau đó lưu trữ chúng vào trong 1 database (ta gọi chúng là key-DB). Bất cứ khi nào chúng ta muốn làm ngắn 1 URL, ta sẽ lấy các keys đã được tạo sẵn trước đó và sử dụng nó. Cách tiếp cận này sẽ khiến mọi thứ trở nên đơn giản và nhanh hơn. Chúng ta không những không cần mã hóa URL mà còn không phải lo lắng về sự trùng lặp hay mâu thuẫn. KGS sẽ đảm bảo mọi keys được insert vào key-DB là duy nhất.

**Việc xử lý song song có gây ra vấn đề gì không?** Khi key được sử dụng, nó sẽ được đánh dấu trong database để chắc chắn rằng nó không được sử dụng lại lần nữa. Nếu có nhiều servers cùng đọc keys này vào cùng 1 thời điểm song song, chúng ta sẽ phải đối mặt với ngữ cảnh khi mà có 2 hay nhiều hơn servers cùng đọc cùng 1 key từ database. Cách nào để ta giải quyết vấn đề xử lý song song này?

Servers có thể sử dụng KGS để đọc/đánh dấu các keys trong database. KGS có thể sử dụng 2 tables để lưu trữ keys: 1 tables dùng cho các keys chưa được sử dụng, và 1 table cho tất cả các keys đã được sử dụng. KGS cấp phát các keys cho 1 trong các servers này, và chúng sẽ thực hiện di chuyển key sang bảng table lưu trữ keys đã sử dụng. KGS có thể luôn luôn giữ 1 vài keys trong memory để cấp phát chúng bất cứ khi nào servers cần tới chúng.

Để đơn giản, KGS sẽ nạp 1 vài keys vào memory, nó có thể di chuyển chúng sang table chứ các keys đã được sử dụng. Điều này đảm bảo mỗi server sẽ lấy về các unique keys. Nếu KGS dies trước khi gán tất cả các keys đã được loaded tới 1 vài server, chúng ta sẽ lãng phí những keys này--điều này có thể chấp nhận được.

KGS cũng đảm bảo không cấp phát trùng keys tới nhiều servers. Ví dụ, nó sẽ phải thực hiện đồng bộ (hay lock) cấu trúc dữ liệu data structure dùng để lưu trữ keys trước khi xóa bỏ những keys này khỏi nó vào cấp phát chúng cho 1 server.

**Kích thước key-DB này là bao nhiêu?** Với base64 encoding, ta có thể tạo ra được 68.7B chuỗi unique keys có độ dài là 6 kí tự. Nếu ta cần 1 byte để lưu trữ 1 kí tự alpha-numeric, thì ta cần:

```bash
6 (characters per key) * 68.7B (unique keys) = 412 GB.
```

**KGS có single point of failure nào không?** Có. Để giải quyết vấn đề này, ta cần có 1 standby replica của KGS. Bất cứ khi nào, primary server dies thì standby server sẽ thay thế vai trò của nó để tạo và cấp phát keys.

**Mỗi app server có thực hiện cache 1 vài keys từ key-DB?** Có, điều này có thể tăng tốc. Mặc dù, trong trường hợp này, nếu application server dies trước khi tiêu thụ tất cả các keys, chúng ta sẽ chấp nhận để mất các keys đó. Điều này có thể chấp nhận do chúng ta có 68B unique keys có độ dài là 6 kí tự.

**Cách chúng ta thực hiện xử lý key lookup?** Ta có thể lookup key trong database để lấy ra full URL. Nếu nó tồn tại trong DB, thì ta sẽ tạo ra “HTTP 302 Redirect” status tới browser, chuyển tiếp stored URL vào trong trường “Location” field của request. Nếu key không tồn tại trong hệ thống, thì ta tạo ra “HTTP 404 Not Found” status hoặc chuyển hướng user về lại homepage.

**Ta có nên áp đặt hạn mức size limits vào các định danh tùy chỉnh custom aliases?** Dịch vụ của ta có hỗ trợ custom aliases. Users có thể lựa chọn bất cứ định danh 'key' nào mà họ thích, nhưng việc cung cấp 1 custom alias là không bắt buộc. Tuy nhiên, ta cần áp đặt hạn mức size limit vào các custom alias để đảm bảo chúng ta có 1 URL database nhất quán. Chúng ta giả định user có thể định nghĩa 1 độ dài tối đa là 16 kí tự trên mỗi customer key (nó đã được phản ánh như trong database schema ở trên)

![High level system design for URL shortening](/grokking-system-design-interviews/5369668320100352.svg "High level system design for URL shortening")

## 7. Phân vùng Data Partitioning và Replication

Để mở rộng scale out DB, chúng ta cần phân tán dữ liệu do nó cần chứa hàng tỷ URLs. Do đó, chúng ta cần phát triển 1 partitioning scheme có khả năng phân tán và chia nó và lưu trữ nó ở trên nhiều các DB servers khác nhau.

**a. Phân tán dữ liệu theo Range based**: ta có thể lưu trữ URLs vào các phân vùng partitions tách biệt dựa vào kí tự đầu tiên của chuỗi hash key. Do đó ta có thể lưu tất cả các URL bất đầu với kí tự 'A' hay 'a' ở trong 1 phân vùng partition, tương tự lưu các URL bắt đầu với kí tự 'B' trong 1 phân vùng partition khác. Cách tiếp cận này được gọi là range-based partitioning. Chúng ta thậm chí có thể nhóm gộp các kí tự ít xuất hiện vào trong cùng 1 database partition. Do đó, chúng ta có thể phát triển 1 static partitioning scheme để luôn lưu trữ/tìm ra 1 URL theo 1 cách dễ đoán định.

Vấn đề chính với cách tiếp cận này là nó có thể dẫn tới việc DB servers không được cân bằng tải. Ví dụ, ta có thể quyết định lưu trữ tất cả URLs bắt đầu với ksi tự 'E' vào trong phân vùng 1 DB partition, nhưng nếu sau này khi chúng ta lại có quá nhiều URLs bắt đầu với kí tự 'E'.

**b. Phân tán dữ liệu theo Hash-Based**: Theo schema này, chúng ta tạo ra 1 chuỗi hash của object mà ta đang lưu trữ. Sau đó ta tính toán được phân vùng partition nào được sử dụng dựa trên chuỗi hash. Trong trường hợp của chúng ta, ta có thể lấy hash của định danh 'key' hay short link để xác định phân vùng partition dùng để lưu trữ data object.

Hàm hashing function sẽ tạo ra các URLs phân phối ngẫu nhiên vào các phân vùng partitions khác nhau (ví dụ hàm hashing function có thể luôn luôn ánh xạ bất kỳ 'key' nào đó với 1 số trong khoảng [1...256]). Số này sẽ biểu thị cho phân vùng partition mà ta lưu trữ object ở trong nó.

Cách tiếp cận này có thể hướng tới việc quá tải overloaded partitions, vấn đề này có thể được giải quyết bằng cách sử dụng Consistent Hashing.

## 8. Cache

Chúng ta có thể cache các URLs thường xuyên được truy cập. Ta có thể sử dụng bất cứ giải pháp nào có sẵn như Memcached, lưu trữ full URLs với chuỗi định danh hashes tương ứng của nó. Do vậy, application servers, trước khi hitting vào backend storage, chúng ta có thể nhanh chóng kiểm tra được nếu cache có chứa URL đang được request.

**Bao nhiêu cache memory mà ta sẽ sử dụng?** Chúng ta có thể bắt đầu dựa trên 20% traffic truy cập hàng ngày và dựa trên ước lượng sử dụng của client, ta có thể biết được bao nhiêu cache servers mà ta cần. Giống như việc định lượng ở trên, chúng ta cần 170GB memory để cache 20% daily traffic. Do các server hiện đại ngày nay có thể có tới 256GB memory, nên ta có thể dễ dàng chứa tất cả các cache của hệ thống trên 1 máy. Hay cách khác, ta có thể sử dụng 1 nhóm các servers nhỏ hơn để chứa các hot URLs này.

**Chính sach loại bỏ cache nào sẽ phù hợp nhất với yêu cầu hệ thống?** Khi cache đầy, và ta muốn thay thế 1 link với 1 URL mới hay hot hơn, ta sẽ làm điều này bằng cách nào? Chính sách Least Recently Used (LRU) có thể phù hợp với hệ thống. Với chính sách này ta có thể loại bỏ các URL ít được sử dụng gần đây nhất. Ta có thể sử dụng cấu trúc dữ liệu Linked Hash Map hay cấu trúc dữ liệu tương tụ để lưu trữ các URLs và các định danh Hashes, cấu trúc này sẽ trợ giúp ta track được các URLs mà được truy cập gần đây.

Để tăng hiệu quả, ta có thể nhân bản replicate các caching servers để phân phối tải load giữa chúng.

**Cách mà mỗi cache replica được cập nhật?** Bất cứ khi nào cache miss, servers sẽ hitting vào backend database. Bất cứ khi nào việc này xảy ra, ta có thể cập nhật cache và chuyển tiếp dữ liệu mới vào tất cả các cache replicas. Mỗi replica có thể cập nhật cache của nó bằng cách thêm vào dữ liệu mới. Nếu 1 replica đã có dữ liệu này, thì nó có thể đơn giản từ chỗi cập nhật.

{{< youtube id="6l_tATR7q_0" title="Request flow for accessing a shortened URL" >}}

## 9. Cân bằng tải Load Balancer (LB)

Ta có thể thêm 1 lớp cân bằng tải Load balancing layer vào 3 vị trí sau trong hệ thống:

1. Giữa Clients và Application servers.
2. Giữa Application Servers và database servers.
3. Giữa Application Servers và Cache servers.

Đầu tiên, ta có thể sử dụng cách tiếp cận Round Robin đơn giản để phân phối các incoming requests cân bằng giữa các backend servers. Đây là cách cân bằng tải LB đơn giản để thực thi. 1 ưu điểm khác của cách tiếp cận này là nếu 1 server chết thì LB sẽ loại bỏ nó ra khoản nhóm vòng xoay và sẽ ngừng gửi traffic tới nó.

1 vấn đề phát sinh với Round Robin LB là nó không quan tâm tới tải server load. Do vậy, nếu 1 server bị quá tải hay bị chậm thì LB sẽ không ngừng gửi request mới tới server đó. Để khắc phục vấn đề này, 1 giải pháp LB thông minh hơn có thể được áp dụng là LB sẽ định kỳ truy vấn về tải của backend server và điểu chỉnh traffic dựa vào đó.

## 10. Thanh lọc và dọn dẹp DB

Các dữ liệu có nên tồn tại mãi mãi hay nên được thanh lọc? Nếu thời gian hết hạn do user định nghĩa đã đến thì điều gì cần nên làm với các link hết hạn này?

Nếu ta chọn cách liên tục tìm kiếm các links đã hết hạn để xóa chúng, thì nó gây sức ép lớn đến database. Thay vào đó, ta có thể chậm rãi xóa bỏ các links cũ đã hết hạn này bằng cách dọn dẹp kiểu lazy cleanup. Service của chúng ta cũng cần chắc chắn với các links hết hạn thì sẽ được xóa dù 1 vài links đã hết hạn vẫn sống lâu hơn nhưng nó sẽ không bao giờ được trả về cho users.

- Bất cứ khi nào user thử truy cập vào 1 link hết hạn, ta có thể xóa link này và trả về lỗi cho user.
- 1 Cleanup service độc lập sẽ chạy định kỳ để xóa các links đã hết hạn ra khỏi storage và cache. Service này sẽ rất nhẹ và được lập lịch chạy vào thời điểm mà user traffice thấp.
- Ta luôn có 1 ngưỡng thời gian hết hạn cho từng link (ví dụ: 2 năm).
- Sau khi xóa 1 link đã hết hạn, ta có thể lưu lại key trở lại key-DB để tái sử dụng lại.
- Ta cũng nên xoá các links mà không được truy cập sau 1 khoảng thời gian, ví dụ như 6 tháng? Đây có thể là 1 mẹo. Do lưu trữ storage tương đối rẻ, ta cũng có thể quyết định lưu trữ links mãi mãi.

![Detailed component design for URL shortening](/grokking-system-design-interviews/6116923725578240.svg "Detailed component design for URL shortening")

## 11. Telemetry

Có tổng số bao nhiêu lần 1 short URL được sử dụng, hay user truy cập chúng từ đâu? Cách mà ta sẽ lưu trữ các thống kê này? Nếu mỗi lần có 1 view mới ta lại thực hiện cập nhật 1 DB row, thì điều gì sẽ xảy ra khi 1 URL phổ biến có 1 lượng lớn các concurrent requests?

Một vài thống kê khác cũng đáng để theo dõi: quốc gia của các visitor, thời điểm của các truy cập, các web page tham chiếu tới, browser, hay platform của các visitor.

## 12. Bảo mật và quyền hạn Security và Permissions

Users có thể tạo ra các private URLs hay cho phép 1 tập các users cụ thể nào đó được phép truy cập URL?

Chúng ta có thể lưu trữ mức quyền hạn permission level (public/private) với từng URL trong database. Ta có thể tạo ra 1 bảng table riêng để lưu trữ các UserIDs có quyền truy cập 1 URL cụ thể. Nếu 1 user không có quyền hay cố gắng truy cập vào 1 URL, ta có thể gửi 1 lỗi (HTTP 401). Như đã trình bày ở trên, chúng ta đang lưu trữ dữ liệu trong NoSQL wide-column database như Cassandra, khóa key cho table lưu trữ quyền hạn permissions sẽ là 'Hash' hay chuỗi do KGS sinh ra. Cột này sẽ lưu trữ giá trị của các UserIDs được phép truy cập URL.
