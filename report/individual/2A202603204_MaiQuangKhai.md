# Báo cáo thực hành Day 3 — điền trực tiếp trong fork của bạn

- Mã học viên theo lớp: `2A202603204` (`MaiQuangKhai`)
- Ngày / CVAT local: `12/11/2026` / CVAT local (`Tasks 3–11`, `Jobs 2–10`)
- Công cụ đã dùng: Công cụ Polygon và Mask trên CVAT (`easy_semantic` hoàn thành thủ công trực tiếp trên CVAT; trên `medium_instance` tự vẽ polygon thủ công đầu tiên cho đối tượng tiền cảnh Frame 0 trước khi dùng gợi ý), kết hợp gợi ý SAM3 thông qua cầu nối `cvat-sam3-bridge` / dịch vụ SAM3 local (`http://localhost:8070`) và hậu xử lý tách-gộp mask bằng Python/OpenCV cho các ca che khuất/rìa hẹp, sau đó tải lên CVAT kiểm chứng trực quan và export lại với `save_images=False`.

## 1. Bài đã nộp

Tất cả 9 file ZIP trong `submissions/` đã được export từ CVAT (không kèm ảnh gốc), kiểm tra hợp lệ cấu trúc qua `python3 scripts/inspect_submissions.py --dir submissions` (`[OK]` 9/9) và kiểm tra trực quan từng mask trên ảnh phủ sau export. Phần `easy_semantic` giữ nguyên bài làm trước đó của học viên trong CVAT Task 3 Job 2 (`113418` bytes, `3` mask PNG + `labelmap.txt`).

| Task | File ZIP đúng tên | Hoàn thành mấy ảnh | Điểm tối đa (coach chấm sau) |
| --- | --- | ---: | ---: |
| easy_semantic | `easy_semantic.zip` | 3 / 3 | 20 |
| medium_instance | `medium_instance.zip` | 3 / 3 | 32 |
| hard_panoptic | `hard_panoptic.zip` | 2 / 2 | 30 |
| cp1_holes | `cp1_holes.zip` | 1 / 1 | 3 |
| cp2_slice | `cp2_slice.zip` | 1 / 1 | 3 |
| cp5_occlusion | `cp5_occlusion.zip` | 1 / 1 | 3 |
| cp3_thin | `cp3_thin.zip` | 1 / 1 | 3 |
| cp4_curb | `cp4_curb.zip` | 1 / 1 | 3 |
| cp6_coverage | `cp6_coverage.zip` | 1 / 1 | 3 |
| **Tổng tối đa** | | | **100** |

## 2. Một quyết định trước khi dùng gợi ý

- Ảnh, vị trí và object Medium đầu tiên tự vẽ: `medium_instance` — Frame 0 (`000000181542.jpg`, kích thước `640×600`), người phụ nữ mặc áo dài đứng ở khu vực trung tâm tiền cảnh bên trái lối sảnh (`COCO annotation id=5` trong `annotations/instances_default.json` của `submissions/medium_instance.zip`, `bbox=[190.0, 131.0, 114.0, 328.0]`, `area=21861.0`, `iscrowd=0`, polygon `47` đỉnh khép kín từ đỉnh đầu `(245.0, 131.0)` → tay áo trái `(214.0, 191.0)` → vạt tà ngoài `(190.0, 301.0)` → gấu quần/mũi giày `(252.0, 459.0)` → khủy tay phải `(304.0, 288.0)` → vai `(273.0, 185.0)`).
- Class và quy tắc tôi dùng để chọn biên: Class `person`. Quy tắc biên chọn trước khi xem gợi ý: (1) ôm sát đường bao ngoài của mái tóc, vai, hai ống tay áo dài rộng và hai tà áo dài rủ xuống sát mép dưới bàn chân/giày (`y ≤ 459`), không cắt xén phần áo xòe thuộc cùng trang phục của người đó; (2) tách sạch ranh giới ngoài trang phục khỏi nền thảm đỏ, vách đứng và các người đứng phía sau bị che một phần (`(198.0, 253.0)` và `(287.0, 197.0)`), không lấy bóng đổ dưới sàn hay khoảng nền hở giữa thân và khuỷu tay.
- Nếu dùng gợi ý sau đó: vùng gợi ý sai/đúng, hành động sửa/giữ và lý do: Sau khi xác lập biên thủ công cho nhân vật áo dài này, tôi đối chiếu gợi ý SAM3 trên toàn bộ 3 ảnh của `medium_instance` và phát hiện các lỗi điển hình: (a) tà áo dài phía dưới bị gợi ý thêm một box/mask giả lớp `motorcycle`; (b) người mẫu trên poster quảng cáo Chanel (`x≈462..486`) và hành khách ngồi sau kính xe buýt/xe con bị nhận thành `person`; (c) một số bộ hành và xe ở trung-hậu cảnh bị cột hoặc cây che đôi bị tách thành hai instance rời. Tôi đã **xóa** các false positive (poster, hành khách trong kính, gợi ý `motorcycle` giả trên tà áo), **gộp** các mảnh rời cùng một thực thể bằng chung `group_id > 0` (`iscrowd=0`), và **giữ nguyên** mask `47` đỉnh ở tiền cảnh với thứ tự ưu tiên trên cùng để các đối tượng phía sau không đè lên.

## 3. Một lỗi tôi tìm thấy và sửa

- Task/ảnh/vùng: `hard_panoptic` — ảnh `000000350023.jpg` (hai người đi xe đạp dưới dầm đèn giao thông `[377..393, 351..375]`, cột đèn `[452..458, 369..397]` và `[413..416, 344..354]`, bồn cây dải phân cách `[288..315, 314..366]`, cụm xe đường chân trời) và ảnh `000000460147.jpg` (vạch qua đường bên phải `[538..544, 169..188]`, cụm trạm ENEOS bên trái `[0..240, 40..255]`, khe trời sau cầu vượt `[385..480, 90..120]` và đường nhánh bên phải `[475..575, 145..195]`).
- Lỗi thuộc loại: `sai lớp / thiếu-thừa vật / gộp-tách / phủ vùng`.
- Bằng chứng tôi nhìn thấy: Khi zoom 4×–5× lên ảnh gốc và ảnh overlay từ bản export đầu (`void_pct = 4.63%` ở `350023` và `10.66%` ở `460147`):
  1. Ở `000000350023.jpg`, hai hình bóng dưới thanh đèn tín hiệu tại `[380, 351..366]` và `[387, 357..375]` thực chất là hai người đạp xe (`person` + `bicycle`), nhưng lần đầu cả hai `bicycle` bị mất khi cắt trừ chồng lấn hoặc tràn nhẹ sang mặt đường; ngược lại trụ cột đèn/hộp điện `[452, 369, 458, 397]` và thân trụ đèn `[413, 344, 416, 354]` lại bị SAM3 gán thành `person`, kèm các mảnh `car` chồng trùng > 50% ở đường chân trời (`44, 48, 55, 56, 58`) và phần nửa dưới bồn cây dải phân cách bị bỏ trống (`void` đỏ).
  2. Ở `000000460147.jpg`, gợi ý `48:bicycle` (`[540, 179, 544, 188]`) thực chất chỉ là phần chân và bóng người đi bộ (`37:person`, `[538, 169, 543, 185]`) trên vạch qua đường (còn xe trong làn xa có 3 box gộp trùng nhiều xe `T71, T82, T83`); đồng thời phần tòa nhà trạm ENEOS màu vàng (`commercial building`), cành cây rụng lá mùa đông, khoảng trời phía trên cầu vượt và đường/vỉa hè xa bị bỏ trống nhiều vùng đỏ.
- Quy tắc và hành động sửa:
  1. Loại bỏ các false `person` là cột đèn/trụ tín hiệu (`59, 42, 51` ở `350023`), bỏ `48:bicycle` giả trên `460147`, và loại các detection xe trùng lặp (`44, 48, 55, 56, 58` ở `350023`; `71, 82, 83` ở `460147`).
  2. Dùng `sam3_interact` khoanh riêng hai xe đạp (`b1, b2`) và cắt biên sạch theo chiều dọc dưới thân hai người đạp xe (`22` giữ phần người `y < 358` → `14 px`; `b1` giữ khung-bánh dưới `22 px`; `25` giữ `24 px`; `b2` giữ `30 px`), kiểm lại trực tiếp trên ảnh crop xuất ra từ `submissions/hard_panoptic.zip`.
  3. Bổ sung `stuff` đúng ranh và bảo toàn nguyên tắc `thing` không chồng `stuff`: thêm `vegetation` cho nửa dưới bồn cây dải phân cách/bụi cỏ công viên bên phải và tán cành trơ lá, `building` cho khối nhà vàng ENEOS (sau lớp cây tiền cảnh), `sky` cho khe trời phía trên cầu vượt (`y < 118`), tách phần đường nhánh bên phải có vạch qua đường (`x >= 520`) thành `road` và phần thềm gạch lát chỉ hướng vàng bên cạnh (`x < 508`) thành `sidewalk`; giữ nguyên cột đèn, biển hiệu đứng riêng, hàng rào thép, trụ cầu đá và cần cẩu làm `void`.
- Sau sửa đã Save và export lại chưa? Đã cập nhật CVAT Task 5, Save và export lại `submissions/hard_panoptic.zip` (`save_images=False`), kiểm tra lại cả ảnh tổng quan lẫn zoom crop từ ZIP xuất cuối: `000000350023.jpg` giảm `void_pct` xuống `3.39%` (`road/sidewalk/building/vegetation/sky` = `5`, `car: 23`, `bus: 2`, `person: 11`, `bicycle: 2`, `motorcycle: 1`, `traffic light: 7`), `000000460147.jpg` giảm `void_pct` xuống `6.48%` (`5 stuff`, `car: 49`, `truck: 4`, `bus: 1`, `motorcycle: 3`, `person: 2`).
- Tự đánh giá tự động: Chưa chấm bằng ground truth vì repo chưa có bộ reference (`0` file đáp án trong fork); mới chạy kiểm hợp đồng cấu trúc bằng `python3 scripts/inspect_submissions.py --dir submissions` (`[OK]` cả 9/9 ZIP).

## 4. Ba ca chưa chắc hoặc đã cân nhắc

| Ảnh/vị trí | Hai cách hiểu có thể | Quy tắc/chứng cứ | Quyết định hoặc câu hỏi cho coach |
| --- | --- | --- | --- |
| 1. `hard_panoptic` / `000000460147.jpg` — khối ENEOS vàng, hộp biển nóc `ENEOS` và biển `Housing Plaza` bên trái (`x∈[0..220], y∈[40..240]`) | (A) Gộp toàn bộ mặt dựng + các biển hiệu gắn phía trên vào `building`; hoặc (B) chỉ tính phần khối nhà/trụ kiến trúc vào `building`, còn bảng quảng cáo dựng riêng và hộp biển mái coi là `sign` (`void` ngoài 12 lớp) | Bảng 12 lớp `hard_panoptic` không có `traffic sign / pole / fence / wall`; phía trước khối nhà còn có tán cành cây rụng lá (`vegetation`) che chéo | Giữ khối kiến trúc vàng trạm ENEOS và tường nhà phía sau tán cành trơ lá là `building`, để biển `Housing Plaza`, `KEIYUCA`, hộp `ENEOS` rời, cột đèn và hàng rào xanh là `void`. Hỏi coach: biển hiệu thương mại gắn liền trên nóc trụ ki-ốt có nên gộp `building` hay giữ `void`? |
| 2. `cp1_holes` / `000000144300.jpg` — ống quần người ngồi sau (`[318, 173..205]`) và tán rạp che bớt đầu xe tải `Kawasaki Racing` (`[314, 55, 495, 173]`) | (A) Bỏ qua phần ống quần rất nhỏ bị che sau người lái và để tán rạp che một phần thân xe tải; hoặc (B) tách `person` thứ 2 thành mảnh riêng không chạm người lái và ưu tiên lớp xe tải xanh trước phần bạt rạp | Quy tắc tách instance khi bị che khuất: mọi bộ phận nhìn thấy rõ thuộc người ngồi sau (chân/quần denim ló dưới yên) cần tách khỏi `person` lái xe và `motorcycle`; trong lần thử đầu tiên vòng lặp ghi đè từng pixel làm bạt rạp che nuốt mất `truck` | Đưa `person` người ngồi sau và phần xe tải `Kawasaki Racing` đúng thứ tự không bị tán bạt đè mất, kiểm lại overlay `cp1` đủ 5 instance (`1 car, 2 person, 1 motorcycle, 1 truck`). Nhờ coach xem phần mũi giày/ống quần lộ ra rất hẹp có đạt ngưỡng tính instance không? |
| 3. `cp6_coverage` / `7daa6479-67988f3f.png` & `cp4_curb` / `7d83710e-4697c3b2.png` — người phản chiếu sau kính cửa hàng, đố kính dọc và vết bùn trên sân bê tông | (A) Gắn `person` cho bóng người/người mẫu sau kính tầng trệt và giữ đố kính/vết bùn rời rạc thành lỗ `void`; hoặc (B) kính cửa hàng là bề mặt thuộc `building` và vết bẩn nhỏ trên sân là `sidewalk` liên tục | Trên ảnh gốc `cp6`, hình người ở khu vực Cột 4 nằm sau mặt kính showroom chứ không phải người đi trên vỉa hè, còn chiếc taxi vàng dưới mái vòm bên phải từng bị gán nhầm `vegetation` đã sửa về `car`; trên `cp4`, các đốm đen trên vỉa hè là vết đất/vữa chứ không phải vật thể tách biệt | Sửa vùng tủ kính Cột 4 về `building`, sửa taxi mái vòm phải về `car`, giữ bộ hành thật trên vỉa hè là `person`; trên `cp4` lấp các đốm bùn trên `sidewalk` (`19.53%`) nhưng giữ rãnh tiếp giáp lề-đường sạch giữa `road` (`13.11%`) và `sidewalk`. Hỏi coach: các thanh nẹp nhôm rất mảnh chia ô cửa kính (`cp6`) nên để `void` hay phủ liền vào `building`? |
