# Báo cáo Redis 

## 1. Giới thiệu Redis

Redis (Remote Dictionary Server) là một hệ thống lưu trữ dữ liệu **mã nguồn mở, trong bộ nhớ (in memory)**, hoạt động dưới dạng **key-value store**. Tuy nhiên, Redis vượt trội hơn so với các key-value store khác nhờ hỗ trợ nhiều cấu trúc dữ liệu phức tạp. 

**Đặc điểm chính**

- **Tốc độ cao**: Do dữ liệu chủ yếu được lưu trữ và xử lý trên RAM, Redis cung cấp độ trễ đọc/ghi rất thấp (thường dưới milisecond)

- **Hỗ trợ nhiều cấu trúc dữ liệu**: Ngoài kiểu string cơ bản, Redis hỗ trợ Lists, Sets, Sorted Sets, Hashes, HyperLogLogs, Geospatial indexes, và Streams. Điều này cho phép giải quyết nhiều bài toán phức tạp một cách hiệu quả ngay tại tầng dữ liệu. 

- **Tính nguyên tử**: Các thao tác trên một cấu trúc dữ liệu đơn lẻ thường là nguyên tử, đảm bảo tính nhất quán dữ liệu. Redis cũng hỗ trợ Transactions và Lua scripting để thực thi nhiều lệnh một cách nguyên tử. 

- **Độ bền (Persistence)**: Mặc dù là in-memory, Redis cung cấp các cơ chế để lưu trữ dữ liệu trên disk (RDB, AOF), đảm bảo dữ liệu không bị mất khi khởi động lại server. 

- **Khả năng mở rộng (Scalability)** : Hỗ trợ Replication (Master-Slave) để tăng khả năng đọc và chịu lỗi, và Clustering để phân tán dữ liệu trên nhiều node, cho phép mở rộng cả ghi và đung lượng lưu trữ.

- **Tính linh hoạt**: Có thể được sử dụng như một database, cache, message broker, và nhiều vai trò khác. 

**Cơ chế hoạt động của Redis**

- **Lưu trữ dữ liệu trong RAM**: Đây là yếu tố cốt lõi tạo nên tốc độ của Redis. Mọi hoạt động đọc/ghi chính đều diễn ra trên bộ nhớ RAM. Dữ liệu có thể được lưu xuống disk một cách đồng bộ hoặc không đồng bộ (tùy cấu hình persistence) để đảm bảo độ bền. 

- **Xử lý request theo Event Loop (Single-Threaded)**: Redis sử dụng mô hình event loop dựa trên I/O multiplexing (như `epoll`, `kqueue`, `select`) để xử lý các kết nối và request từ client trên một luồng duy nhất (single thread). 

    - Khi một client kết nối hoặc gửi request, sự kiện đó được đưa vào hàng đợi sự kiện.
    - Event loop liên tục kiểm tra hàng đợi này.
    - Khi có sự kiện (ví dụ: dữ liệu đến từ socket), Redis xử lý lệnh tương ứng. Do các lệnh Redis thường rất nhanh (vì thao tác trên RAM), việc xử lý trên một luồng không gây tắc nghẽn đáng kể cho các request khác. 
    - Sau khi xử lý xong, kết quả được gửi trả lại client. 
    - Mô hình này tránh được overhead của việc tạo và quản lý nhiều luồng, đơn giản hóa việc xử lý đồng thời và tránh các vấn đề phức tạp liên quan đến race conditions khi truy cập dữ liệu chung. Tuy nhiên, các lệnh chạy lâu (như `KEYS` trên tập dữ liệu lớn hoặc các Lua script phức tạp) có thể block event loop, ảnh hưởng đến của các client khác. 

## 2. Persistence trong Redis (Độ bền dữ liệu)

Redis cung cấp các cơ chế để lưu trạng thái dữ liệu từ RAM xuống disk, đảm bảo dữ liệu không bị mất khi server gặp sự cố hoặc restart. 

**RDB (Redis Database Backup)**

- **Cơ chế**: Snapshotting (Chụp ảnh nhanh). Redis định kỳ tạo một bản sao (snapshot) của toàn bộ dataset trong bộ nhớ tại một thời điểm nhất định và lưu vào một file `.rdb` trên đĩa.
- **Cách hoạt động**: Redis `fork` một tiến trình con. Tiến trình con này sẽ thực hiện việc ghi dữ liệu vào file `.rdb` trong khi tiến trình cha tiếp tục phục vụ các request mới. Cơ chế copy-on-write của hệ điều hành giúp giảm thiểu ảnh hưởng hiệu năng lên tiến trình cha. 
- **Ưu điểm**: 
    - File `.rdb` nhỏ gọn, tối ưu cho việc backup và di chuyển. 
    - Tốc độ khôi phục từ file `.rdb` thường nhanh hơn AOF vì chỉ cần tải một file duy nhất. 
    - Ít ảnh hưởng đến hiệu năng ghi của Redis trong hoạt động bình thường (việc ghi xuống đĩa được thực hiện bởi tiến trình con).
- **Nhược điểm**: 
    - Có khả năng mất dữ liệu giữa các lần snapshot. Nếu Redis bị crasj trước khi snapshot tiếp theo được tạo, tất cả các thay đổi từ lần snapshot cuối cùng sẽ bị mất. Khoảng thời gian mất dữ liệu phụ thuộc vào tần suất cấu hình snapshot (ví dụ: `save 900 1` - lưu sau 900s nếu có ít nhất 1 key thay đổi). 

**AOF (Append Only File)**

- **Cơ chế**: Ghi lại log các lệnh ghi. Redis ghi lại tất cả các lệnh làm thay đổi dữ liệu (như `SET`, `INCR`, `LPUSH`, `SADD`, v.v.) vào một file `.aof` theo thứ tự chúng được thực thi. 
- **Cách hoạt động**: Khi Redis restart, nó sẽ đọc file `.aof` và thực thi lại tuần tự các lệnh đã ghiu để khôi phục lại trạng thái ban đầu. 
- **Chính sách ghi (fsync)**: 
    - `always`: Ghi vào đũa sau mỗi lệnh (an toàn nhất nhưng lại chậm nhất).
    - `everysec` (mặc định): Ghi vào disk mỗi giây (cân bằng giữa an toàn và hiệu năng, có thể mất tối đa 1 giây dữ liệu)
    - `no`: Để hệ điều hành quyết định khi nào ghi (nhanh nhất nhưng kém an toàn nhất).
- **AOF Rewrite**: Do file `.aof` liên tục lớn lên, Redis cung cấp cơ chế "rewrite" để tạo ra một file `.aof` mới, nhỏ gọn hơn nhưng vẫn chứa đủ thông tin để tái tạo dataset hiện tại. Quá trình này cũng tương tự RDB, sử dụng tiến trình con. 
- **Ưu điểm**: 
    - Độ bền cao hơn RDB. Với cấu hình `everysec`, chỉ có thể mất tối đa 1 giây dữ liệu. Với `always`, gần như không mất dữ liệu (trừ trường hợp hệ thống file hoặc đĩa cứng gặp vấn đề).
    - File `.aof` dễ đọc và phân tích hơn (là log các lệnh).
- **Nhược điểm**: 
    - File `.aof` thường lớn hơn file `.rdb` tương đương.
    - Tốc độ khôi phục từng AOF có thể chậm hơn RDB, đặc biệt với file `.aof` lớn, vì phải thực thi lại từng lệnh. 
    - Có thể ảnh hưởng đến hiệu năng ghi nhiều hơn RDB (tùy thuộc vào `fsync`)

**Hybrid Persistence**

- **Cơ chế**: Kết hợp cả RDB và AOF. Khi bật chế độ này (từ Redis 4.0 trở đi), quá trình AOF rewrite sẽ không ghi lại từng lệnh mà ghi phần RDB snapshot vào đầu file AOF, sau đó tiếp tục ghi các lệnh thay đổi xảy ra trong quá trình rewrite.
- **Ưu điểm**: 
    - Tận dụng tốc độ khôi phục nhanh của RDB (phần snapshot ban đầu).
    - Đảm bảo độ bền cao của AOF (các lệnh thay đổi sau snapshot).
    - Cung cấp sự cân bằng tốt nhất giữa tốc độ khôi phục và độ an toàn dữ liệu. Đây thường là lựa chọn được khuyến nghị. 
