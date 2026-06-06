# 🔒 SHA-256 Hardware Implementation in Verilog (Optimized)

[![FIPS 180-4](https://img.shields.io/badge/Standard-FIPS%20180--4-blue.svg)](https://csrc.nist.gov/publications/detail/fips/180/4/final)
[![Language](https://img.shields.io/badge/Language-Verilog%20HDL-orange.svg)]()
[![Architecture](https://img.shields.io/badge/Architecture-Unrolled--by--2-success.svg)]()

Dự án này là một bộ tăng tốc phần cứng (Hardware Accelerator) cho thuật toán băm mật mã **SHA-256**, được mô tả bằng ngôn ngữ phần cứng Verilog HDL. Hệ thống được thiết kế tối ưu hóa về mặt tốc độ (High Throughput) với kiến trúc lõi **Unrolled-by-2** và giao thức truyền dữ liệu **Back-to-back**.

## ✨ Tính năng nổi bật

* **Tuân thủ chuẩn FIPS 180-4:** Hỗ trợ đầy đủ thuật toán SHA-256 với kích thước block 512-bit và digest 256-bit.
* **Auto-Padding FSM:** Tích hợp sẵn bộ điều khiển thông điệp (Message Controller) có khả năng tự động đệm dữ liệu (padding) theo chuẩn. Hỗ trợ mượt mà cả 3 kịch bản padding:
  * Tin nhắn $\le$ 55 byte (1 Block).
  * Tin nhắn từ 56 đến 63 byte (2 Blocks - Trạng thái `STATE_EMIT_LASTA`).
  * Tin nhắn đúng 64 byte (2 Blocks).
* **Kiến trúc lõi Unrolled-by-2:** Tính toán đồng thời 2 vòng băm (rounds) trong cùng 1 chu kỳ xung nhịp. Giảm số chu kỳ của vòng lặp chính từ 64 xuống chỉ còn **32 chu kỳ**.
* **Hiệu suất cực đại (Maximum Throughput):** Tổng thời gian xử lý cố định cho mỗi block chỉ mất đúng **36 chu kỳ clock**.
* **Giao tiếp Back-to-back:** Hỗ trợ nhận dữ liệu liên tục 1 byte / 1 clock cycle không có chu kỳ chết (Zero Idle Cycles) khi mạch đang ở trạng thái rảnh (`in_ready = 1`).

## 📐 Kiến trúc hệ thống

Dự án được chia thành các module chính sau:

1. `sha256_top.v`: Module Top-level, quản lý giao tiếp tổng thể và đồng bộ dữ liệu.
2. `sha_message_controller.v`: Máy trạng thái (FSM) quản lý bộ đệm 512-bit, tự động thêm bit `1`, padding các bit `0` và chèn 64-bit độ dài thông điệp ở cuối.
3. `sha_core.v`: Lõi toán học SHA-256. Thực hiện mở rộng thông điệp (Message Expansion) và chạy 64 vòng băm thông qua các hàm logic ($Maj, Ch, \Sigma, \sigma$).

### Phân tích Timing (36 Cycles / Block)
Với kiến trúc Unrolled, lõi phần cứng tiêu tốn một lượng chu kỳ cố định và tối ưu cho mỗi block 512-bit:
* **1 cycle:** Padding & Đóng gói block.
* **1 cycle:** Latch dữ liệu vào Top và khởi động Core.
* **1 cycle:** Khởi tạo $a \rightarrow h$ và mồi dữ liệu (`CALC_WK`).
* **32 cycles:** Xử lý 64 vòng băm (2 vòng/cycle ở `CALC_ROUNDS`).
* **1 cycle:** Cộng tích lũy (Davies-Meyer) và xuất cờ `digest_valid` (`DONE`).
* **Tổng cộng:** **36 Clock Cycles**.

## 🚀 Hướng dẫn Mô phỏng (Simulation)

File Testbench (`tb_sha256_top.v`) được thiết kế để đọc dữ liệu từ một file văn bản ngoại vi (`input.txt`) và bón dữ liệu trực tiếp vào mạch với tốc độ cực đại.

### Các bước chạy mô phỏng:
1. Clone repository này về máy.
2. Tạo một file `input.txt` trong thư mục dự án (hoặc sửa lại đường dẫn `$fopen` trong file Testbench cho khớp với máy của bạn).
3. Nhập chuỗi cần băm vào file `input.txt` (ví dụ: `Hello World!`).
4. Thêm các file `*.v` vào phần mềm mô phỏng (Vivado / ModelSim / Icarus Verilog).
5. Đặt module `tb_sha256_top` làm top module và tiến hành chạy Run Simulation.

### Tính toán chu kỳ mô phỏng (Ví dụ với tin nhắn 1 Block)
Testbench đã được tối ưu hóa để ép đường truyền chạy liên tục (Back-to-back), do đó công thức tính tổng số chu kỳ từ lúc nạp byte đầu tiên đến khi có kết quả băm là:
> **Total Cycles = N + 36** *(Trong đó N là số byte của tin nhắn, $N \le 55$)*

*Ví dụ: Nạp chuỗi `Hello World!` (12 bytes) sẽ tiêu tốn chính xác `12 + 36 = 48 chu kỳ clock`.*

## 💻 Giao diện cổng I/O (Top Module)

| Tín hiệu | Hướng | Chiều rộng | Mô tả |
| :--- | :---: | :---: | :--- |
| `clk` | Input | 1 bit | Xung nhịp hệ thống (System Clock) |
| `reset_n` | Input | 1 bit | Tín hiệu Reset tích cực mức thấp |
| `byte_valid` | Input | 1 bit | Báo hiệu có byte dữ liệu hợp lệ ở ngõ vào |
| `byte_last` | Input | 1 bit | Báo hiệu đây là byte cuối cùng của tin nhắn |
| `data_in` | Input | 8 bit | Dữ liệu đầu vào (1 byte / cycle) |
| `in_ready` | Output | 1 bit | Mạch rảnh, sẵn sàng nhận dữ liệu |
| `digest` | Output | 256 bit | Mã băm SHA-256 kết quả |
| `digest_valid`| Output | 1 bit | Cờ báo hiệu kết quả băm đã tính xong và hợp lệ |
| `message_done`| Output | 1 bit | Báo hiệu đã hoàn thành toàn bộ thông điệp |

## 🛠 Môi trường & Công cụ phát triển
* **Ngôn ngữ:** Verilog 2001
* **Phần mềm tổng hợp & mô phỏng khuyên dùng:** AMD/Xilinx Vivado, ModelSim.

---
*Dự án được thiết kế cho mục đích học tập và nghiên cứu kiến trúc máy tính / phần cứng mật mã.*
