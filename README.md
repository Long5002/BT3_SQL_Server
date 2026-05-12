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
    TrangThai NVARCHAR(50) DEFAULT N'Đang vay', 
    FOREIGN KEY (MaKH) REFERENCES KhachHang(MaKH) ON DELETE CASCADE
);

-- 3. Tạo bảng Tài sản thế chấp (1 Hợp đồng có nhiều tài sản)
CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT NOT NULL,
    TenTaiSan NVARCHAR(255) NOT NULL,
    GiaTriUocTinh DECIMAL(18, 2),       -- Giá trị để so sánh khi trả hàng
    MoTa NVARCHAR(500),
    TrangThai NVARCHAR(50) DEFAULT N'Đang cầm cố',
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
ALTER TABLE HopDong ADD [Deadline2] DATE NOT NULL;
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/5649280e-a443-4fcd-b831-f84a9761f54f" />

Viết Store Procedure:

```
CREATE TYPE Table_DanhSachTaiSan AS TABLE (
    TenTaiSan NVARCHAR(255),
    GiaTriUocTinh DECIMAL(18, 2),
    MoTa NVARCHAR(500)
);
GO

CREATE PROCEDURE sp_TiepNhanHopDong
    @MaKH INT,
    @SoTienGoc DECIMAL(18, 2),
    @Deadline1 DATE,
    @Deadline2 DATE,
    @DSTaiSan Table_DanhSachTaiSan READONLY
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;
    BEGIN TRY
        -- 1. Lưu thông tin hợp đồng
        INSERT INTO HopDong (MaKH, SoTienGoc, Deadline1, Deadline2, TrangThai)
        VALUES (@MaKH, @SoTienGoc, @Deadline1, @Deadline2, N'Đang vay');

        DECLARE @MaHD_Moi INT = SCOPE_IDENTITY();

        -- 2. Lưu danh sách tài sản
        INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriUocTinh, MoTa)
        SELECT @MaHD_Moi, TenTaiSan, GiaTriUocTinh, MoTa FROM @DSTaiSan;

        -- 3. Ghi Log khởi tạo
        INSERT INTO LogBienDong (MaHD, LoaiBienDong, SoTienThayDoi, DuNoHienTai, GhiChu)
        VALUES (@MaHD_Moi, N'Mở hợp đồng', @SoTienGoc, @SoTienGoc, N'Bắt đầu hợp đồng vay');

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/8784aa19-6158-457b-a813-e0b37db397d5" />

Test Event 1:


