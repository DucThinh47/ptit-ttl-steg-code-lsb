# Lab: Giấu tin trong mạng sử dụng kỹ thuật điều chỉnh LSB của trường TTL
## Lý thuyết
**1. Kênh ẩn LSB trên trường TTL (Time-To-Live)**

Đây là kỹ thuật giấu tin cơ bản được sử dụng,tập trung chỉ vào một trường (TTL) và làm rõ hơn về LSB.
- `Kênh ẩn (Covert Channel)`: Là nguyên tắc cốt lõi, tạo một kênh giao tiếp bí mật.
- `Trường sử dụng`: `Time-To-Live (TTL)` trong IP Header.
- `Kỹ thuật mã hóa`: `LSB (Least Significant Bit)`.Việc thay đổi LSB chỉ làm thay đổi giá trị thập phân của số đi 1 đơn vị. 
    - Ưu điểm: Đây là cách thay đổi giá trị một cách tinh vi nhất, tạo ra sự khác biệt nhỏ nhất và khó bị phát hiện nhất nếu chỉ phân tích giá trị thập phân thông thường.
    - Để giấu bit 0: Gói tin được gửi với giá trị TTL có LSB là 0 (ví dụ: TTL = 64).
    - Để giấu bit 1: Gói tin được gửi với giá trị TTL có LSB là 1 (ví dụ: TTL = 65).

**2. Lý thuyết "Chaffing" - Thêm Gói tin Gây nhiễu**
- `Vấn đề`: Nếu một kẻ tấn công chỉ gửi các gói tin có LSB của TTL được điều khiển (ví dụ: chỉ gửi 64, 65, 64, 65...), luồng lưu lượng này sẽ rất đáng ngờ và dễ bị phát hiện.
- `Giải pháp (Chaffing)`: Script `send_encoded_2.py` thực hiện "gửi các gói tin bị làm nhiễu". Điều này có nghĩa là nó gửi xen kẽ hai loại gói tin:
    - `Gói tin ẩn (Stego Packets)`: Gói tin thật sự mang bit 0/1 của thông điệp (ví dụ: TTL=64/65).
    - `Gói tin mồi (Chaff Packets / Noise)`: Các gói tin hợp lệ, bình thường, được chèn vào không mang thông tin gì cả. Chúng có thể có giá trị TTL ngẫu nhiên hoặc LSB ngẫu nhiên.
- `Mục đích`: Làm cho việc phát hiện trở nên khó khăn hơn. Thay vì một luồng 100% là gói tin đáng ngờ, kẻ tấn công tạo ra một luồng lưu lượng lớn hơn, trong đó chỉ một phần nhỏ (các gói tin ẩn) là có ý nghĩa. Bên phân tích phải "lọc" nhiễu để tìm ra tín hiệu thật.

**3. Nguyên tắc "Kênh ẩn phụ thuộc Độ dài" (Length-Dependent Channel)**

Một đặc điểm quan trọng của giao thức ẩn này.
- `Tham số Bắt buộc`: Bên nhận `(receive_ttl_steg_1.py)` bắt buộc phải biết trước độ dài của thông điệp `(--chars 10)`.
- `Ý nghĩa`: Giao thức giấu tin này rất đơn giản. Nó không có cơ chế tự báo hiệu "Kết thúc thông điệp" (End-of-Message / EOM).
- `Hệ quả`: Bên gửi và bên nhận phải thống nhất (out-of-band) về độ dài tin nhắn trước khi truyền. Nếu bên nhận không biết độ dài, nó sẽ không biết khi nào phải dừng giải mã (hoặc sẽ giải mã sai, thiếu).

**4. Phát hiện dựa trên Phân tích Hậu kỳ (Post-Hoc Analysis)**

Bài lab mô tả một quy trình phát hiện và giải mã gồm nhiều bước, thay vì giải mã theo thời gian thực.
- `Ghi lại (Logging)`: `receive_ttl_steg_2.py` lắng nghe và ghi tất cả các gói tin (cả gói ẩn và gói nhiễu) vào `packet_log.txt`.
- `Lọc (Filtering)`: Nhiệm vụ của sinh viên là phải phân tích `packet_log.txt` (hoặc Wireshark) để lọc bỏ các gói nhiễu (chaff) và chỉ trích xuất LSB của các gói tin ẩn thật sự.
- `Giải mã (Decoding)`: Chuỗi bit đã được lọc sạch `(output.txt)` sau đó mới được đưa vào script `decode.py` để chuyển đổi trở lại thành ký tự (ví dụ: `'PTIT'`).
## Thực hành
Trên máy sender, mở và đọc nội dung file send_encoded_1.py:

    nano send_encoded_1.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image0.png?raw=true)

Để ý thông điệp `message=’B21DCAT181’` có 10 kí tự. Bên máy receiver, chạy lệnh sau:

    sudo python3 receive_ttl_steg_1.py --chars 10

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image1.png?raw=true)

Đồng thời bên máy sender, chạy lệnh sau:

    sudo python3 send_encoded_1.py

Quan sát cách các gói tin được gửi và nhận:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image2.png?raw=true)

Tiếp theo, sửa file send_encoded_1.py, gửi và giải mã thông điệp ‘PTIT’:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image3.png?raw=true)

Trên máy receiver, thực hiện lệnh sau để lắng nghe gói tin gửi đến:
sudo python3 receive_ttl_steg_2.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image4.png?raw=true)

Lúc này trên máy sender, thực thi file send_encoded_2.py:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image5.png?raw=true)

Sau khi chờ máy sender gửi xong, bên máy receiver ấn Ctrl+C để dừng lắng nghe. Kết quả sẽ được lưu vào file packet_log.txt, đọc nội dung file để xem log mạng khi có xuất hiện gói tin có thông điệp ẩn trong thực tế:
cat packet_log.txt:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image6.png?raw=true)

Tiếp theo, chỉnh sửa nội dung thông điệp trong file send_encoded_2.py thành ‘PTIT’, thực hiện gửi lại các gói tin đến máy receiver:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image7.png?raw=true)

Trên máy receiver, thực hiện đọc file packet_log.txt, tìm ra chuỗi bit của thông điệp ‘PTIT’ và lưu kết quả vào file output.txt: 

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image8.png?raw=true)

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image9.png?raw=true)

Trên máy receiver, chạy lệnh: 

    python3 decode.py

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image10.png?raw=true)

Nhập chuỗi bit được lưu trong output.txt:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image11.png?raw=true)

Kết quả là thông điệp ‘PTIT’. 

Trên máy receiver, mở Wireshark:

    wireshark &

Thực hiện gửi lại các gói tin từ máy sender: 

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image12.png?raw=true)

Quan sát trên wireshark, tìm và nhận dạng các gói có chứa thông điệp bên trong:

![img](https://github.com/DucThinh47/ptit-ttl-steg-code-lsb/blob/main/images/image13.png?raw=true)





