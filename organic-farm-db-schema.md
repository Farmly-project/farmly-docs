# Thiết Kế Cơ Sở Dữ Liệu Chuẩn Cho Đồ Án Website Thương Mại Điện Tử Nông Sản Sạch

## 1. Tổng Quan & Lý Do Tái Thiết Kế Cơ Sở Dữ Liệu

Theo yêu cầu chính thức từ đồ án **Website Thương mại điện tử Nông sản sạch (Organic Farm Ecommerce - Mô hình Single Farm)**, hệ thống phải giải quyết bài toán cốt lõi của nông sản tươi sống: **minh bạch nguồn gốc (truy xuất QR Code), quản lý tồn kho theo Lô (Batch) theo nguyên tắc FEFO (Hết hạn trước - Xuất trước), xử lý xuất hủy/hao hụt nội bộ, mua hàng theo gói định kỳ (Subscription), và quy trình xử lý khiếu nại nông sản dập nát**.

Tập tệp sơ đồ cơ sở dữ liệu ban đầu trong `db.pdf` được thiết kế cho các sàn thương mại điện tử bán lẻ tổng hợp (như quần áo, hàng tiêu dùng đa thuộc tính), chứa nhiều bảng dư thừa không phù hợp (như quản lý biến thể kích thước/màu sắc `variant_option_values`, đa ngôn ngữ `locales`, `translations`), đồng thời **thiếu toàn bộ các bảng dữ liệu nghiệp vụ nông nghiệp và vận hành kho nông sản**.

Tài liệu này cung cấp thiết kế cơ sở dữ liệu PostgreSQL hoàn chỉnh, tối ưu hóa 100% cho triển khai trên nền tảng **Supabase Cloud**, phục vụ đầy đủ 5 vai trò người dùng hệ thống.

---

## 2. Bảng So Sánh Cấu Trúc: Sơ Đồ Cũ (db.pdf) vs. Sơ Đồ Mới Chuẩn Đồ Án

| Hạng mục / Module | Sơ đồ cũ trong `db.pdf` | Sơ đồ mới đã tối ưu (Supabase PostgreSQL) | Đánh giá & Lý do điều chỉnh |
| :--- | :--- | :--- | :--- |
| **Phân quyền người dùng** | Chỉ có 3 roles (`admin`, `customer`, `unregistered`). | Phân quyền 5 roles chuẩn: `guest`, `customer`, `farm_staff`, `sales`, `admin`. | Đáp ứng đúng phân tách vai trò nghiệp vụ vận hành nông trại thực tế. |
| **Quản lý Sản phẩm & Kho** | Quản lý kho tổng hợp theo sản phẩm/biến thể (`stock_quantity`, `variant_option_values`). | Quản lý tồn kho nghiêm ngặt theo **Lô hàng (`batches`)** dựa trên `harvest_date` & `exp_date`. | Bắt buộc cho mô hình nông sản nhằm thực hiện xuất kho theo quy tắc **FEFO**. |
| **Truy xuất Nguồn gốc** | Không có dữ liệu truy xuất hay mã QR. | Thêm bảng **`farming_logs`** (Nhật ký canh tác), **`quality_certifications`** (VietGAP/USDA), và **`qr_codes`**. | Cho phép khách quét mã QR xem toàn bộ dòng thời gian gieo trồng và chứng nhận. |
| **Vận hành Kho & Xử lý sự cố** | Không hỗ trợ ghi nhận hàng hư hỏng. | Thêm bảng **`inventory_disposals`** (Ghi nhận xuất hủy/hao hụt dập nát) và **`complaints`** (Khiếu nại đổi trả). | Phản ánh chính xác tỷ lệ hao hụt thực tế của nông sản tươi. |
| **Hình thức Bán hàng** | Chỉ mua lẻ từng đơn. | Thêm module **`subscriptions`** & **`subscription_items`** (Gói rau củ theo tháng/tuần). | Đáp ứng tính năng Should-have mua hàng định kỳ tự sinh đơn. |
| **Đa ngôn ngữ & Biến thể rườm rà** | Chứa `locales`, `product_translations`, `category_translations`, `variant_options`... | Loại bỏ toàn bộ các bảng dịch thuật và biến thể quần áo không cần thiết. | Đơn giản hóa cấu trúc CSDL, giảm tải cho sinh viên phát triển. |

---

## 3. Danh Sách Các Bảng Dữ Liệu Cốt Lõi Trong Cơ Sở Dữ Liệu Mới

Cơ sở dữ liệu mới bao gồm 16 bảng chính được chia thành 5 phân hệ nghiệp vụ:

### Phân hệ 1: Quản lý Người dùng & Phân quyền (Auth & Profiles)
1. **`profiles`**: Lưu thông tin mở rộng của người dùng (họ tên, số điện thoại, avatar, địa chỉ), liên kết trực tiếp với bảng `auth.users` của Supabase. Trường `role` nhận 1 trong 5 giá trị: `guest`, `customer`, `farm_staff`, `sales`, `admin`.

### Phân hệ 2: Danh mục Nông sản & Kiểm định Chất lượng
2. **`categories`**: Danh mục loại nông sản (Rau ăn lá, Củ quả, Trái cây tươi, Nấm hữu cơ...).
3. **`products`**: Thông tin sản phẩm nông sản (tên, mô tả, đơn vị tính: kg/gam/bó, giá niêm yết, thời gian bảo quản trung bình).
4. **`product_images`**: Bộ sưu tập hình ảnh thực tế của từng sản phẩm.
5. **`quality_certifications`**: Lưu thông tin và file scan chứng nhận chất lượng (VietGAP, GlobalGAP, USDA Organic, kiểm định mẫu đất/nước).

### Phân hệ 3: Nghiệp vụ Nông nghiệp, Lô hàng & Truy xuất QR (Core FEFO)
6. **`batches`**: Bảng quản lý Lô hàng thu hoạch. Lưu `batch_code`, `harvest_date`, `exp_date`, `initial_quantity`, `current_quantity`, `status` (`growing`, `harvested`, `in_stock`, `sold_out`, `expired`), `is_flash_sale`, `flash_sale_price`.
7. **`farming_logs`**: Nhật ký canh tác theo Lô hàng (ngày gieo hạt, bón phân, tưới nước, thu hoạch) đính kèm danh sách `media_urls` (ảnh/video). Do **Farm Staff** cập nhật.
8. **`qr_codes`**: Quản lý mã QR Code sinh tự động cho từng Lô hàng, lưu URL truy xuất nguồn gốc (`trace_link`) và lượt quét.
9. **`inventory_disposals`**: Ghi nhận số lượng xuất hủy nội bộ (do dập nát, hỏng hóc, hết hạn), lưu lý do và người thực hiện (`disposed_by`).

### Phân hệ 4: Giỏ hàng, Đơn hàng & Thanh toán
10. **`carts`** & **`cart_items`**: Giỏ hàng của người dùng/khách vãng lai.
11. **`orders`**: Đơn hàng mua sắm, lưu địa chỉ giao hàng, phí vận chuyển động (tính theo km), trạng thái đơn hàng (`pending`, `confirmed`, `processing`, `shipping`, `delivered`, `completed`, `cancelled`).
12. **`order_items`**: Chi tiết đơn hàng, liên kết trực tiếp với **`batch_id`** để trừ kho chính xác theo lô xuất.
13. **`payments`**: Giao dịch thanh toán online (PayOS, MoMo, VNPay, COD), lưu `payos_order_code`, `transaction_id`, `payment_status` (`unpaid`, `paid`, `failed`, `refunded`), `qr_code_url`.

### Phân hệ 5: Định kỳ (Subscription) & Khiếu nại Đổi trả
14. **`subscriptions`** & **`subscription_items`**: Quản lý gói đăng ký mua nông sản định kỳ theo tháng/tuần (ví dụ: giao 3kg rau vào Thứ 3 & Thứ 6). Tự động sinh đơn hàng theo lịch.
15. **`complaints`**: Đơn khiếu nại nông sản hỏng/dập nát do **Customer** gửi, đính kèm `media_urls` (ảnh/video bằng chứng). Do nhân viên **Sales** tiếp nhận và duyệt hoàn tiền (`refund_amount`) hoặc giao bù.

---

## 4. Kịch Bản SQL DDL Hoàn Chỉnh (PostgreSQL / Supabase Ready)

Dưới đây là mã SQL khởi tạo toàn bộ cấu trúc Cơ sở dữ liệu, Enum types, Khóa ngoại, và Trigger xuất kho FEFO tự động:

```sql
-- =============================================================================
-- KỊCH BẢN KHỞI TẠO CƠ SỞ DỮ LIỆU: THƯƠNG MẠI ĐIỆN TỬ NÔNG SẢN SẠCH
-- Hệ quản trị CSDL: PostgreSQL 15+ (Tối ưu hóa cho Supabase Cloud)
-- =============================================================================

-- 1. KHỞI TẠO CÁC ENUM TYPES
CREATE TYPE user_role_enum AS ENUM ('guest', 'customer', 'farm_staff', 'sales', 'admin');
CREATE TYPE batch_status_enum AS ENUM ('growing', 'harvested', 'in_stock', 'sold_out', 'expired');
CREATE TYPE order_status_enum AS ENUM ('pending', 'confirmed', 'processing', 'shipping', 'delivered', 'completed', 'cancelled');
CREATE TYPE payment_method_enum AS ENUM ('cod', 'payos', 'momo', 'vnpay', 'bank_transfer');
CREATE TYPE payment_status_enum AS ENUM ('unpaid', 'paid', 'failed', 'refunded');
CREATE TYPE subscription_status_enum AS ENUM ('active', 'paused', 'cancelled');
CREATE TYPE complaint_status_enum AS ENUM ('pending', 'approved', 'rejected', 'resolved');

-- 2. BẢNG PROFILES (Liên kết với auth.users của Supabase)
CREATE TABLE profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(150) NOT NULL,
    phone VARCHAR(20),
    avatar_url TEXT,
    address TEXT,
    role user_role_enum DEFAULT 'customer'::user_role_enum,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. BẢNG DANH MỤC NÔNG SẢN
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(120) UNIQUE NOT NULL,
    description TEXT,
    image_url TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. BẢNG SẢN PHẨM NÔNG SẢN
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    category_id INT REFERENCES categories(id) ON DELETE SET NULL,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(220) UNIQUE NOT NULL,
    unit VARCHAR(50) NOT NULL DEFAULT 'kg', -- kg, gram, bó, túi
    base_price DECIMAL(12, 2) NOT NULL,
    short_description TEXT,
    description TEXT,
    preservation_days INT DEFAULT 7, -- Số ngày bảo quản tiêu chuẩn
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 5. BẢNG HÌNH ẢNH SẢN PHẨM
CREATE TABLE product_images (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id) ON DELETE CASCADE,
    image_url TEXT NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE,
    display_order INT DEFAULT 0
);

-- 6. BẢNG CHỨNG NHẬN CHẤT LƯỢNG (VietGAP, USDA, Scan nghiệm thu đất/nước)
CREATE TABLE quality_certifications (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id) ON DELETE CASCADE,
    cert_name VARCHAR(150) NOT NULL, -- VietGAP, GlobalGAP, Organic USDA
    cert_number VARCHAR(100),
    issued_by VARCHAR(200),
    issue_date DATE,
    expiry_date DATE,
    file_url TEXT NOT NULL, -- Scan PDF/Ảnh lưu trên Supabase Storage
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 7. BẢNG LÔ HÀNG (BATCHES - QUẢN LÝ KHO THEO FEFO)
CREATE TABLE batches (
    id SERIAL PRIMARY KEY,
    batch_code VARCHAR(100) UNIQUE NOT NULL, -- VD: BATCH-20261111-RAU01
    product_id INT NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    harvest_date DATE NOT NULL,
    exp_date DATE NOT NULL,
    initial_quantity DECIMAL(10, 2) NOT NULL,
    current_quantity DECIMAL(10, 2) NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL,
    status batch_status_enum DEFAULT 'in_stock'::batch_status_enum,
    is_flash_sale BOOLEAN DEFAULT FALSE,
    flash_sale_price DECIMAL(12, 2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_exp_harvest CHECK (exp_date >= harvest_date),
    CONSTRAINT chk_qty CHECK (current_quantity >= 0)
);

-- INDEX TỐI ƯU TRUY VẤN FEFO (Sắp xếp theo hạn sử dụng tăng dần)
CREATE INDEX idx_batches_fefo ON batches(product_id, exp_date ASC, current_quantity) WHERE status = 'in_stock';

-- 8. BẢNG NHẬT KÝ CANH TÁC (FARMING LOGS)
CREATE TABLE farming_logs (
    id SERIAL PRIMARY KEY,
    batch_id INT NOT NULL REFERENCES batches(id) ON DELETE CASCADE,
    stage VARCHAR(100) NOT NULL, -- Gieo hạt, Bón phân hữu cơ, Tưới tiêu, Thu hoạch
    title VARCHAR(200) NOT NULL,
    description TEXT,
    media_urls JSONB DEFAULT '[]'::jsonb, -- Danh sách URL ảnh/video lưu trên Supabase Storage
    performed_by UUID REFERENCES profiles(id),
    logged_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 9. BẢNG MÃ QR TRUY XUẤT NGUỒN GỐC
CREATE TABLE qr_codes (
    id SERIAL PRIMARY KEY,
    batch_id INT UNIQUE NOT NULL REFERENCES batches(id) ON DELETE CASCADE,
    qr_code_url TEXT NOT NULL, -- Đường dẫn ảnh QR Code sinh tự động
    trace_link TEXT NOT NULL, -- Link web cho khách quét: /trace/BATCH-20261111-RAU01
    scan_count INT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 10. BẢNG XUẤT HỦY / HAO HỤT NỘI BỘ
CREATE TABLE inventory_disposals (
    id SERIAL PRIMARY KEY,
    batch_id INT NOT NULL REFERENCES batches(id) ON DELETE RESTRICT,
    quantity_disposed DECIMAL(10, 2) NOT NULL,
    reason TEXT NOT NULL, -- Dập nát trong vận chuyển nội bộ, Háo hụt tự nhiên, Hết hạn
    disposed_by UUID REFERENCES profiles(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 11. BẢNG ĐƠN HÀNG (ORDERS)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(100) UNIQUE NOT NULL,
    customer_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
    customer_name VARCHAR(150) NOT NULL,
    customer_phone VARCHAR(20) NOT NULL,
    shipping_address TEXT NOT NULL,
    distance_km DECIMAL(6, 2) DEFAULT 0.0, -- Khoảng cách tính ship động
    shipping_fee DECIMAL(12, 2) DEFAULT 0.0,
    subtotal DECIMAL(12, 2) NOT NULL,
    total_amount DECIMAL(12, 2) NOT NULL,
    order_status order_status_enum DEFAULT 'pending'::order_status_enum,
    payment_status payment_status_enum DEFAULT 'unpaid'::payment_status_enum,
    customer_note TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 12. BẢNG CHI TIẾT ĐƠN HÀNG (ORDER ITEMS - LIÊN KẾT TRỰC TIẾP LÔ FEFO)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id),
    batch_id INT REFERENCES batches(id), -- Lô hàng cụ thể xuất kho cho sản phẩm này
    product_name VARCHAR(200) NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL,
    quantity DECIMAL(10, 2) NOT NULL,
    subtotal DECIMAL(12, 2) NOT NULL
);

-- 13. BẢNG THANH TOÁN (PAYMENTS - TÍCH HỢP PAYOS / MOMO / VNPAY)
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    payment_method payment_method_enum NOT NULL,
    transaction_id VARCHAR(100),
    payos_order_code BIGINT,
    amount DECIMAL(12, 2) NOT NULL,
    status payment_status_enum DEFAULT 'unpaid'::payment_status_enum,
    qr_code_url TEXT, -- Link VietQR/PayOS QR code
    gateway_response JSONB,
    paid_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 14. BẢNG GÓI ĐỊNH KỲ (SUBSCRIPTIONS)
CREATE TABLE subscriptions (
    id SERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
    package_name VARCHAR(150) NOT NULL, -- VD: Gói Rau Xanh Gia Đình 1 Tháng
    delivery_days VARCHAR(50) NOT NULL, -- VD: 'TUE,FRI'
    status subscription_status_enum DEFAULT 'active'::subscription_status_enum,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    next_delivery_date DATE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 15. BẢNG CHI TIẾT SẢN PHẨM TRONG GÓI ĐỊNH KỲ
CREATE TABLE subscription_items (
    id SERIAL PRIMARY KEY,
    subscription_id INT NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id),
    quantity DECIMAL(10, 2) NOT NULL
);

-- 16. BẢNG KHIẾU NẠI & ĐỔI TRẢ (COMPLAINTS)
CREATE TABLE complaints (
    id SERIAL PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    customer_id UUID NOT NULL REFERENCES profiles(id),
    reason TEXT NOT NULL,
    media_urls JSONB DEFAULT '[]'::jsonb, -- Ảnh/Video sản phẩm dập hỏng từ khách
    status complaint_status_enum DEFAULT 'pending'::complaint_status_enum,
    refund_amount DECIMAL(12, 2) DEFAULT 0.0,
    sales_note TEXT,
    handled_by UUID REFERENCES profiles(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. Thao Tác Nghiệp Vụ Tự Động: Thuật Toán Xuất Kho FEFO

Để đảm bảo hệ thống tự động xuất các lô nông sản **Hết hạn trước - Xuất trước**, Backend Node.js hoặc Supabase Stored Procedure sẽ thực hiện truy vấn chọn lô hàng theo logic sau:

```sql
-- Hàm SQL tự động gợi ý Lô hàng xuất kho theo nguyên tắc FEFO cho 1 sản phẩm
CREATE OR REPLACE FUNCTION get_fefo_batch_for_product(
    p_product_id INT,
    p_required_qty DECIMAL
)
RETURNS TABLE (
    batch_id INT,
    batch_code VARCHAR,
    exp_date DATE,
    available_qty DECIMAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        b.id AS batch_id,
        b.batch_code,
        b.exp_date,
        b.current_quantity AS available_qty
    FROM batches b
    WHERE b.product_id = p_product_id
      AND b.status = 'in_stock'
      AND b.current_quantity > 0
      AND b.exp_date >= CURRENT_DATE
    ORDER BY b.exp_date ASC, b.created_at ASC;
END;
$$ LANGUAGE plpgsql;
```

---

## 6. Hướng Dẫn Triển Khai Trên Supabase

1. Mở trang quản trị dự án **Supabase Dashboard** -> vào phần **SQL Editor**.
2. Dán toàn bộ mã SQL ở **Mục 4 và Mục 5** vào trình soạn thảo và nhấn **Run**.
3. Vào mục **Table Editor** để kiểm tra 16 bảng dữ liệu vừa được tạo.
4. Vào mục **Storage** -> Tạo 2 Storage Buckets công khai (Public):
   * `farming-media`: Lưu hình ảnh/video nhật ký canh tác & file scan kiểm định VietGAP.
   * `complaint-media`: Lưu hình ảnh/video khiếu nại dập nát từ khách hàng.
