# Bài 1: Quản lý biến trong RAM ECU – Từ cơ bản đến chuyên sâu

**Mục tiêu:**

- Hiểu rõ biến là gì, cách nó được lưu trong RAM của ECU.  
- Nắm vững các kiểu dữ liệu cố định kích thước (`stdint.h`) – tiêu chuẩn bắt buộc trong lập trình nhúng ô tô.  
- Biết cách hạn chế số thực, sử dụng số nguyên với tỉ lệ (scaling) để đảm bảo an toàn và hiệu năng thời gian thực.

---

## 1. Chuẩn bị công cụ

Để thực hành, bạn cần cài đặt môi trường lập trình C trên máy tính:

- **VS Code**: [Tải tại đây](https://code.visualstudio.com)
- **Extensions cần cài trong VS Code**:
  - C/C++ (Microsoft)
  - Code Runner
- **Trình biên dịch GCC** (thông qua MinGW64 hoặc MSYS2): [Hướng dẫn tải GCC/MinGW64](https://winlibs.com)

Sau khi cài đặt, kiểm tra bằng cách mở terminal và gõ:

```bash
gcc --version
```

## 2. Biến là gì?

### 2.1 Khái niệm cơ bản

Biến (variable) là **một vùng nhớ trong RAM** được đặt tên để lưu trữ dữ liệu tạm thời trong quá trình chương trình chạy.

Mỗi biến có:

- **Tên biến** (định danh)

- **Kiểu dữ liệu** → xác định kích thước vùng nhớ và cách diễn giải các bit.

- **Giá trị** được lưu dưới dạng nhị phân.

Ví dụ thông thường (chỉ mang tính giới thiệu):

```c
#include <stdio.h>

int main(void) {
  int tuoi = 22;
  float gpa = 3.55f;
  char ten = 'P';

  printf("Ten: %c\n", ten);
  printf("Tuoi: %d\n", tuoi);
  printf("GPA: %f\n", gpa);

  return 0;
}
```

**Kết quả mong đợi khi chạy:**

```bash
Ten: P
Tuoi: 22
GPA: 3.55
```

> *Lưu ý:* Có thể thay thông tin cá nhân vào — phải gõ tay toàn bộ.

**=> Giải thích chi tiết luồng chương trình chạy và tại sao có kết quả như vậy:**

<details> <summary><b>Xem giải thích (bấm để mở)</b></summary>

Chương trình bắt đầu từ hàm `main()`. Bên trong `main`, lần lượt:

- `int tuoi = 22;` – Trình biên dịch dành ra 4 byte trong RAM (trên PC 64-bit, `int` 4 byte) và ghi giá trị 22 vào đó.

- `float gpa = 3.55f;` – Dành ra 4 byte (kiểu `float` 4 byte) và lưu giá trị 3.55 dưới dạng số dấu phẩy động theo chuẩn IEEE 754.

- `char ten = 'P';` – Dành ra 1 byte và lưu mã ASCII của ký tự ‘P’ (là 80 ở hệ thập phân, hay 0x50 ở hệ hex).

Khi gọi `printf`:

- `printf("Ten: %c\n", ten);` – `%c` báo cho `printf` biết đối số thứ hai là một ký tự, nó sẽ đọc giá trị byte của `ten` (80) và in ra ký tự tương ứng trong bảng ASCII, đó là ‘P’. Sau đó `\n` đưa con trỏ xuống dòng mới.

- `printf("Tuoi: %d\n", tuoi);` – `%d` giúp `printf` diễn giải 4 byte của `tuoi` như một số nguyên có dấu, và in ra “22”.

- `printf("GPA: %f\n", gpa);` – `%f` giúp `printf` diễn giải 4 byte của `gpa` như số thực dấu phẩy động và in với 6 chữ số thập phân mặc định, kết quả là “3.550000” nhưng vì không có phần thập phân khác không nên thường hiển thị “3.55” tuỳ môi trường (thực ra là “3.550000” nếu chạy chính xác chuẩn – tuy nhiên nhiều compiler làm tròn hiển thị). Ở đây in ra “3.55” (thực chất là “3.550000” nhưng bị cắt bớt số 0 vô nghĩa).

Tóm lại, `printf` dựa vào định dạng `%c`, `%d`, `%f` để diễn giải các byte trong RAM một cách phù hợp và hiển thị ra màn hình.

</details>

> Lưu ý: Đoạn code trên dùng các kiểu dữ liệu cơ bản của `C`, phù hợp cho việc học cú pháp. Tuy nhiên, trong hệ thống nhúng ô tô, chúng ta **không nên** dùng trực tiếp các kiểu này vì kích thước của chúng phụ thuộc vào nền tảng (máy tính 64-bit, vi điều khiển 8-bit, 32-bit...). Điều này sẽ được làm rõ ở phần tiếp theo.

## 3. Vấn đề kích thước biến và giải pháp `stdint.h`

### 3.1 Tại sao `int`, `float`... là mối nguy trong nhúng?

Trong lập trình PC, `int` thường là 4 byte. Nhưng:

- Trên vi điều khiển AVR (Arduino Uno) – `int` là 2 byte.

- Trên ARM Cortex-M (STM32) – `int` là 4 byte.

Khi phát triển phần mềm cho ECU, dữ liệu phải được trao đổi giữa các cảm biến, vi điều khiển và bus **CAN** với độ chính xác tuyệt đối. Nếu bạn dùng `int` và "hy vọng" nó 4 byte, bạn có thể làm hỏng cấu trúc gói tin, gây lỗi hệ thống nghiêm trọng.

⇒ Nguyên tắc vàng: Luôn biết chính xác kích thước từng biến. Dùng thư viện `<stdint.h>` (chuẩn C99) để khai báo kiểu dữ liệu có kích thước cố định.

### 3.2 Bảng các kiểu dữ liệu cố định

| **Kiểu cũ (tránh dùng)** | **Kiểu mới (`stdint.h`)** | **Kích thước** | **Miền giá trị (không dấu / có dấu)** |
| --- | --- | --- | --- |
| `char` | `uint8_t` / `int8_t` | 1 byte | 0..255 / -128..127 |
| `short` | `uint16_t` / `int16_t` | 2 byte | 0..65,535 / -32,768..32,767 |
| `int`, `long` | `uint32_t` / `int32_t` | 4 byte | 0..4,294,967,295 / –2,147,483,648..2,147,483,647 |
| `long long` | `uint64_t` / `int64_t` | 8 byte | Rất lớn |

Quy ước đặt tên:

- `u` = unsigned (không dấu)

- `int` = integer

- `8/16/32/64` = số bit

- `_t` = type

### 3.3 Ví dụ minh họa và kiểm tra kích thước

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
  // Khai báo kiểu cố định – tiêu chuẩn nhúng
  uint8_t  so_xy_lanh   = 4;       // 1 byte
  int32_t  nhiet_do     = -40;     // 4 byte có dấu
  uint32_t toc_do_xe    = 180000;  // 4 byte không dấu

  // Kiểm tra kích thước thực tế trên môi trường hiện tại
  printf("Kich thuoc uint8_t : %d byte\n", (int)sizeof(uint8_t));
  printf("Kich thuoc int32_t : %d byte\n", (int)sizeof(int32_t));
  printf("Kich thuoc uint32_t: %d byte\n", (int)sizeof(uint32_t));

  // In giá trị
  printf("So xy lanh: %u\n", so_xy_lanh);
  printf("Nhiet do: %d\n", nhiet_do);
  printf("Toc do xe: %u\n", toc_do_xe);

  return 0;
}
```

**=> Giải thích chi tiết luồng chương trình chạy và tại sao có kết quả như vậy:**

<details> <summary><b>Xem giải thích</b></summary>

`#include <stdint.h>` cung cấp các định nghĩa kiểu như `uint8_t`, `int32_t`, v.v.

- `uint8_t so_xy_lanh = 4;` – dành ra 1 byte, lưu giá trị 4.

- `int32_t nhiet_do = -40;` – dành ra 4 byte, lưu giá trị -40 dưới dạng bù hai.

- `uint32_t toc_do_xe = 180000;` – dành ra 4 byte, lưu 180000.

Toán tử `sizeof` trả về kích thước (tính bằng byte) của kiểu/biến. Vì các kiểu này có kích thước cố định độc lập nền tảng, ta luôn nhận được kết quả:

```text
Kich thuoc uint8_t : 1 byte
Kich thuoc int32_t : 4 byte
Kich thuoc uint32_t: 4 byte
```

Tiếp theo, `printf` với các định dạng:

- `%u` dành cho unsigned int, ở đây ta truyền `so_xy_lanh` (`uint8_t`) tự động được thăng cấp thành `int`/`unsigned int`, in ra 4.

- `%d` cho `int`/`int32_t`, in ra -40.

- `%u` cho `uint32_t`, in ra 180000.

Không có gì bất ngờ vì kích thước đã được cố định.

</details>

> Bài tập nhỏ: Hãy thay `so_xy_lanh = 260` rồi chạy lại. Giá trị in ra là gì? Giải thích hiện tượng tràn số (overflow) trên kiểu `uint8_t`.

**=> Trả lời và giải thích:**

<details> <summary><b>Xem trả lời</b></summary>

Khi gán `so_xy_lanh = 260`, vì `uint8_t` chỉ có 8 bit (giá trị tối đa 255), số 260 (nhị phân 100000100 – 9 bit) sẽ bị cắt mất bit cao nhất, chỉ còn 8 bit thấp: `00000100` (4 ở thập phân). Do đó, chương trình in ra 4. Đây là hành vi tràn số (overflow) không được ngôn ngữ C kiểm tra; với số không dấu, nó wrap-around modulo 256. Trong ECU, điều này cực kỳ nguy hiểm – một giá trị cảm biến có thể bị sai hoàn toàn mà không có lỗi biên dịch.

</details>

## 4. Biến nằm trong RAM như thế nào?

### 4.1 Bản chất vật lý

Khi bạn khai báo:

```c
uint32_t toc_do_xe = 180000;
```

RAM sẽ:

1. Dành ra một khối 4 ô nhớ liên tiếp (mỗi ô 1 byte).

2. Khối này có một địa chỉ bắt đầu duy nhất, ví dụ `0x20001000`.

3. Giá trị `180000` (thập phân) được chuyển thành nhị phân: `0000 0000 0000 0010 1011 1111 1100 1000` (32 bit — **MSB** (Most Significant Byte/Bit) → **LSB** (Least Significant Byte/Bit)).

4. Các byte được lưu theo thứ tự **Little-Endian** (byte có trọng số thấp nhất lưu ở địa chỉ thấp nhất):

| Địa chỉ | Giá trị byte (hex) | Ghi chú |
| --- | --- | --- |
| 0x20001000 | 0xC8 | Byte thấp nhất |
| 0x20001001 | 0xBF | |
| 0x20001002 | 0x02 | |
| 0x20001003 | 0x00 | Byte cao nhất |

Hiểu được cách bố trí này cực kỳ quan trọng khi giao tiếp với phần cứng (ví dụ: đọc dữ liệu từ cảm biến qua **SPI**, hoặc đóng gói tín hiệu **CAN**).

### 4.2 Thí nghiệm nhỏ (tham khảo sau khi học con trỏ)

```c
uint32_t value = 0x12345678;
uint8_t *p = (uint8_t*)&value;
printf("%X %X %X %X\n", p[0], p[1], p[2], p[3]);
```

Kết quả (Little-Endian):

```bash
78 56 34 12
```

**=> Giải thích chi tiết luồng chương trình chạy và tại sao có kết quả như vậy:**

<details> <summary><b>Xem giải thích</b></summary>

- `value` là biến 4 byte, được gán giá trị hex `0x12345678`.

- `&value` lấy địa chỉ của `value` (kiểu `uint32_t*`).

- Ép kiểu (`uint8_t*`) biến địa chỉ đó thành con trỏ tới byte.

- `p[0]` trỏ tới byte đầu tiên tại địa chỉ thấp nhất. Trên kiến trúc Little-Endian (như x86 của PC), byte thấp nhất của `0x12345678` là `0x78` nằm ở địa chỉ thấp nhất. Do đó, `p[0]` = `0x78`, `p[1]` = `0x56`, `p[2]` = `0x34`, `p[3]` = `0x12`.

- Kết quả in ra “78 56 34 12” khớp với thứ tự Little-Endian.

Nếu chạy trên một vi điều khiển Big-Endian, thứ tự sẽ là “12 34 56 78”. Đây là lý do khi trao đổi dữ liệu qua mạng (CAN, Ethernet) ta cần quy ước endianness rõ ràng.

</details>

## 5. Ứng dụng trong ECU ô tô

### 5.1 Cảm biến và biến trong RAM

Trong một ECU thực tế, tín hiệu từ cảm biến (nhiệt độ, áp suất, tốc độ...) được đọc liên tục (vài mili-giây một lần) và lưu vào RAM dưới dạng các biến.

Tuy nhiên, chúng không được lưu trực tiếp dưới dạng `float` như suy nghĩ thông thường.

### 5.2 Vì sao hạn chế float trong ECU?

- Hầu hết vi điều khiển trong ô tô (đặc biệt các dòng tiết kiệm chi phí, độ an toàn cao) **không có FPU** (Floating-Point Unit) phần cứng.

- Các phép toán `float` nếu không có FPU sẽ được **mô phỏng bằng phần mềm**, tiêu tốn hàng trăm chu kỳ CPU.

- Trong hệ thống thời gian thực (phanh ABS, túi khí), việc chậm trễ vài micro-giây có thể gây hậu quả nghiêm trọng.

#### **Giải pháp**: Sử dụng **số nguyên** + tỉ lệ (**scaling**)

Ví dụ: thay vì `float dien_ap = 3.55;` (Volt), ta lưu `uint16_t dien_ap_mV = 3550;` (mili-Volt).

**=> Chưa hiểu thật sự cách dùng số nguyên + scaling trong code như nào?** Tư duy như nào khi gặp một giá trị cần xử lý để biến đổi thành số nguyên + scaling?

<details> <summary><b>Xem giải thích chi tiết</b></summary>

**Tư duy scaling:**

Bất kỳ đại lượng vật lý nào (điện áp, nhiệt độ, áp suất, tốc độ...) đều có một đơn vị cơ bản và một độ phân giải mong muốn. Ta sẽ chọn một đơn vị nhỏ hơn (ví dụ mV thay vì V, °C×100 thay vì °C) để biểu diễn bằng số nguyên.

**Quy trình:**

1. Xác định độ phân giải nhỏ nhất cần thiết (ví dụ: 0.01°C, 1 mV, 10 Pa).

2. Chọn đơn vị lưu trữ là bội số của độ phân giải đó:

   - Nếu muốn biểu diễn nhiệt độ với 2 chữ số thập phân, lưu giá trị = `nhiệt_độ_thực × 100` (kiểu `int16_t`).

   - Nếu điện áp cần chính xác tới mV, lưu giá trị = `điện_áp_V × 1000` (kiểu `uint16_t`).

3. Mọi tính toán (cộng, trừ, nhân, chia) đều thực hiện trên các giá trị nguyên này, cẩn thận với thứ tự phép toán để tránh tràn số hoặc mất độ chính xác. Luôn nhân trước, chia sau, và ép kiểu về dạng rộng hơn nếu cần.

4. Chỉ khi cần hiển thị hoặc truyền cho module khác, ta mới quy đổi ngược về đơn vị gốc bằng cách chia cho hệ số scale.

**Ví dụ thực hành tư duy:**

Cho cảm biến nhiệt độ:

- Ngõ ra điện áp 0..5V tương ứng -40°C..125°C.

- Độ phân giải yêu cầu: 0.1°C.

Ta sẽ:

- Lưu nhiệt độ trong biến `int16_t nhiet_do_x10;` (đơn vị 0.1°C).

- Công thức quy đổi từ điện áp (mV) sang `nhiet_do_x10` chỉ dùng số nguyên, tránh `float` (xem mục 5.3).

</details>

Quy đổi khi cần hiển thị/gửi lên mạng:

- Giá trị thực = `dien_ap_mV / 1000.0` (trên bộ phận hiển thị có FPU)

- Hoặc truyền nguyên giá trị nguyên qua **CAN** bus, bên nhận sẽ tự quy đổi.

Lợi ích:

- Tính toán nhanh, an toàn.

- Tiết kiệm tài nguyên CPU/RAM.

- Tránh được sai số tích lũy của số thực.

### 5.3 Ví dụ thực tế: Đọc nhiệt độ động cơ

```c
#include <stdint.h>

// Giả sử driver đọc ADC trả về giá trị 12-bit (0..4095)
// tương ứng với 0V..5V. Cảm biến nhiệt độ cho 10mV/°C, offset 500mV tại 0°C
// Ta sẽ tính toán hoàn toàn bằng số nguyên (đơn vị mV và °C * 100)

uint16_t adc_raw;          // giá trị thô từ ADC
uint16_t dien_ap_mV;       // đơn vị mV
int16_t nhiet_do_x100;    // nhiệt độ * 100 (để giữ 2 chữ số thập phân)

// Công thức (dùng số nguyên):
dien_ap_mV   = (uint16_t)(((uint32_t)adc_raw * 5000) / 4095);
nhiet_do_x100 = (int16_t)((dien_ap_mV - 500) * 10);  // vì 10mV/°C, nhân thêm 100 → *10
// Bây giờ nhiet_do_x100 = 450 nghĩa là 4.50°C
```

Không hề có `float` nào xuất hiện, nhưng vẫn đảm bảo độ chính xác và tốc độ.

**=> Số nguyên + scaling tại sao lại được xử lý như vậy?** Nên tư duy suy nghĩ như nào?

**=> Giải thích chi tiết luồng chương trình chạy và tại sao có kết quả như vậy:**

<details> <summary><b>Xem giải thích đầy đủ</b></summary>

**Tư duy thiết kế công thức số nguyên:**

**Bước 1 – Quy đổi ADC sang điện áp (mV):**

ADC 12-bit: giá trị 0..4095 biểu diễn tuyến tính điện áp 0..5000 mV.

→ Điện áp (mV) = `(ADC_raw × 5000) / 4095`.

Để tránh tràn số khi nhân `adc_raw` (tối đa 4095) với 5000 (≈20 triệu) vượt quá 65535 của `uint16_t`, ta ép `adc_raw` sang `uint32_t` trước khi nhân. Sau đó chia cho 4095, kết quả trong khoảng 0..5000, ép về `uint16_t`.

**Bước 2 – Quy đổi điện áp sang nhiệt độ:**

Cảm biến: 10mV/°C, offset 500mV tại 0°C.

→ Nhiệt độ (°C) = `(điện áp mV – 500) / 10`.

Muốn lưu với độ phân giải 0.01°C, ta nhân thêm 100:

```text
nhiet_do_x100 = ((dien_ap_mV - 500) × 100) / 10 = (dien_ap_mV - 500) × 10
```

(Chú ý: 100/10 = 10, rút gọn để tránh phép tính không cần thiết).

Vì `dien_ap_mV` trong khoảng 0..5000, hiệu số tối đa 4500, nhân 10 = 45000, vừa trong tầm `int16_t` (tối đa 32767) cho giá trị dương; tuy nhiên nếu offset thấp hơn điện áp, giá trị có thể âm, `int16_t` vẫn chứa được -32768..32767, đủ rộng.

**Chạy từng bước** (giả định `adc_raw = 2048`, khoảng 2.5V):

- `(uint32_t)adc_raw * 5000` = 2048 × 5000 = 10,240,000.

- Chia cho 4095 ≈ 2500.61, phần nguyên là 2500 (vì chia số nguyên cắt bỏ thập phân). Vậy `dien_ap_mV = 2500`.

- `dien_ap_mV - 500` = 2000.

- Nhân với 10: 2000 × 10 = 20000. Vậy `nhiet_do_x100 = 20000`, tương đương 200.00°C.

  - Ví dụ gốc nói `nhiet_do_x100 = 450` nghĩa là 4.50°C — đó là kịch bản khác: khi `dien_ap_mV - 500 = 45` mV → 45 × 10 = 450. Điều này xảy ra khi `adc_raw` rất nhỏ (điện áp ≈ 545mV).

**Kết luận:** Ta đã thay thế toàn bộ số thực bằng số nguyên, đảm bảo CPU thực hiện vài lệnh nhân/chia số nguyên trong vài chu kỳ, thay vì gọi thư viện giả lập `float` hàng trăm chu kỳ.

</details>

## 6. Tổng kết bài học

- Biến là vùng nhớ RAM có địa chỉ, kích thước và giá trị nhị phân.

- Trong hệ thống nhúng ô tô, **tuyệt đối sử dụng các kiểu dữ liệu cố định từ** `<stdint.h>` (`uint8_t`, `int32_t`...) để đảm bảo tính nhất quán trên mọi nền tảng.

- Hạn chế `float`, thay bằng số nguyên có tỉ lệ (**scaling**) để đáp ứng yêu cầu thời gian thực và an toàn.

- Mỗi kỹ sư cần có tư duy "từng byte một" và hiểu cách dữ liệu được lưu vật lý trong bộ nhớ.

## 7. Câu hỏi củng cố

1. `%c`, `%d`, `%f` trong `printf` khác nhau như thế nào?

**=> Trả lời:**

<details> <summary><b>Xem trả lời</b></summary>

- `%c` – định dạng ký tự (`char`): `printf` đọc 1 byte từ đối số, hiểu đó là mã ASCII và in ra ký tự tương ứng.

- `%d` – định dạng số nguyên có dấu (`int`): `printf` đọc 4 byte (hoặc kích thước của `int`), diễn giải dưới dạng số nguyên có dấu (bù hai) và in ra dạng thập phân.

- `%f` – định dạng số thực dấu phẩy động (`float`/`double`): `printf` đọc 8 byte (vì `float` được tự động thăng cấp lên `double` khi truyền vào hàm biến đối), diễn giải theo chuẩn IEEE 754 và in ra với 6 chữ số thập phân mặc định.

Chúng khác nhau ở cách diễn giải các byte trong RAM và cách hiển thị.

</details>

2. Tại sao viết `3.55f` chứ không phải `3.55`?

**=> Trả lời:**

<details> <summary><b>Xem trả lời</b></summary>

Trong C, hằng số thực không có hậu tố được mặc định là kiểu `double` (8 byte). Khi viết `float gpa = 3.55;`, trình biên dịch sẽ ngầm chuyển từ `double` về `float`, có thể gây cảnh báo mất độ chính xác. Viết `3.55f` khai báo rõ đây là hằng số kiểu `float` (4 byte), tránh chuyển đổi không cần thiết, phù hợp khi gán cho biến `float` hoặc khi tối ưu bộ nhớ trong nhúng.

</details>

3. `\n` ở cuối mỗi `printf` có tác dụng gì?

**=> Trả lời:**

<details> <summary><b>Xem trả lời</b></summary>

`\n` là ký tự xuống dòng (newline, mã ASCII 10). Nó đưa con trỏ xuống đầu dòng tiếp theo. Nếu không có `\n`, các dòng in ra sẽ nối liền nhau, khó đọc. Ngoài ra, với chế độ line-buffered của `stdout`, `\n` còn có tác dụng flush bộ đệm (đẩy dữ liệu ra thiết bị xuất ngay lập tức).

</details>

4. Chuyện gì sẽ xảy ra nếu cố nhét số `300` (`int`) vào 1 hộp `char`?

**=> Trả lời:**

<details> <summary><b>Xem trả lời</b></summary>

`char` chỉ có 1 byte (thường 8 bit), miền giá trị 0..255 hoặc -128..127. Khi gán `char c = 300;`, giá trị 300 (`0x12C`) bị cắt còn 8 bit thấp (`0x2C` = 44). Nếu `char` có dấu, 44 vẫn nằm trong tầm dương; nếu là `unsigned char`, cũng là 44. Nhưng nếu giá trị lớn hơn 255 và bị cắt, kết quả sẽ là `giá_trị % 256`. Điều này gây mất dữ liệu nghiêm trọng, không có lỗi biên dịch bắt buộc (có thể chỉ cảnh báo). Trong nhúng, đây là mối nguy hiểm thường trực khi không kiểm soát kích thước kiểu dữ liệu.

</details>

## 8. Bài tập về nhà

1. **Làm quen** `stdint.h`: Viết một chương trình khai báo các biến:

- `uint8_t` lưu số cửa sổ trời (giá trị 0-255)

- `int16_t` lưu nhiệt độ điều hòa (có thể âm)

- `uint32_t` lưu quãng đường xe đã đi (km)

In ra kích thước và giá trị của chúng. Thử nghiệm tràn số với từng kiểu.

**=> Trả lời:**

<details> <summary><b>Xem code mẫu và giải thích</b></summary>

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    uint8_t so_cua_so_troi = 5;        // 0..255
    int16_t nhiet_dieu_hoa = 24;      // có thể -40..+40
    uint32_t quang_duong = 150000;     // km

    printf("so_cua_so_troi: %u (size %d byte)\n", so_cua_so_troi, (int)sizeof(so_cua_so_troi));
    printf("nhiet_dieu_hoa: %d (size %d byte)\n", nhiet_dieu_hoa, (int)sizeof(nhiet_dieu_hoa));
    printf("quang_duong: %u (size %d byte)\n", quang_duong, (int)sizeof(quang_duong));

    // Thử nghiệm tràn số
    so_cua_so_troi = 260;               // vượt 255
    printf("Sau tran: so_cua_so_troi = %u\n", so_cua_so_troi); // in ra 4

    nhiet_dieu_hoa = 32767;             // max int16_t
    nhiet_dieu_hoa = nhiet_dieu_hoa + 1; // tràn
    printf("Sau tran: nhiet_dieu_hoa = %d\n", nhiet_dieu_hoa); // -32768

    quang_duong = 4294967295U;          // max uint32_t
    quang_duong = quang_duong + 1;      // tràn, wrap về 0
    printf("Sau tran: quang_duong = %u\n", quang_duong); // 0

    return 0;
}
```

**Giải thích hiện tượng tràn:**

- `uint8_t` (0..255) khi nhận 260: 260 % 256 = 4.

- `int16_t` (tối đa 32767) khi cộng thêm 1 thành 32768, vượt quá biểu diễn có dấu → gây tràn (undefined behavior trong C, nhưng thường wrap sang -32768 theo bù hai).

- `uint32_t` (0..4294967295) khi cộng 1 vào giá trị max → wrap về 0 (đảm bảo modulo 2³² với số không dấu).

Việc tràn số không được phần cứng báo lỗi, người lập trình phải tự kiểm soát.

</details>

2. **Tư duy scaling**: Một cảm biến áp suất lốp trả về giá trị ADC 10-bit (0..1023) tương ứng 0..5V. Cảm biến có độ nhạy 20mV/kPa, offset 100mV tại 0 kPa. Hãy viết công thức tính áp suất theo đơn vị Pa (Pascal) chỉ dùng số nguyên, không dùng `float`.

**=> Trả lời:**

<details> <summary><b>Xem lời giải và giải thích</b></summary>

**Bước 1 – Quy đổi ADC sang điện áp:**

ADC 10-bit → giá trị 0..1023 tương ứng 0..5000 mV.

```text
V_mV = (ADC_raw × 5000) / 1023
```

**Bước 2 – Quy đổi điện áp sang áp suất:**

Đặc tính cảm biến: cứ 20mV → 1 kPa; tại 0 kPa, điện áp là 100 mV.

```text
P_kPa = (V_mV - 100) / 20
```

Đổi sang Pascal (Pa): 1 kPa = 1000 Pa → `P_Pa = P_kPa × 1000`.

**Bước 3 – Gộp công thức số nguyên** (lưu ý thứ tự nhân/chia để giữ độ chính xác):

```text
P_Pa = ( (ADC_raw * 5000) / 1023 - 100 ) * 50
```

vì chia 20 sau đó nhân 1000 tương đương nhân với 1000/20 = 50. Nhưng phải cẩn thận với dấu ngoặc và kiểu dữ liệu.

**Triển khai code an toàn:**

```c
uint16_t adc_raw;        // 0..1023
uint16_t V_mV;           // 0..5000
int32_t P_Pa;            // áp suất Pascal (có thể âm nếu điện áp < offset)

V_mV = (uint16_t)(((uint32_t)adc_raw * 5000) / 1023);
// Trừ offset rồi nhân 50 (vì 1000/20 = 50)
P_Pa = ((int32_t)V_mV - 100) * 50;
```

Nếu cần độ phân giải cao hơn (ví dụ 0.1 Pa), có thể lưu `P_Pa_x10` và nhân thêm 10 ở khâu cuối. Tư duy scaling vẫn giữ nguyên.

</details>

3. **Khám phá Little-Endian**: Sử dụng con trỏ `uint8_t` để in ra từng byte của một biến `uint32_t` có giá trị `0xAABBCCDD`. Xác nhận thứ tự byte trên máy bạn.

**=> Trả lời:**

<details> <summary><b>Xem code và giải thích</b></summary>

```c
#include <stdio.h>
#include <stdint.h>

int main(void) {
    uint32_t value = 0xAABBCCDD;
    uint8_t *p = (uint8_t*)&value;
    printf("%X %X %X %X\n", p[0], p[1], p[2], p[3]);
    return 0;
}
```

Trên PC (x86-64, Little-Endian), kết quả sẽ là:

```text
DD CC BB AA
```

**Giải thích:**

- Byte thấp nhất `0xDD` được lưu ở địa chỉ thấp nhất (`p[0]`), kế đến `0xCC`, `0xBB`, `0xAA`.

- Điều này khẳng định máy dùng **Little-Endian**.

- Với các vi điều khiển Big-Endian (một số dòng PowerPC, SPARC), thứ tự sẽ là `AA BB CC DD`.

- Khi làm việc với **CAN**/**LIN**, chúng ta phải kiểm tra endianness của MCU và quy ước của tầng giao tiếp.

</details>
