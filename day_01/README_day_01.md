# Bài 1: Tập trung vào cách quản lý biến trong RAM ECU

Một kỹ sư ô tô giỏi cần hiểu rõ từng byte dữ liệu để đảm bảo an toàn hệ thống.

## Chuẩn bị

- VS Code

- Extension:

  - C/C++

  - Coderunner

- Trình biên dịch

  - MSYS2 + GCC

## I. Khám phá "Hộp chứa giá trị" bên trong ECU

### 1. Biến là gì?

Biến = "Hộp chứa giá trị" trog RAM

Phân loại:

- char : 1 byte (Ký tự)

- int : 4 byte (Số nguyên)

- float : 4 byte (Số thực)

Phải nói rõ hộp to hay nhỏ (chỉ rõ kiểu dữ liệu)

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

> Liên kết thực tế: Trong ECU ô tô, mỗi cảm biến (nhiệt độ, tốc độ, áp suất) là 1 biến float được đọc và lưu vào RAM mỗi vài ms. Bài hôm này hôm nay chính là nền tảng của điều đó.
