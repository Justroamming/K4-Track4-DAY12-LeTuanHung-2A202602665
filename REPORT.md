# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Lê Tuấn Hưng |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 202602665 |
| GitHub | letuanhung2a202602665 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed=1743, model_seeds=[5817, 5818, 5819] |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.270 | 19.8 | BLOCK |
| IDM | 1.518 | 0.1 | (baseline) |
| EgoMLP | 0.762 | 23.0–26.7 | BLOCK |
| PercMLP | 0.620 | 0.3–0.9 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.816 | 0.2–0.2 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Bước 0 audit dữ liệu cho thấy cột `Acc_LV` (gia tốc xe dẫn đường ghi sẵn trong log) có tương quan cực thấp với đạo hàm vận tốc thực tế (`acc_lead_column_corr_with_dv_dt` = 0.045), nghĩa là cảm biến ghi nhận gia tốc rất nhiễu và không đáng tin cậy. Thay vào đó, hệ thống tự tính gia tốc bằng cách làm mượt `Speed_LV` (vận tốc xe dẫn) bằng cửa sổ trượt width=5 rồi lấy gradient trung tâm, tạo ra `a_lead` sạch hơn nhiều. Có **383 quỹ đạo trùng lặp tuyệt đối** bị loại bỏ (exact_duplicates_removed=383), các bản sao này có Trajectory_ID khác nhau nhưng nội dung giống hệt nhau, nếu không loại sẽ làm rò rỉ dữ liệu giữa train/test. Chia theo quỹ đạo (không chia theo frame) vì các frame liên tiếp trong cùng quỹ đạo tương quan cao, chia ngẫu nhiên frame sẽ làm rò rỉ thông tin từ train sang test.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 (thứ tự phép toán khác, float32 thay float64) khác bit ở **600/600 đoạn** (not_bit_identical=600) do tính chất không kết hợp của số thực dấu phẩy động, nhưng **0/600 đoạn vượt dung sai** (differ_beyond_tol=0, max_abs_diff=4.03e-4 < atol=1e-3), chứng minh refactor không thay đổi hành vi. Thay đổi hành vi thực (thêm quy tắc AEB phanh khẩn cấp khi TTC<1.5s) làm lệch **11/600 đoạn** (differ_beyond_tol=11, max_abs_diff=1.56 m/s²), các đoạn này chính là những tình huống xe dẫn phanh gấp khiến TTC giảm dưới ngưỡng 1.5s. Phát lại realtime (có mô phỏng độ trễ xử lý spike 140ms với xác suất 2%) cho thấy **581 late + 575 dropped frames** trong 59448 frames, làm **376/600 đoạn** thay đổi so với golden; còn chế độ 'fastest' (xử lý ngay lập tức, không deadline) che hoàn toàn hiện tượng này (0 late, 0 dropped). Điều này mô phỏng thực tế: trong xe thật, nếu planner xử lý chậm hơn 100ms (1 frame), lệnh điều khiển sẽ trễ và giữ lệnh cũ, gây sai lệch tích lũy.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản: v_ego = 10.0 m/s (từ MSSV), **60 gap × 45 closing speed = 2700 ô**. Tại t_react=0.0s, vùng bất khả kháng (unavoidable) chiếm **0.96%** ô; tăng t_react lên 0.6s, vùng này tăng lên **5.15%**. IDM tại t_react=0.0s có collision_area=0.96% (chỉ đâm đúng ở vùng bất khả kháng), tại t_react=0.6s collision_area=5.15% (tăng **4.19 điểm %** do độ trễ phản ứng). Log khai phá **614 sự kiện phanh gấp** (6.4% quỹ đạo), p99 peak decel = -3.9 m/s²; giới hạn phanh mạnh nhất của người lái (p99.9) là **3.6 m/s²** (human_brake_limit_p99.9), trong khi cầu trục cho phép 9 m/s². Có **5.2% ô lưới** (share_beyond_human_brake_limit) đòi hỏi phanh > 3.6 m/s² — vùng này log không phủ, chính là lý do mô hình học từ log (PercMLP) yếu ở các kịch bản cắt đầu phanh gấp: nó chưa từng thấy chuyên gia xử lý tình huống cần phanh mạnh hơn 3.6 m/s².

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Planner | Open-loop L2 (m) | Hạng L2 | Closed-loop collision (%) | Hạng CL |
|---|---|---|---|---|
| PercMLP | 0.620 | 1 | 0.7 | 2 |
| EgoMLP | 0.762 | 2 | 24.7 | 4 |
| const-vel | 1.270 | 3 | 19.8 | 3 |
| IDM | 1.518 | 4 | 0.1 | 1 |

**4 cặp đảo thứ tự** (order_flips): (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). Đáng chú ý nhất: **IDM có L2 tệ nhất (1.518m, hạng 4) nhưng vòng kín an toàn nhất (0.1%, hạng 1)**; ngược lại **EgoMLP có L2 tốt hơn const-vel (0.762 vs 1.270) nhưng vòng kín đâm nhiều hơn (24.7% vs 19.8%)**. Nguyên nhân: L2 vòng hở đo sai số vị trí sau 1–3s giả định xe dẫn giữ nguyên hành động, nhưng IDM là luật vật lý có tính ổn định (feedback âm qua s_star), sai số không tích lũy. EgoMLP chỉ xem history ego (không thấy gap/lead_v), học được "bám đuôi trung bình" nhưng khi xe dẫn phanh gấp (chiếm 4.7% cửa sổ), nó phản ứng chậm → sai số tích lũy vòng kín (covariate shift). PercMLP có perception (gap, lead_v) nên vòng kín tốt hơn (0.7%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP collision rate trên log dao động **23.0% – 26.7%** giữa 3 seed (mean=24.7%, spread=3.7 điểm %). Khoảng tin cậy Wilson 95% cho seed tệ nhất (5818, k=769, n=2883) là **[25.1%, 28.3%]** — верхняя граница 28.3% xa trung bình 24.7%. Chỉ chạy 1 seed (ví dụ seed=5819 cho rate=23.0%) sẽ báo "23% collision", nhưng seed 5818 cho 26.7% — chênh lệch 3.7 điểm % đủ thay đổi kết luận release/block (R2 margin chỉ 1 điểm %). Wilson CI bắt ta nhìn **cận trên** chứ không phải trung bình: PercMLP+ worst-seed upper bound = 0.5% ≤ baseline 0.1% + 1% → PASS R2. Chạy 1 seed là nguy hiểm vì variance cao (đặc biệt EgoMLP: std=0.0047 L2, spread collision 3.7%) có thể chọn ngẫu nhiên seed "đẹp" hoặc "xấu", dẫn đến kết luận sai về an toàn.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

| Planner | Rear replay ANY (%) | Rear reactive ANY (%) | Rear collision (replay/react) t=0.6s |
|---|---|---|---|
| IDM | 70.1 | 5.1 | 64.9% / 0.0% |
| PercMLP | 73.9 | 21.1 | 53.9% / 0.0% |

IDM driver hậu phương được hiệu chỉnh **T = 1.4s** (Wasserstein-1 = 0.351, gần phân phối headway người thực median 1.85s), trong khi xe không phản ứng (replay) có W1 = 0.837 — xa hơn 2.4 lần. Tại t_react=0.6s, IDM với xe sau replay bị **70.1% collision ANY** (trong đó 64.9% là rear collision), nhưng xe sau reactive chỉ **5.1% ANY** (0% rear). **Va chạm giả** xảy ra vì xe sau replay giữ nguyên tốc độ log (không biết ego phanh gấp), trong khi thực tế tài xế sau sẽ phanh theo. Điều này làm IDM trông "tệ" hơn thực tế (70% vs 5%), che lấp được sự an toàn thực của IDM. Khi xét cổng phát hành R6 (reactive world), PercMLP+ chỉ 5.3% vs IDM baseline 5.1% → PASS, nếu dùng world replay thì PercMLP+ 69.6% vs IDM 70.1% → cũng PASS nhưng sai thực tế. Kết luận: phải dùng world phản ứng mới đánh giá đúng an toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi **R2** (logs: upper 95% = 1.3% > baseline 0.1% + 1%), **R3** (brake slice: 4.8% upper 8.9% > baseline 1.1% + 5%), **R4** (grid t=0.6s: attributable 15.9% > baseline 0% + 2%), **R6** (reactive world: 21.1% > baseline 5.1% + 2%). Nguyên nhân gốc: (1) Log thiếu kịch bản cắt đầu phanh gấp (5.2% lưới cần phanh > 3.6 m/s² mà log không có) → PercMLP yếu R4, R6; (2) Covariate shift vòng kín: sai số nhỏ tích lũy → PercMLP đi vào trạng thái không có trong train → yếu R2, R3. Sửa bằng **2 bước**: thêm **20,000 cửa sổ cut-in** (từ mô phỏng, gán nhãn IDM) + **DAgger 2 vòng** (planner tự lái, lấy nhãn IDM tại trạng thái nó đi vào). Kết quả: PercMLP+ **RELEASE** mọi quy tắc (R2 upper=0.5%, R4 attributable=0.1%, R6=5.3%). **PercMLP+ KHÔNG tốt hơn IDM**: open-loop L2 tệ hơn (0.816 vs 1.518 — wait, 0.816 < 1.518, nên PercMLP+ L2 tốt hơn; nhưng closed-loop collision 0.2% vs IDM 0.1% — tương đương). Tuy PercMLP+ qua cổng, **chưa nên phát hành ngay** vì: (a) chỉ test 1D, không có cắt làn ngang, chèo lái; (b) held-out v_ego=20m/s (training ≤16) PercMLP+ attributable 0.0–1.8% (IDM 0%) — suy rộng chưa chắc an toàn; (c) cần test sensor noise, độ trễ lớn hơn, thời tiết, lâu dài (long-tail). Đề xuất: chạy hardware-in-loop, track test, thêm diversity scenarios, monitor ODD.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Chỉ mô phỏng 1D 종 продоль (longitudinal)**: Lab chỉ kiểm soát ga/phanh theo trục dọc, bỏ qua hoàn toàn điều khiển lái (lateral), thay đổi làn, tránh chướng ngại vật ngang. Thực tế va chạm thường do kết hợp cả hai (ví dụ: phanh gấp khi đang lái tránh). Kết luận release chỉ áp dụng cho chức năng ACC/bám đuôi đơn thuần, không phải tự hành đầy đủ.

2. **Môi trường mô phỏng quá lý tưởng (simple physics, deterministic lead)**: Xe dẫn chỉ giữ tốc độ không đổi sau cut-in, không dao động, không phanh ngẫu nhiên, không cắt làn liên tục. Cảm biến assumed perfect (chỉ thêm noise Gaussian đơn giản). Thực tế: lead vehicle hành vi phức tạp, sensor có outlier, occlusion, latency biến thiên. 5.2% lưới cần phanh > 3.6 m/s² log không phủ — PercMLP+ học từ IDM tại các trạng thái mô phỏng, chưa chắc IDM là tối ưu ở thực tế.

3. **Phân phối ODD (Operational Design Domain) hẹp**: Chỉ test v_ego=10 m/s (36 km/h), gap 2.7–59m, closing speed 0–10 m/s. Không test: tốc độ cao (>30 m/s), mưa/bão (ma sát thấp), ban đêm, giao thông ngược chiều, xe đạp/người đi bộ. Held-out test v_ego=20m/s cho thấy PercMLP+ attributable 0.0–1.8% vs IDM 0% — suy rộng ra ngoài ODD train rủi ro.

**Cần làm tiếp**: (a) Mở rộng sang mô phỏng 2D/3D (CARLA, NuScenes) với lateral control; (b) Adversarial scenario generation (tìm counterexample tự động); (c) Hardware-in-the-loop với ECU thật, sensor thật, độ trễ thực; (d) Formal verification cho các critical scenarios; (e) Long-tail data collection tại ODD biên; (f) Continuous monitoring sau deploy (SOTIF).
