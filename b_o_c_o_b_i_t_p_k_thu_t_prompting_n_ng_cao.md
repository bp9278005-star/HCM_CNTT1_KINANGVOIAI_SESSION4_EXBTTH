# BÁO CÁO BÀI TẬP: THIẾT KẾ GIẢI PHÁP PROMPTING ỨNG DỤNG THỰC TẾ

**Chủ đề:** Ứng dụng kỹ thuật Chain-of-Thought (CoT), Prompt Chaining và xử lý ngoại lệ (đứt gãy dữ liệu) để tự động hóa quy trình xử lý đơn hàng và phản hồi khách hàng.

---

## PHẦN 1: PHÂN TÍCH BỐI CẢNH & THIẾT KẾ GIẢI PHÁP (I/O)

### 1. Bối cảnh thực tế
Hiện tại, cửa hàng thời trang/phụ kiện online (mình lấy ví dụ là **Shop Thời Trang Năng Động**) nhận được rất nhiều tin nhắn đặt hàng và khiếu nại từ khách hàng qua Fanpage/Zalo. Các tin nhắn này thường viết tắt, thiếu thông tin (quên ghi size, thiếu số điện thoại, sai mã giảm giá) hoặc gộp chung nhiều yêu cầu. 

Nếu dùng một câu lệnh (prompt) đơn lẻ bình thường, AI rất dễ bị "ảo giác" (tự bịa ra số điện thoại hoặc đoán mò size của khách) hoặc bỏ sót các đơn bị thiếu thông tin. Vì vậy, mình thiết kế một **chuỗi Prompt (Prompt Chaining)** kết hợp tư duy **từng bước (Chain-of-Thought)** để lọc, kiểm tra lỗi và phân loại xử lý tự động trước khi chuyển cho nhân viên chốt đơn.

### 2. Sơ đồ luồng xử lý (Prompt Chaining)
Quy trình được chia làm 2 mắt xích (2 Prompts nối tiếp nhau):
*   **Mắt xích 1 (Prompt 1 - Trích xuất & Kiểm tra Logic):** Đọc tin nhắn thô của khách, suy luận từng bước (CoT) để bóc tách thông tin, đối chiếu với danh sách sản phẩm/mã giảm giá và bắt lỗi dữ liệu (thiếu/sai). Đầu ra là một file JSON chuẩn.
*   **Mắt xích 2 (Prompt 2 - Xử lý đứt gãy & Soạn phản hồi):** Nhận đầu vào là JSON từ Prompt 1. Dựa vào trạng thái đơn hàng (`Hợp lệ` hay `Lỗi/Thiếu thông tin`) để tự động viết tin nhắn phản hồi phù hợp cho khách.

### 3. Thiết kế Đầu vào (Input) và Đầu ra (Output)

#### A. Đầu vào (Input)
1.  **Dữ liệu tin nhắn khách hàng:** Đoạn chat thô khách gửi (có thể lộn xộn, viết tắt, sai chính tả).
2.  **Dữ liệu tham chiếu của Shop (Context):**
    *   Danh sách sản phẩm còn hàng (Tên SP, Size hiện có, Giá).
    *   Danh sách mã khuyến mãi hợp lệ.

#### B. Đầu ra mong muốn (Output)
1.  **Output của Prompt 1 (Dạng JSON để hệ thống dễ đọc):**
    *   `khach_hang`: Tên và SĐT (nếu không có ghi `null`).
    *   `san_pham`: Tên SP, Size, Số lượng.
    *   `ma_giam_gia`: Mã khách nhập và trạng thái (`Hợp lệ` / `Không hợp lệ` / `Không có`).
    *   `trang_thai_don`: `DU_THONG_TIN` hoặc `THIEU_THONG_TIN`.
    *   `danh_sach_loi`: Liệt kê cụ thể thiếu gì hoặc sai gì.
2.  **Output của Prompt 2 (Dạng văn bản gửi khách):**
    *   Tin nhắn phản hồi tự nhiên, lịch sự, xác nhận đơn (nếu đủ) hoặc hỏi bổ sung đúng phần bị thiếu/sai (nếu lỗi).

### 4. Dự trù bẫy dữ liệu & Cách xử lý đứt gãy
*   **Bẫy 1: Khách quên SĐT hoặc SĐT không đủ 10 số.** -> *Xử lý:* Ép AI kiểm tra độ dài chuỗi số, nếu không đủ 10 số bắt đầu bằng số 0 thì đánh dấu lỗi `SĐT không hợp lệ`, tuyệt đối không tự bịa số.
*   **Bẫy 2: Khách đặt size hết hàng hoặc không ghi size.** -> *Xử lý:* Đối chiếu với kho hàng ở Input. Nếu thiếu hoặc hết, báo lỗi cụ thể để Prompt 2 hỏi lại khách.
*   **Bẫy 3: Khách nhập mã giảm giá hết hạn/sai.** -> *Xử lý:* Không hủy đơn, chỉ đánh dấu mã không hợp lệ và báo lại giá gốc cho khách biết.

---

## PHẦN 2: VIẾT PROMPT HOÀN CHỈNH

### Prompt 1: Phân tích logic và bóc tách dữ liệu (Áp dụng Chain-of-Thought)

```text
Bạn là trợ lý xử lý dữ liệu đơn hàng của Shop Thời Trang Năng Động. Nhiệm vụ của bạn là đọc tin nhắn của khách hàng, đối chiếu với dữ liệu kho và trích xuất thông tin ra định dạng JSON.

### DỮ LIỆU KHO HÀNG & KHUYẾN MÃI HIỆN TẠI:
- Sản phẩm 1: Áo thun Basic (Còn size: M, L) - Giá: 150.000đ
- Sản phẩm 2: Quần Jean ống rộng (Còn size: S, M) - Giá: 320.000đ
- Mã giảm giá hợp lệ: "CHAOBAN" (Giảm 20.000đ), "FREESHIP" (Miễn phí vận chuyển).

### QUY TẮC XỬ LÝ (BẮT BUỘC TUÂN THỦ):
1. Tuyệt đối KHÔNG tự suy đoán hoặc bịa đặt thông tin nếu khách không cung cấp. Nếu thiếu, gán giá trị là null.
2. Số điện thoại hợp lệ phải có đúng 10 chữ số và bắt đầu bằng số 0.
3. Kiểm tra kỹ size khách đặt có nằm trong danh sách "Còn size" hay không.

### HƯỚNG DẪN SUY LUẬN TỪNG BƯỚC (CHAIN-OF-THOUGHT):
Trước khi xuất JSON, hãy tự suy nghĩ trong thẻ <thinking> theo các bước sau:
- Bước 1: Tìm thông tin định danh (Tên, SĐT) và kiểm tra tính hợp lệ của SĐT.
- Bước 2: Xác định sản phẩm, số lượng, size khách muốn mua. Đối chiếu với Dữ liệu kho xem còn hàng không.
- Bước 3: Kiểm tra mã giảm giá khách nhắc tới có nằm trong danh sách hợp lệ không.
- Bước 4: Tổng hợp các điểm bị thiếu hoặc sai lệch. Nếu có bất kỳ lỗi nào (thiếu SĐT, sai SĐT, thiếu size, hết size), trạng thái đơn phải là "CAN_XU_LY_LAI". Nếu mọi thông tin đặt hàng cơ bản (SĐT, Tên SP, Size) đều đúng và đủ, trạng thái là "HOP_LE" (lưu ý: sai mã giảm giá vẫn tính là HOP_LE nhưng cần ghi chú lỗi mã).

### ĐỊNH DẠNG ĐẦU RA:
Chỉ trả về phần suy luận trong thẻ <thinking> và kết quả cuối cùng dưới dạng một khối mã JSON duy nhất có cấu trúc:
{
  "ten_khach": string hoặc null,
  "so_dien_thoai": string hoặc null,
  "chi_tiet_don": [
    {
      "ten_sp": string,
      "size": string hoặc null,
      "so_luong": number
    }
  ],
  "ma_giam_gia": {
    "ma_nhap": string hoặc null,
    "trang_thai": "Hợp lệ" | "Không hợp lệ" | "Không có"
  },
  "trang_thai_don": "HOP_LE" | "CAN_XU_LY_LAI",
  "ghi_chu_loi": [danh sách các lỗi cụ thể bằng tiếng Việt]
}

### TIN NHẮN CỦA KHÁCH HÀNG:
"{Noi_dung_tin_nhan}"
```

### Prompt 2: Xử lý rẽ nhánh và soạn tin nhắn phản hồi (Chaining từ Prompt 1)

```text
Bạn là nhân viên chăm sóc khách hàng thân thiện của Shop Thời Trang Năng Động. Dựa vào dữ liệu JSON phân tích đơn hàng dưới đây, hãy viết một tin nhắn phản hồi cho khách hàng.

### KỊCH BẢN PHẢN HỒI:
- Trường hợp 1: Nếu "trang_thai_don" là "HOP_LE":
  + Cảm ơn khách đã đặt hàng.
  + Xác nhận lại tên sản phẩm, size, số lượng và số điện thoại nhận hàng.
  + Nếu "ma_giam_gia.trang_thai" là "Không hợp lệ", hãy khéo léo báo cho khách biết mã đó đã hết hạn/không đúng, nên đơn sẽ tính theo giá gốc, hỏi khách có đồng ý lên đơn không.
- Trường hợp 2: Nếu "trang_thai_don" là "CAN_XU_LY_LAI":
  + Đọc kỹ mảng "ghi_chu_loi".
  + Lịch sự xin lỗi vì sự bất tiện và hỏi khách bổ sung/điều chỉnh chính xác những thông tin đang bị thiếu hoặc sai trong "ghi_chu_loi" (ví dụ: hết size thì gợi ý size còn trong kho, thiếu/sai SĐT thì xin lại SĐT).
  + Giữ giọng văn tự nhiên, ngắn gọn, dùng xưng hô "Shop" và "bạn".

### DỮ LIỆU JSON ĐẦU VÀO:
{Output_JSON_tu_Prompt_1}
```

---

## PHẦN 3: KẾT QUẢ CHẠY THỬ (TESTING)

Mình tiến hành chạy thử với **1 trường hợp bẫy dữ liệu phức tạp** (Khách viết tắt, thiếu số điện thoại, đặt size đã hết hàng và dùng mã giảm giá sai).

*   **Input tin nhắn khách hàng:** 
    > *"Shop ơi cho mình lấy 1 áo thun basic size S với 1 quần jean ống rộng size M nhé. Áp mã GIAM50K cho mình. Sđt mình là 091234567, tên Lan."*
    *(Phân tích bẫy: SĐT chỉ có 9 số -> Lỗi; Áo thun Basic kho chỉ còn M, L nhưng khách đặt S -> Hết hàng/Lỗi; Mã GIAM50K không có trong hệ thống -> Sai mã).*

### 1. Kết quả chạy thử Prompt 1 (AI phản hồi)

```xml
<thinking>
- Bước 1: Khách tên "Lan". Số điện thoại cung cấp là "091234567". Đếm độ dài: có 9 chữ số. Theo quy tắc, SĐT phải có 10 chữ số -> Lỗi SĐT thiếu số.
- Bước 2: 
  + Sản phẩm 1: "áo thun basic", số lượng 1, size "S". Đối chiếu kho: Áo thun Basic chỉ còn size M, L. Size S đã hết -> Lỗi hết size S (còn M, L).
  + Sản phẩm 2: "quần jean ống rộng", số lượng 1, size "M". Đối chiếu kho: Quần Jean ống rộng còn size S, M -> Hợp lệ.
- Bước 3: Mã giảm giá khách nhập là "GIAM50K". Đối chiếu danh sách hợp lệ ("CHAOBAN", "FREESHIP") -> Mã không hợp lệ.
- Bước 4: Vì có lỗi sai SĐT và hết size sản phẩm nên trạng thái đơn là "CAN_XU_LY_LAI".
</thinking>
```

```json
{
  "ten_khach": "Lan",
  "so_dien_thoai": "091234567",
  "chi_tiet_don": [
    {
      "ten_sp": "Áo thun Basic",
      "size": "S",
      "so_luong": 1
    },
    {
      "ten_sp": "Quần Jean ống rộng",
      "size": "M",
      "so_luong": 1
    }
  ],
  "ma_giam_gia": {
    "ma_nhap": "GIAM50K",
    "trang_thai": "Không hợp lệ"
  },
  "trang_thai_don": "CAN_XU_LY_LAI",
  "ghi_chu_loi": [
    "Số điện thoại 091234567 bị thiếu số (chỉ có 9 số, cần đủ 10 số).",
    "Áo thun Basic hiện đã hết size S (kho chỉ còn size M và L).",
    "Mã giảm giá GIAM50K không hợp lệ hoặc đã hết hạn."
  ]
}
```

### 2. Kết quả chạy thử Prompt 2 (Lấy JSON trên đưa vào Prompt 2)

> Chào bạn Lan, cảm ơn bạn đã quan tâm và đặt hàng tại Shop Thời Trang Năng Động ạ! 
> 
> Hiện tại Shop đã ghi nhận mẫu **Quần Jean ống rộng (size M)** của bạn, tuy nhiên đơn hàng của mình đang gặp một vài vấn đề nhỏ nên Shop chưa lên đơn ngay được, bạn kiểm tra giúp Shop nhé:
> 1. **Về sản phẩm:** Mẫu **Áo thun Basic** bên Shop hiện đang tạm hết size S rồi ạ, chỉ còn sẵn **size M và L**, không biết bạn có muốn đổi sang size M không ạ?
> 2. **Về số điện thoại:** Số `091234567` bạn gửi đang bị thiếu mất 1 số (mới có 9 số), bạn nhắn lại giúp Shop số chính xác để shipper liên hệ nhé.
> 3. **Về mã giảm giá:** Mã `GIAM50K` hiện không áp dụng được ạ, bạn có thể đổi sang dùng mã `CHAOBAN` (giảm 20k) hoặc `FREESHIP` của Shop nhé!
> 
> Bạn phản hồi lại các thông tin trên để Shop chốt đơn sớm cho mình nha!

---

## PHẦN 4: ĐÁNH GIÁ & RÚT KINH NGHIỆM

1.  **Hiệu quả của Chain-of-Thought:** Việc bắt AI suy nghĩ trong thẻ `<thinking>` trước khi xuất JSON giúp giảm hẳn việc AI bỏ sót lỗi. Ở ví dụ trên, AI đã đếm chính xác độ dài số điện thoại (9 số) thay vì lướt qua như cách hỏi thông thường.
2.  **Hiệu quả của Prompt Chaining:** Tách riêng bước *bóc tách dữ liệu* và bước *giao tiếp khách hàng* giúp kiểm soát luồng đi của dữ liệu rất chặt chẽ. Nếu sau này đưa vào code thực tế (như Python hoặc Node.js), lập trình viên có thể dễ dàng lấy chuỗi JSON ở Bước 1 để lưu vào Database trước khi gọi API cho Bước 2.