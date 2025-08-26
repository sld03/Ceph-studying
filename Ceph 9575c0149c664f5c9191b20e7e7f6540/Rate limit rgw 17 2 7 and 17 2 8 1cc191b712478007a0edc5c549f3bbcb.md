# Rate limit rgw 17.2.7 and 17.2.8

# Test với môi trường 4 rgw trên 2 rgw service

![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image.png)

Có 4 rgw, hai cái là 17.2.7, hai cái 17.2.8. Cả 4 cái đều được load balance bởi một con HA Proxy

![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%201.png)

Có thêm một con client để tiến hành read write. Tất cả nằm trên cùng 1 network, cả được public và cluter của Ceph cũng vậy

# Lý thuyết trong document

Tính năng ratelimit sẽ limit số operation (read ops và write ops) và số bytes trong một phút.

Dùng GET và HEAD thì sẽ là read request, còn lại là write request

## Review lại cách hoạt động của metrics

**Với limit số operations**

Rados Gateway là chỗ lưu metric, và metrics ở các gateway không tương tác với nhau. Nghĩa là ví dụ user bị limit 10 operation/phút, thì mỗi gateway sẽ được 5 ops mỗi phút.

**Với limit số bytes**

Băng thông để tính số byte/phút chỉ được tính khi request được accept. Tức là sau phi ops hoàn thành thì nó mới tính số byte đã thực hiện trong phút đó. Ví dụ một user bị limit 100Mb/phút, mà thực hiện 1 operation read với object 1 Gb, Operation vẫn sẽ được thực hiện. Tuy nhiên, nó đã vượt quá 900Mb so với Limit về số bytes. Như vậy, user này đang có nợ (debt). Nợ tối đa là 2 lần limit số byte. Trong trường hợp này, user vượt quá 900Mb thì sẽ bị nợ 200 bytes, và bị block operation đến 2 phút.

# Tiến hành các test case

- Limit số User ops read
    
    ![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%202.png)
    
    Chạy script để liên tục lấy bucket. **s3cmd** trong trường hợp này được config với endpoint là HAproxy, cân bằng tải giữa 4 RGW
    
    ![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%203.png)
    
    Khi đặt sleep 0.5, thực hiện 30 read operation thực hiện trong khoảng 15 giây, thì không hề bị rate limit, dù limit chỉ là 12 read ops/ phút
    
    Khi giảm sleep xuống còn 0.25, tức là thực hiện 30 read ops trong khoảng  7.5 giây, thì có xuất hiện rate limit ở operation thứ 28. 
    
    ![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%204.png)
    
    Giảm xuống 0.1 thì dừng ở 26, 
    
    Giảm xuống 0.02 thì vẫn dùng ở tầm 24
    
    Nhìn chung nó sẽ x2 cái 
    
    Kết luận: tính năng limit ops hoạt động không đúng. Nó sẽ không phải là limit số operation trong một phút, mà là một đơn vị nhỏ hơn. và cơ chế ghi nợ của nó cũng không phải là kiểu block operation đến hết phút đó.
    
- Limit số User ops write
    
    ![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%205.png)
    
    Tính năng này cũng tương tự như cái kia, hoạt động không đúng với đơn vị phút, mà nó là đơn vị gì đó nhỏ hơn
    
    Khi test trên s3cmd với endpoint là chỉ 1 rgw, nó bị giới hạn bởi 6 operation. Theo lý thuyết, 4 con rgw thì phải mỗi con limit là 3, nhưng đây lại là 6. Có vẻ nó đang chia đôi, theo 2 loại version hoặc 2 rgw service khác nhau.
    
- Limit số byte read
    
    ![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%206.png)
    
    Tạo file 10Mb, up lên S3, sau đó get về liên tục. Nếu get đủ nhanh, sẽ dừng lại ở 6 - 8 ops, nhưng không phải limit theo phút. Đáng ra nó chỉ limit ở tầm 3 operation thôi, nhưng ở đây nó đang x2 lên. 
    
- Limit số byte write
    
    Cũng tương tự
    

# Setup lại chỉ còn 1 rgw service

![image.png](Rate%20limit%20rgw%2017%202%207%20and%2017%202%208%201cc191b712478007a0edc5c549f3bbcb/image%207.png)

Chỉ có 1 rgw service, 4 rgw, hai con 17.2.7, 2 con 17.2.8. Kì vọng kết quả sẽ khác nhưng nó vẫn tương đương :))

Lần này nó  x2 số operation lên, cụ thể là với limit operation, thì khi set bằng 12 thì nó sẽ limit ở 24 operation, xong nó dừng 3 giây, và chạy tiếp 24 operation tiếp, sau đó limit dần dần. Kết luận vẫn tương tự

# Kết luận

**Với limit operation**: theo lý thuyết, 4 con rgw thì phải số ops phải chia 4 cho mỗi con, nhưng nó chỉ chia 2. Vì vậy khi set limit ops là 12 thì nó sẽ là 24

**Với limit bytes:** cũng tương tự