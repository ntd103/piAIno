# Piaino Documentation — Nghiên cứu Thị trường, Hệ thống Câu hỏi Ad-hoc & Đặc tả Kỹ thuật AI Piano Guru

> **Tổng quan dự án:** **Piaino** là nền tảng Web App hỗ trợ tự học Piano và tự động hóa giảng dạy Piano Online/Hybrid với **AI Guru** hoạt động trên tín hiệu MIDI kết hợp bản nhạc cấu trúc (MusicXML). Không giống các ứng dụng phổ thông chỉ chấm đúng/sai cao độ theo lưới metronome cứng nhắc, Piaino đánh giá sâu tới **cường độ tách bè (polyphonic voicing), sự co giãn nhịp biểu cảm (rubato vs. lỗi nhịp), kỹ thuật ngắt/nối ngón (articulation & key overlap), độ sạch của sustain pedal (CC64 harmonic blur)** và tự động biên dịch thành bài tập sửa lỗi 60 giây.

---

## Cấu trúc Bộ Tài liệu Nghiên cứu & Thiết kế (`docs/`)

| Thứ tự | Tài liệu | Nội dung cốt lõi |
|---:|---|---|
| **01** | [01-market-research-learners-and-competitors.md](01-market-research-learners-and-competitors.md) | **Khảo sát Người học & Đối thủ (2026):** So sánh 10 nền tảng (Simply Piano, Flowkey, Piano Marvel, Skoove, Yousician, Playsheet, SightPiano, ROLI Airwave, Tonebase). Chỉ rõ những gì đối thủ đã bão hòa và 5 khoảng trống chuyên môn sâu (**The AI Guru Gap**: Rubato vs. Metronome, Voicing đa bè, Articulation/Note-Off, Pedal CC64, Root-Cause Coaching). |
| **02** | [02-market-research-teachers-and-automation.md](02-market-research-teachers-and-automation.md) | **Khảo sát Giáo viên Piano & Tự động hóa Online/Hybrid:** Phân tích 5 giới hạn vật lý/vận hành của dạy piano online (nén âm thanh WebRTC/Zoom làm mất dải động $pp\text{--}ff$ và bồi âm pedal, "Điểm mù 6 ngày tự tập", quá tải chấm bài tập video giữa tuần). Đối chiếu với Synkii, PracticeMeter, Syncopated, Piano.studio, Trinity NoteLab và đề xuất **6 quy trình tự động hóa ROI cao nhất**. |
| **03** | [03-datasets-and-research-landscape-2026.md](03-datasets-and-research-landscape-2026.md) | **Hệ sinh thái Datasets, Papers & Notebooks Mới nhất (tới 2026):** Phân tích giới hạn của `ASAP v1.2` gốc và mở rộng sang **(n)ASAP** (7M nốt căn chỉnh 1-1), **PercePiano** (19 tiêu chí cảm thụ chuyên gia, 2024–2026), **MAJEPPA** (ISMIR 2026, 6 cấp độ kỹ năng $\times$ 6 bối cảnh), **PERiScoPe v1.1** (01/2026, 35.815 cặp, 88.4M nốt), **Aria-MIDI** (ICLR 2025, 1.18M files), **ATEPP**, **PIG** và hệ thư viện `partitura`, `parangonar`, `pymatchmaker`, `RCG-DMAF-Net`. |
| **04** | [04-adhoc-questions-and-step-by-step-reasoning.md](04-adhoc-questions-and-step-by-step-reasoning.md) | **Chuỗi Lập luận First-Principles & Hỏi–Đáp Ad-hoc:** Giải quyết các bài toán hóc búa nhất: *Làm sao bắt bệnh kỹ thuật từ MIDI mà không bịa chuyện? Làm sao phân biệt Rubato nghệ thuật với đánh sai nhịp? Làm sao hiệu chuẩn Velocity giữa các dòng đàn điện khác nhau? Làm sao thiết kế UX/UI 2 pha và ngân sách độ trễ (`< 10 ms` khi đang đàn, `< 100 ms` khi hiện bảng chấm) để không gây loạn mắt?* |
| **05** | [05-modeling-deep-dive-line-by-line.md](05-modeling-deep-dive-line-by-line.md) | **Đặc tả Modeling Chi tiết Từng Đơn vị Tính toán (Debug Line-by-Line):** Đi từ gói tin MIDI 3-byte $\rightarrow$ `KeyPedalStateMachine` (tách `t_finger_off` và `t_sound_off`) $\rightarrow$ Công thức Log-Likelihood của `Jump-Aware Score Aligner` $\rightarrow$ Bảng định nghĩa toán học & trace số thực cho **44 chiều đặc trưng Nốt** + **51 chiều đặc trưng Đoạn** $\rightarrow$ Mã nguồn PyTorch từng dòng kèm kích thước Tensor của mạng **`Piaino-HCAF-Net`** $\rightarrow$ Bộ biên dịch JSON chẩn đoán sang lời khuyên Guru & Micro-Drill. |
| **06** | [06-system-architecture-ux-and-testing.md](06-system-architecture-ux-and-testing.md) | **Kiến trúc Hệ thống, Wireframe UX/UI, Kiểm thử & Lộ trình:** Kiến trúc Split-Compute (Browser WebMIDI + WASM + FastAPI/ONNX + WebRTC MIDI Relay cho lớp học online), thiết kế màn hình Live Play / Guru Review / Teacher Briefing, ma trận 6 bộ Test tự động và lộ trình 4 giai đoạn. |
| **Ref** | [../data/asap-dataset/DEEP-UNDERSTAND.md](../data/asap-dataset/DEEP-UNDERSTAND.md) | **Tài liệu Phân tích Kỹ thuật `ASAP v1.2` Local Checkout:** Kiểm kê chi tiết 1.067 bản biểu diễn, 235 score variants, 222 tác phẩm logic và các bẫy xử lý dữ liệu thực tế. |

---

## Tóm tắt 5 Quyết định Thiết kế Cốt lõi (Key Architectural Decisions)

1. **Không dùng 1 LLM Blackbox duy nhất cho dữ liệu MIDI thô — Dùng Kiến trúc Lai 3 Tầng (Tri-Tier Hybrid):**
   - **Tầng 1 (Client WASM, `< 2 ms`):** Căn chỉnh nốt thời gian thực có hỗ trợ quay lui/nhảy nhịp (`Jump-Aware HMM / OLTW`) + Máy trạng thái vật lý tách biệt thời điểm *ngón tay rời phím (`t_finger_off`)* và thời điểm *âm thanh tắt sau Pedal (`t_sound_off`)*.
   - **Tầng 2 (Fast Neural Assessment `Piaino-HCAF-Net`, `< 25 ms`):** Kết hợp chuỗi đặc trưng 44 chiều cấp độ nốt qua cấu trúc phân cấp **Note $\rightarrow$ Beat $\rightarrow$ Measure $\rightarrow$ Phrase** với nhánh **51 đặc trưng kỹ thuật toàn cục** bằng cơ chế *Residual Cross-Gating*, xuất đồng thời: **(a) Điểm 6 khía cạnh tại từng ô nhịp (vẽ Heatmap)**, **(b) 19 chiều cảm thụ chuyên gia (`PercePiano`)**, **(c) Cấp độ kỹ năng (`MAJEPPA`)**, và **(d) Vector hiệu chỉnh từng nốt để phát lại bản mẫu Guru**.
   - **Tầng 3 (Grounded Pedagogical Coach, `< 500 ms` streaming):** Biên dịch các chỉ số bất thường đã được định vị chính xác thành lời khuyên sư phạm theo **Quy tắc 3-1-1** (3 trụ cột tổng quan – 1 điểm nghẽn ưu tiên nhất – 1 bài tập Micro-Drill 60 giây) với **0% Hallucination**.
2. **Tách biệt UX/UI thành 2 Pha theo Tải nhận thức (Dual-Phase Cognitive UX):**
   - **Đang đàn (`Live Play Mode`):** Tối giản tuyệt đối, không hiện chữ hay biểu đồ phức tạp gây vấp ngón; chỉ trượt con trỏ, đổi màu nhẹ đầu nốt và hiển thị thanh cân bằng 2 tay / trạng thái pedal ở tầm nhìn ngoại vi.
   - **Vừa dừng tay (`Guru Review Mode`):** Đẩy kết quả Heatmap khuông nhạc + Điểm số trong `< 100 ms` (Push #1), tiếp nối bằng lời khuyên chi tiết và nút nghe so sánh *Bản bạn vừa đàn vs. Bản Guru sửa mẫu* (Push #2).
3. **Giải quyết triệt để giới hạn Dạy Piano Online cho Giáo viên:**
   - Truyền trực tiếp luồng sự kiện MIDI (`< 2 KB/s`) kết hợp đồng bộ bản nhạc MusicXML để tái tạo âm thanh gốc 100% ở phía giáo viên (loại bỏ hoàn toàn hiện tượng nén mất dynamics/pedal của Zoom/Meet).
   - Tự động hóa **Chấm sơ bộ bài tập về nhà (AI Pre-Screening)**, **Cảnh báo sớm thói quen sai trong 6 ngày tự tập**, và **Bản tóm tắt chẩn đoán 2 phút trước giờ lên lớp**.
