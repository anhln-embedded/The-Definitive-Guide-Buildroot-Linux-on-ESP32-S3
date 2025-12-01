# The Definitive Guide: Buildroot Linux on ESP32-S3

## Complete Handbook for Porting Linux Kernel 6.x (Xtensa FDPIC)

Tài liệu này là hướng dẫn đầy đủ – từ A đến Z – giúp bạn xây dựng, flash và vận hành Embedded Linux (Buildroot) trên vi điều khiển ESP32-S3, sử dụng kiến trúc Xtensa LX7 No-MMU cùng định dạng ELF đặc biệt FDPIC.

## 🔄 Boot Process Overview

### Khi bạn nhấn nút Reset, điều gì sẽ xảy ra bên trong con chip?

1. **ROM Bootloader**: (Cứng trong chip) chạy đầu tiên, tải Bootloader từ Flash (0x0).

2. **ESP-IDF Bootloader**: Khởi tạo RAM, Flash, đọc Partition Table.

3. **App (Network Adapter)**:

   - Khởi tạo WiFi Driver (chạy trên Core 0).
   - Tìm phân vùng `linux`.
   - Nhảy (Jump) tới địa chỉ Kernel và trao quyền điều khiển cho Linux (chạy trên Core 1).

4. **Linux Kernel**:

   - Bung lụa! Khởi tạo driver UART, Filesystem.
   - Mount phân vùng `rootfs`.
   - Chạy tiến trình đầu tiên: `/sbin/init`.

5. **Init Process**:

   - Đọc cấu hình `/etc/inittab`.
   - Chạy các script khởi động (`/etc/init.d/...`).
   - Chạy script của bạn (`/etc/profile.d/sysinfo.sh`).

6. **Login Prompt**: Hiện ra `buildroot login:` và chờ bạn nhập `root`.

## 📌 1. System Overview

| Thành phần      | Thông số                                       |
| --------------- | ---------------------------------------------- |
| CPU             | ESP32-S3 Xtensa LX7 (Little Endian, 32-bit)    |
| MMU             | Không có (No-MMU, chỉ hỗ trợ FDPIC ELF)        |
| Flash đề nghị   | 16MB                                           |
| Toolchain       | Xtensa uClibc FDPIC (crosstool-NG + dynconfig) |
| Kernel          | Linux 6.x FDPIC Patch                          |
| Root Filesystem | Buildroot (CRAMFS cho RO + JFFS2 cho RW)       |

## 🎯 2. OPTION 1 — QUICK START

Dành cho người muốn chạy Linux ngay lập tức bằng binaries có sẵn

### Điểm mới

🔄 Toàn bộ Bootloader + Partition Table + Firmware network_adapter đã được gộp thành 1 file duy nhất: `firmware_merged.bin`  
➡️ Nạp duy nhất 1 lần tại địa chỉ 0x0000 bằng `esptool.py`.

### 2.1. Chuẩn bị

Chuẩn bị các file sau (download từ bạn cung cấp):

```text
firmware_merged.bin   <-- (bootloader + partition table + network_adapter)
xipImage              <-- Linux Kernel (execute-in-place)
rootfs.cramfs         <-- Root filesystem (read-only)
etc.jffs2             <-- RW overlay config partition
```

Export ESP-IDF:

```bash
. $IDF_PATH/export.sh
```

### 2.2. Flash firmware gộp (DUY NHẤT 1 FILE)

```bash
export ESP_PORT=COMx

esptool.py --chip esp32s3 --port $ESP_PORT --baud 460800 \
  write_flash 0x0 firmware_merged.bin
```

### 2.3. Flash Linux Kernel + RootFS

```bash
parttool.py -p $ESP_PORT write_partition --partition-name linux --input xipImage
parttool.py -p $ESP_PORT write_partition --partition-name rootfs --input rootfs.cramfs
parttool.py -p $ESP_PORT write_partition --partition-name etc --input etc.jffs2
```

### 2.4. Khởi động Linux

```bash
idf.py monitor
```

Khi xuất hiện:

```text
buildroot login:
```

→ Gõ `root`.

## 🛠️ 3. OPTION 2 — BUILD FROM SOURCE

### Full Toolchain → Kernel → RootFS pipeline

## 🏗️ PHASE 1 — Build Xtensa FDPIC Toolchain

### 3.1. Chuẩn bị dynconfig

```bash
cd ~/mcu
git clone https://github.com/jcmvbkbc/xtensa-dynconfig -b original
git clone https://github.com/jcmvbkbc/config-esp32s3 esp32s3
make -C xtensa-dynconfig ORIG=1 CONF_DIR=`pwd` esp32s3.so
```

### 3.2. Clone crosstool-NG FDPIC

```bash
git clone https://github.com/jcmvbkbc/crosstool-NG.git -b xtensa-fdpic
```

### 3.3. Fix endianness (BẮT BUỘC)

```bash
export XTENSA_GNU_CONFIG=$HOME/mcu/xtensa-dynconfig/esp32s3.so
```

### 3.4. Build toolchain

```bash
cd crosstool-NG
./bootstrap && ./configure --enable-local && make
./ct-ng xtensa-esp32s3-linux-uclibcfdpic
./ct-ng build
```

Toolchain sẽ xuất hiện tại:

```text
~/mcu/crosstool-NG/builds/xtensa-esp32s3-linux-uclibcfdpic
```

## 🏗️ PHASE 2 — Build Linux Kernel + RootFS với Buildroot

### 4.1. Clone Buildroot FDPIC

```bash
cd ~/mcu
git clone https://github.com/jcmvbkbc/buildroot -b xtensa-2024.02-fdpic
make -C buildroot O=`pwd`/build-buildroot-esp32s3 esp32s3_defconfig
```

### 4.2. Menuconfig (Kết nối toolchain + chọn packages)

```bash
make -C buildroot O=`pwd`/build-buildroot-esp32s3 menuconfig
```

**Thiết lập quan trọng:**

#### Toolchain

- Set External Toolchain → trỏ đến folder CT-NG

#### Packages

- Python3
- SQLite
- Dropbear (SSH)

### 4.3. Áp dụng FIXES QUAN TRỌNG

#### Fix 1 — fork() failure / No-MMU limitation

```bash
echo "PCRE2_CONF_OPTS += --disable-pcre2grep" >> buildroot/package/pcre2/pcre2.mk
```

#### Fix 2 — GCC Internal Compiler Error (Xtensa FDPIC bug)

```bash
sed -i '1i override CFLAGS += -O0' buildroot/ports/unix/Makefile
```

### 4.4. Build hệ thống

```bash
make -C buildroot O=`pwd`/build-buildroot-esp32s3 clean
make -j$(nproc) -C buildroot O=`pwd`/build-buildroot-esp32s3
```

Kết quả sẽ tạo ra:

```text
xipImage
rootfs.cramfs
etc.jffs2
```

## 🚀 PHASE 3 — Flash & Run Linux (Full pipeline)

**Flash firmware gộp:**

```bash
esptool.py write_flash 0x0 firmware_merged.bin
```

**Flash Linux:**

```bash
parttool.py -p $ESP_PORT write_partition --partition-name linux --input xipImage
parttool.py -p $ESP_PORT write_partition --partition-name rootfs --input rootfs.cramfs
parttool.py -p $ESP_PORT write_partition --partition-name etc --input etc.jffs2
```

## ✔️ 5. Verification Checklist

| Kiểm tra       | Lệnh                  |
| -------------- | --------------------- |
| Python OK      | `python3 --version`   |
| SQLite OK      | `sqlite3 --version`   |
| WiFi OK        | `iw dev espsta0 link` |
| Flash mount OK | `df -h`               |
| SSH OK         | `dropbear`            |
