# BT3_SQL_Server

<h3>Đề tài: Thiết kế và cài đặt CSDL quản lý cầm đồ </h3>

## Thông tin sinh viên:

* Họ và tên: Nguyễn Việt Long

* MSSV: K235480106047

* Lớp: K59KMT

* Giảng viên hướng dẫn: ThS. Đỗ Duy Cốp

## Nhiệm vụ 1 (file PDF)

## Nhiệm vụ 2 (file Script.sql)

Tạo mới database:

```
Create database QuanLyCamDo_K235480106047;
Go
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/dfb304f0-e0cd-4b4a-9d0c-b32d95d40af5" />

Tạo bảng và khởi tạo dữ liệu:

```
Use QuanLyCamDo_K235480106047;
Go

	-- 1. Tạo bảng Khách hàng
CREATE TABLE KhachHang (
    MaKH INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(255) NOT NULL,
    SoCCCD VARCHAR(20) UNIQUE NOT NULL, -- Định danh duy nhất
    SoDienThoai VARCHAR(15) UNIQUE NOT NULL,
    DiaChi NVARCHAR(255)
);

-- 2. Tạo bảng Hợp đồng
CREATE TABLE HopDong (
    MaHD INT IDENTITY(1,1) PRIMARY KEY,
    MaKH INT NOT NULL,
    NgayLap DATETIME DEFAULT GETDATE(),
    Deadline1 DATE NOT NULL,           -- Mốc thời gian chuyển từ lãi đơn sang lãi kép
    SoTienGoc DECIMAL(18, 2) NOT NULL,  -- Số tiền vay ban đầu
    TrangThai NVARCHAR(50) DEFAULT 'Đang vay', 
    FOREIGN KEY (MaKH) REFERENCES KhachHang(MaKH) ON DELETE CASCADE
);

-- 3. Tạo bảng Tài sản thế chấp (1 Hợp đồng có nhiều tài sản)
CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL,
    TenTaiSan NVARCHAR(255) NOT NULL,
    GiaTriUocTinh DECIMAL(18, 2),       -- Giá trị để so sánh khi trả hàng
    MoTa NVARCHAR(500),
    TrangThai NVARCHAR(50) DEFAULT 'Đang cầm cố',
    NgayCapNhat DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (MaHD) REFERENCES HopDong(MaHD) ON DELETE CASCADE
);

-- 4. Tạo bảng Log (Lưu lịch sử biến động trạng thái và tiền nợ)
CREATE TABLE LogBienDong (
    MaLog INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL,
    NgayBienDong DATETIME DEFAULT CURRENT_TIMESTAMP, -- Thời gian thực
    LoaiBienDong NVARCHAR(100),          -- 'Trả nợ gốc', 'Chốt lãi kép', 'Thanh lý'
    SoTienThayDoi DECIMAL(18, 2),       -- Số tiền biến động (+/-)
    DuNoHienTai DECIMAL(18, 2),          -- Tổng dư nợ thực tế sau biến động
    GhiChu NVARCHAR(500),
    FOREIGN KEY (MaHD) REFERENCES HopDong(MaHD) ON DELETE CASCADE
);
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/f6324f68-206a-4805-8891-04c64479e258" />

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/2b88aa0d-9376-4084-a752-11224798ee3f" />

Sơ đồ ERD

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/679f621c-92df-4963-86ba-3f773b9b846f" />

### Event 1: Đăng kí hợp đồng mới (vay tiền)

Viết Store Procedure tiếp nhận hợp đồng: Lưu thông tin khách hàng, danh sách tài sản (kèm giá trị định giá), số tiền vay gốc và thiết lập 2 mốc Deadline1, Deadline2.

Thêm Deadline 2 vào bảng HopDong:

```
Use QuanLyCamDo_K235480106047;
Go
ALTER TABLE HopDong ADD [Deadline 2] DATE NOT NULL;
```



