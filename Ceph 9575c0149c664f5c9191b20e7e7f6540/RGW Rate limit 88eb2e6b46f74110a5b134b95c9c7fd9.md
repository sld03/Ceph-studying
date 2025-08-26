# RGW Rate limit

# 0. Setup

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image.png)

# 1. Rate limit cho User

## 1.1. Rate limit số bytes write

Khi enable rate limit với `--write-limit-byte` là 1000000 bytes ~ 1Gb / phút, thì khi dùng s3cmd upload một file 100Mb data random thì tốc độ các multipart như dưới.

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%201.png)

Sau một thời gian khá dài (30 phút) chờ đợi thì operation put object 100Mb vẫn chưa xong ⇒ Tính năng này hoạt động không chuẩn với S3, vì theo tính toán với rate limit là 1000000 byte/ phút thì chỉ cần vài giây là xong.

Trong khi đó, nếu disable rate limit đi thì việc upload file rất nhanh

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%202.png)

Trên đây là khi thực hiện với client là s3cmd, sau đó có thử thực hiện với boto3 trên python sdk upload vẫn không thành công. 

## 1.2. Rate limit read for user

Sau khi set limit thì gặp lỗi này khi lấy một file 1Gb

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%203.png)

Với file kích thước nhỏ không phải multipart thì chạy oke.

## 1.3. Limit số lượng Operation cho User

Tính năng này chạy ổn cho car write và read.

# 2. Tính năng cho bucket

## 2.1. Limit số ops cho bucket

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%204.png)

Chỗ này là các read operation được thực hiện liên tục sau khi set `—max-read-ops`  là 10 thì khi chạy nó chỉ đến 6 7 cái read operation là nó bị limit rồi, như vậy nó đang không hoạt động đúng như mong muốn. Cái này có thể là do một số nguyên do, dự đoán là:

- Do một số tham số gì đó kiểu mình dùng đến mấy chục phần trăm của limit thì nó dừng
- Khi xử lý rgw, trước kia em xem log, thì khi mình gửi một post request lên thì log lại log ra 2 request, cụ thể là trong log thì trước một request post thì sẽ có một request get được thực hiện mà chưa hiểu vì sao. Ví dụ là cái request có `req=0x7f1f389906f0`  thì nó được thực hiện trước khi cái post được thực hiện ở dưới.

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%205.png)

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%206.png)

Kết quả với write cũng tương tự, khi số ops limit là 10, thì có lúc 12 đến 15 ops mới bị rate limit.

## 2.2. Limit số byte cho bucket

Cũng tương tự như của user, không chạy được cho object lớn cần phải multipart

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%207.png)

# 3. Kiểm tra chia limit theo số lượng rgw daemon

Thiết lập môi trường gồm 2 rgw, một con nginx là proxy server, distribute request đến 2 daemon đó theo kiểu round-robin, limit số ops là 10.

Kết quả:

- Khi gửi 10 request đến proxy, thì đến request thứ 11 thì bị ratelimit như trên hình, chứng tỏ hoạt động đúng như mô tả.
    
    ![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%208.png)
    
- Khi gửi request liên tục lên một trong 2 daemon, không qua proxy thì chỉ được 5 request đầu là thực hiện được, đến cái thứ 6 thì bị rate limit, chứng tỏ tính năng hoạt động đúng.
    
    ![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%209.png)
    

# 4. Môi trường 3 rgw, 2 daemon được đặt sau một proxy

Trước

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2010.png)

Sau khi gửi 100 requests

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2011.png)

Chỗ này là hình ảnh metric trên HAProxy trước và sau khi thực hiện 100 request get một lúc lên proxy. Proxy đứng trước 2/3 rgw, và số ops limit là 100 cho cả read và write. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2012.png)

**Giờ thử cho HAProxy đứng trước 1/3 con rgw**

Trước

Sau khi gửi 55 request

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2013.png)

Khi set limit về số ops cho cả bucket và user, thì khi user đạt đến limit về số ops và bucket thì chưa,  nó vẫn sẽ dừng request lại, như thế là đúng với đặc tả. 

# 5. Test theo số bytes trên 1 phút với tất cả dữ liệu

Cái này chạy đúng nhé

Lưu ý, với **get object**, khi dùng s3cmd, thì nó sẽ có một cái get_bucket_location, và một cái get_obj, còn nếu dùng sdk (boto3) thì nó sẽ có một 2 cái get_obj, một cái là head request, một cái là get request

Như doc có đề cập, ví dụ như limit là 200Mb/phút, mà khi đó mình lại thực hiện lên đến 300Mb trên phút đó rồi thì mình sẽ coi như nợ 100Mb. Cho đến khi cái nợ này được trả, thì request đến đây sẽ bị block. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2014.png)

Theo docs thì cái limits theo bytes này thì không bị chia ra như cái số ops kia.

Như trên theo tính toán với ratelimit là 200Mb, thì đến cái thứ 22 là nó bị chặn request, thế là đúng.

Tuy nhiên trong trường hợp này nó cũng ảo vì nếu cho một con proxy đứng trước 2 rgw, thì nó cũng bị limit ở 200Mb, mà nếu cho proxy đứng trước 3 con thì nó sẽ bị limit ở tận 600Mb. Nguyên do có thể thấy là:

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2015.png)

Khi có 2 con rgw đứng sau proxy, bản chất của cái get request là nó sẽ có 2 operation, bao gồm get_bucket_location và get_obj. Cái get_bucket_location thì nó không tốn quá nhiều dung lượng, kiểu chỉ vài kb, còn cái get_obj là cái lấy dữ liệu chính. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2016.png)

Còn hình trên là khi mà mình cho 3 rgw đứng sau proxy, khi đó các request sẽ được phân bố đều hơn, thì limit sẽ nhân 3 lên.

Từ đây có thể kết luận rằng khi một cái rgw mà nó đạt đến limit thì nó sẽ tự động ngắt request vào nó, còn cái khác nếu chưa hết thì vẫn request bình thường.

### Làm cái anonymous user với cả thằng global user

### Anonymous user

Cái này nó hoạt động đúng nhá:

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2017.png)

Để làm cái này cần thiết lập ACL cho một cái object là —acl-public, dùng s3cmd setacl. Sau đó chạy một chuỗi câu lệnh curl để lấy object về một cách liên tục. Đến cái thứ 61, thì nó đã bị limit và chỉ nhận 215 bytes thay vì 10Mb như file gốc. Dữ liệu của 215 bytes đó là error message:

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2018.png)

Metric thì được phân bố đều và đúng như dự tính

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2019.png)

Nếu xét về số ops, thì cái này cũng chạy đúng cho anonymous user luôn.

### Với Global User

Cái này chạy cũng đúng nhé với limit số ops và cả số bytes.

### Trộn các cái lại với nhau

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2020.png)

Theo như thế này, thì limit cho user (không anonymous) sẽ là 100, còn cho user anonymous sẽ là 150).

Nhưng mà còn một cái nữa, là khi dùng s3cmd với anonymous request thì nó chỉ là một operation chứ không phải 2 như cái user bình thường kia.

Đối với cái user bình thường thì qua 300 request là bị dừng, như thế là đúng, vì có 3 con rgw, mỗi con chịu được 100 request. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2021.png)

Bây giờ là cái cho cái user ratelimit ở global xuống còn 70 ops, nó vẫn cho kết quả đúng. Cái này sẽ đúng với doc là khi mà một cái limit đến hạn mà cái khác chưa đến, nó sẽ limit. 

**Limit theo số bytes:**

Trái ngược với cái đằng trên, khi set global ratelimit set max read lên 200 Mb như hình, thì không thấy dấu hiệu gì của việc limit rate theo số byte cả. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2022.png)

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2023.png)

Như trên, thấy được dù cho là một chuỗi request được gửi lên trong một thời gian rất ngắn (vài giây), mỗi rgw out ra hơn khoảng 400Mb dữ liệu mà vẫn không bị limit dù ratelimit là 200Mb. ⇒ Tính năng này không đúng lắm.

Cùng lúc đó, nếu như ta set limit cho một user cụ thể, thì nó sẽ bỏ qua cái global và chỉ lấy cái user thôi:

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2024.png)

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2025.png)

Tuy nhiên khi ta tạo user khác không có limit về số bytes cho user đó, thì cái limit global sẽ có tác dụng. Điều này chứng tỏ user limit sẽ ghi đè global limit nếu 2 cái được set cùng nhau. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2026.png)

Bây giờ thử xem cái số ops nó có bị ghi đè không. 

Khi set limit cho user 1 là 50 ops và limit global là 20, thì nó cũng sẽ bỏ qua cái global config và global config sẽ bị ghi đè. 

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2027.png)

Tiếp theo, test thử với anonymous user, xem nó có bị ghi đè không, nhưng có vẻ là nó không bị. Khi set global limit cho anonymous user là 50 ops.

### Set global limit băng thông thấp, kết hợp với set cho một user là unlimit và trạng thái là enable, sau đó upload file với tốc độ cao, đang upload thì disable

![image.png](RGW%20Rate%20limit%2088eb2e6b46f74110a5b134b95c9c7fd9/image%2028.png)

Có. Đối với object mà đang upload multipart, thì khi đang upload mượt, ta disable cái unlimit từ user ratelimit đi thì nó sẽ lập tức bị chặn luôn.

### Công thức tính limit request và băng thông trong trường hơp có 3 rgw nhưng chỉ sử dụng 2

Trong trường hợp có `n` rgw, HAProxy đứng trước `m` rgw (`m < n`) để phục vụ cho user. Nếu như muốn set ratelimit cho user là `x` , khi đó ta cần setup trên hệ thống để limit cho user là `r=x/m` . Công thức này áp dụng cho cả số ops bytes, cả read và write.