# 🎬 OPTICINE: HỆ THỐNG QUẢN TRỊ VẬN HÀNH & ĐIỀU PHỐI TÀI NGUYÊN RẠP CHIẾU PHIM

Một dự án Học tập Dựa trên Nghiên cứu (RBL) nâng cao được phát triển. Dự án này tập trung giải quyết các bài toán logic backend phức tạp và thách thức phân bổ tài nguyên trong quản lý rạp chiếu phim.

Khác với các ứng dụng đặt vé thông thường, **OptiCine** là một nền tảng Back-Office chuyên biệt được thiết kế để tự động hóa việc xếp lịch chiếu phim, tối ưu hóa doanh thu thông qua các thuật toán thỏa mãn ràng buộc, và đảm bảo kiểm soát chặt chẽ các giao dịch cơ sở dữ liệu.

---

## 🔬 1. Trọng tâm Nghiên cứu (RBL): Tự động hóa Xếp lịch

Cốt lõi nghiên cứu của dự án này giải quyết **Bài toán Thỏa mãn Ràng buộc (Constraint Satisfaction Problem - CSP)** vốn có trong việc phân bổ các tài nguyên giới hạn (phòng chiếu, khung giờ, nhân sự) nhằm tối đa hóa doanh thu kinh doanh và đồng thời ngăn chặn các xung đột lịch trình.

### 1.1 Đặt vấn đề
* **Xung đột tài nguyên:** Việc xếp lịch thủ công thường dẫn đến các suất chiếu bị trùng lặp, khoảng thời gian dọn vệ sinh giữa hai phim không đủ, hoặc xếp trùng hai phim vào cùng một phòng chiếu.
* **Phân bổ doanh thu chưa tối ưu:** Các bộ phim bom tấn không được phân bổ vào các phòng chiếu có sức chứa lớn nhất tại đúng những khung giờ vàng (Prime Time) một cách nhất quán do giới hạn tính toán của con người.

### 1.2 Mô hình Toán học & Hàm Mục tiêu
Thuật toán xếp lịch tự động sẽ đánh giá các tài nguyên khả dụng để tối đa hóa hàm ước lượng doanh thu sau:

📈 Tối đa hóa Doanh thu (Z) = Σ [ P(i) × C(i) × W(i) ]

Trong đó:
* 🎯 P(i): Tỉ lệ lấp đầy dự kiến của phim (i) (dựa trên bảng xếp hạng độ hot).
* 🪑 C(i): Sức chứa (số lượng ghế) của phòng chiếu được phân công.
* ⏰ W(i): Hệ số nhân theo khung giờ (Ví dụ: Giờ vàng Tối thứ 7 = 1.5, Sáng thứ 2 = 0.8).

### 1.3 Chiến lược Tối ưu hóa & Xử lý sự cố Database
Để hỗ trợ các truy vấn cường độ cao từ thuật toán CSP, kiến trúc cơ sở dữ liệu MySQL được tối ưu hóa sâu:
* **Đánh chỉ mục đa cột (Multi-Column Indexing):** Áp dụng cho các trường `room_id`, `start_time`, và `end_time` để giảm thiểu độ trễ tối đa khi kiểm tra tình trạng phòng trống.
* **ACID Transactions:** Khi quản lý phê duyệt một lịch chiếu được gợi ý, hệ thống phải thực hiện thao tác lưu hàng loạt (batch inserts) cho hàng trăm suất chiếu. Việc kiểm soát giao dịch (transaction) nghiêm ngặt đảm bảo tính toàn vẹn dữ liệu (Commit thành công toàn bộ hoặc Rollback toàn bộ) để ngăn chặn dữ liệu rác.

---

## 🏗️ 2. Kiến trúc Phần mềm & Các Design Pattern (GoF)

Source code được cấu trúc để tách biệt logic nghiệp vụ cốt lõi khỏi các framework phụ thuộc, đảm bảo hệ thống dễ bảo trì và dễ viết test, đáp ứng các tiêu chí khắt khe của buổi bảo vệ SWD392.

### 2.1 Các Design Pattern đã áp dụng

#### A. Strategy Pattern (Module lõi: `scheduler-service`)
Cho phép hệ thống chuyển đổi linh hoạt giữa các thuật toán xếp lịch khác nhau dựa trên mục tiêu của ban quản lý.
* **Interface:** `SchedulingStrategy`
* **Các lớp triển khai (Concrete Strategies):**
  * `RevenueMaximizeStrategy`: Ưu tiên xếp kín các bộ phim bom tấn vào các phòng lớn trong khung giờ vàng.
  * `DiversityStrategy`: Đảm bảo lịch chiếu được cân bằng cho cả phim độc lập (indie) và phim bom tấn ở các khung giờ khác nhau.

#### B. Chain of Responsibility Pattern (Module lõi: `approval-service`)
Quản lý luồng xử lý phê duyệt phân cấp cho giấy phép phim và các lịch chiếu vừa được thuật toán tạo ra.
* **Luồng xử lý:** Bản nháp từ Thuật toán $\rightarrow$ Trưởng ca kiểm tra $\rightarrow$ Quản lý chi nhánh phê duyệt $\rightarrow$ Đồng bộ lên Frontend bán vé.
* Pattern này giúp giảm sự phụ thuộc cứng giữa người gửi yêu cầu phê duyệt và người nhận, cho phép dễ dàng thêm/bớt các cấp quản lý trong tương lai.

#### C. Builder Pattern (Module lõi: `schedule-builder`)
Được sử dụng để khởi tạo đối tượng `DailySchedule` (Lịch chiếu trong ngày) cực kỳ phức tạp. Đối tượng này bao gồm việc phân công nhiều `Room` (Phòng chiếu) và chứa các mảng `Showtime` (Suất chiếu) lồng nhau. Pattern này đóng gói quy trình lắp ráp phức tạp, giúp phần code gọi API trở nên gọn gàng và dễ đọc.

---

## 🛠️ 3. Công nghệ Sử dụng

* **Backend Framework lõi:** Java 17, Spring Boot 3.x
* **Cơ sở dữ liệu (Transactional):** MySQL 8.x
* **Caching (Tối ưu hiệu năng):** Redis (Lưu tạm trạng thái các phòng chiếu trong lúc thuật toán thực thi)
* **Build Tool:** Maven
* **Quản lý mã nguồn:** Git & GitHub (Áp dụng chiến lược phân nhánh rõ ràng để làm việc nhóm)

---

## 📂 4. Cấu trúc Thư mục

Bên trong hệ thống backend, cấu trúc được phân chia rành mạch:
* **`domain/`**: Các thực thể nghiệp vụ cốt lõi (Movie, Room, Showtime).
* **`service/`**: Logic nghiệp vụ và các thuật toán xếp lịch.
* **`strategy/`**: Các lớp cài đặt cho Strategy Pattern.
* **`approval/`**: Các bộ xử lý (handlers) cho Chain of Responsibility.
* **`repository/`**: Các interface giao tiếp với MySQL thông qua JPA.
* **`presentation/`**: Các REST Controller cung cấp API cho màn hình Dashboard của quản lý rạp.






[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/rBNuyfdi)

[!Link Jira](https://swd392sba301group69.atlassian.net/jira/software/projects/KAN/boards/2?atlOrigin=eyJpIjoiYTBiNTQyMzYzYmFhNDlkMDg4NjRlMzQxNTU4ZjdkNWMiLCJwIjoiaiJ9)