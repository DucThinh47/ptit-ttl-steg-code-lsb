# Lab: Giấu tin trong mạng sử dụng kỹ thuật điều chỉnh LSB của các trường header TCP
## Lý thuyết
**1. Lý thuyết "Kênh ẩn" (Covert Channel)**

Lý thuyết cơ bản nhất được thể hiện là việc tạo ra một `kênh ẩn (covert channel)`.
- `Định nghĩa`: Đây là một kênh giao tiếp bí mật được xây dựng bên trên một kênh giao tiếp hợp lệ. Trong bài lab này, kênh hợp lệ là các `gói tin TCP/IP` thông thường. Kênh ẩn được tạo ra bằng cách điều chỉnh các `trường dữ liệu (metadata)` của kênh hợp lệ đó.
- `Yêu cầu`: Kỹ thuật này đòi hỏi sự đồng bộ giữa bên gửi và bên nhận. Cả hai bên phải thống nhất trước về một "giao thức" bí mật:
    - Trường nào được dùng để giấu tin? (Ở đây là `TTL` và `Window Size`).
    - Quy tắc mã hóa là gì? (Ở đây là quy tắc `LSB`: `TTL=100` nghĩa là `bit 0`, `TTL=101` nghĩa là `bit 1`,...).
    - Bên nhận (`detect_steg_packets.py`) phải biết chính xác quy tắc này để trích xuất và giải mã (`decode_bits.py`) thông điệp.
    
**2. Nguyên tắc "Tối thiểu hóa Bất thường" (Minimizing Anomaly)**

Đây là nguyên tắc quan trọng nhất để tránh bị phát hiện. Kẻ giấu tin không chọn các giá trị ngẫu nhiên mà chọn các giá trị tuân thủ giao thức và trông "bình thường".
- `Sử dụng trường linh hoạt`: Kỹ thuật này nhắm vào các trường trong header (như `TTL` và `Window Size`) có độ linh hoạt tự nhiên.
    - `TTL (Time-To-Live)`: Giá trị này giảm đi `1` sau mỗi "bước nhảy" (hop) qua router. Giá trị ban đầu (như `64`, `128`, `255`) là phổ biến, nhưng các giá trị như `100`, `101` vẫn hoàn toàn hợp lệ và không gây nghi ngờ ngay lập tức.
    - `Window Size (WS)`: Trường này dùng để kiểm soát luồng và có thể thay đổi trong suốt phiên kết nối.
- `Tránh giá trị bất thường`: Bài lab giải thích rõ tại sao chọn `[100, 101]` và `[2048, 2049]`:
    - Chúng chỉ chênh lệch `1` đơn vị (do thay đổi bit LSB).
    - Chúng nằm trong dải giá trị hợp lệ (ví dụ: TTL `100/101` gần với `128`; `WS 2048` là một giá trị chuẩn `2^11`).
    - Nếu chọn các giá trị quá khác biệt (ví dụ: giấu tin bằng cách đặt `TTL=1` hoặc `TTL=500`), các hệ thống giám sát an ninh mạng (như IDS/IPS) hoặc thậm chí chính các router sẽ dễ dàng phát hiện và loại bỏ gói tin.

**3. Lý thuyết "Mã hóa Bit có trọng số thấp nhất" (LSB Encoding)**

Đây là kỹ thuật mã hóa cụ thể được sử dụng, dựa trên nguyên tắc "Tối thiểu hóa bất thường" ở trên.
- `LSB (Least Significant Bit)`: Là bit cuối cùng (bên phải nhất) trong biểu diễn nhị phân của một số.
- `Tại sao dùng LSB?` Việc thay đổi LSB chỉ làm thay đổi giá trị thập phân của số đi `1 đơn vị` (ví dụ: `100 ...0` -> `101 ...1`; `2048 ...0` -> `2049 ...1`).
- `Ưu điểm`: Đây là cách thay đổi giá trị một cách tinh vi nhất, tạo ra sự khác biệt nhỏ nhất và khó bị phát hiện nhất nếu chỉ phân tích giá trị thập phân thông thường.

**4. Nguyên tắc "Phân mảnh Thông điệp" (Message Fragmentation)**

Một gói tin TCP đơn lẻ chỉ có thể mang một lượng tin ẩn rất nhỏ.
- `Một gói = Vài bit`: Trong bài lab này, mỗi gói tin chỉ mang được `2 bit` thông điệp ẩn (1 bit ở trường `TTL` và 1 bit ở trường `Window Size`).
- `Ghép chuỗi`: Để gửi một thông điệp có ý nghĩa (như `"ATTT"`, tương đương `32 bits`), bên gửi (`send_mixed_packets.py`) phải chia nhỏ thông điệp thành các bit riêng lẻ và gửi chúng đi trong một chuỗi nhiều gói tin.
- `Bên nhận (receiver)`: Phải bắt, lọc (dùng` Wireshark` hoặc script) và sắp xếp lại các gói tin này theo đúng thứ tự để trích xuất và ghép các bit lại thành thông điệp hoàn chỉnh.
## Thực hành
Trên máy `sender`, chỉnh sửa file `send_mixed_packets.py`:

    nano send_mixed_packets.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image0.png?raw=true)

Sửa giá trị message thành `“ATTT”`:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image1.png?raw=true)

Lưu file và đóng lại, bên máy `receiver`, tiến hành thực thi file `detect_steg_packets.py` để thiết lập kênh lắng nghe gói tin gửi đến:

    sudo python3 detect_steg_packets.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image2.png?raw=true)

Đồng thời, bên máy `sender`, tiến hành thực thi `file send_mixed_packets.py` để gửi gói tin:

    sudo python3 send_mixed_packets.py

Sau khi bên máy `sender` thông báo gửi xong gói tin, bên máy `receiver` ấn `Ctrl+C` để dừng bắt gói tin. Quy tắc giấu tin cơ bản của kỹ thuật này:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image3.png?raw=true)

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image4.png?raw=true)

Dựa vào quy tắc này, phân tích logs bên máy `receiver`, tìm ra giá trị bits của thông điệp ẩn. Sau đó, trên máy `sender`, thực thi file `decode_bits.py` để tìm ra nội dung thông điệp:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image5.png?raw=true)

Trên máy `receiver`, mở `Wireshark`:

    wireshark &

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image6.png?raw=true)

Tiếp theo, bên máy `sender`, thực thi file `send_ttl_win.py`:

    sudo python3 send_ttl_win.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image7.png?raw=true)

Quan sát `Wireshark`: 

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image8.png?raw=true)

Phân tích và tìm ra các gói có chứa thông điệp ẩn, sau đó ghép lại tìm ra giá trị bits của thông điệp. Có thể áp dụng bộ lọc để tìm dễ dàng hơn:

    tcp && ip.src == <ip_máy_sender>

Sau khi có được giá trị bits của thông điệp ẩn, trên máy `receiver`, thực thi file `decode_bits.py` để tìm ra thông điệp:

    python3 decode_bits.py






