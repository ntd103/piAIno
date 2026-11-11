# 04. Lập luận Từng bước & Hệ thống Câu hỏi Ad-hoc (Usecase, Latency, UX/UI, Dataset, Modeling, System, Testing)

> **Mục tiêu tài liệu:** Xây dựng chuỗi lập luận nguyên lý đầu tiên (First-Principles Reasoning) thông qua việc tự đặt và giải quyết triệt để các câu hỏi hóc búa nhất (Ad-hoc Questions). Trọng tâm xuyên suốt: **Làm sao để Model phục vụ ĐÚNG Usecase thực tế của người học/giáo viên, đồng thời đảm bảo TỐC ĐỘ PHẢN HỒI tức thì và UX/UI không gây quá tải nhận thức.**

---

## Phần 1: Câu hỏi Trọng tâm về Usecase & Sự khớp nối giữa Model và Nhu cầu User (Model–Usecase Fit)

### Q1.1: Tín hiệu MIDI chỉ có `Pitch`, `Onset`, `Offset`, `Velocity` và `CC64` (không có Camera quay tay, không có Microphone thu âm vật lý). Làm sao AI đóng vai trò "Guru" bắt bệnh kỹ thuật và âm sắc mà không bị "bịa chuyện" (Hallucination)?

**Lập luận & Giải pháp:**
Một giáo viên giàu kinh nghiệm khi nhắm mắt nghe học trò đàn (hoặc nhìn dữ liệu cơ học phím đàn) vẫn chẩn đoán chính xác 90% lỗi kỹ thuật vì **mọi chuyển động cơ học sai của bàn tay đều để lại "dấu vân tay vật lý" (kinematic signature) bất biến trên chuỗi sự kiện MIDI**:

| Vấn đề kỹ thuật của học viên | Dấu vân tay vật lý trên dữ liệu MIDI (Đo được 100%) | Giới hạn cần trung thực (Không suy diễn quá đà) |
|---|---|---|
| **Ngón 4–5 yếu hoặc bị dính khi chạy gam/trill** | Khoảng cách giữa các lần chạm phím $\text{IOI}_{k} = t_{\text{on}, k} - t_{\text{on}, k-1}$ bị co giãn bất thường (chập-nhanh-chập) kèm theo **Key Overlap Time** $\text{KOT}_k = t_{\text{off}, k-1} - t_{\text{on}, k} > 35\text{ ms}$ (ngón trước chưa nhấc lên ngón sau đã đè xuống) và `velocity` sụt giảm $> 20\%$. | Hệ thống kết luận: *"Hai nốt này bị dính phím (+42ms overlap) và lực ngón yếu đi"* — **không** phán bừa *"Khuỷu tay bạn bị nhô cao"* nếu không có camera. |
| **Cổ tay cứng / Đập phím khi đánh hợp âm lặp (Chordal Stiffening)** | Khi đánh chuỗi hợp âm 8th-notes nhanh, `velocity` tăng vọt kèm theo `duration ratio` co ngắn đột ngột hoặc chênh lệch thời điểm chạm phím giữa các nốt trong cùng 1 hợp âm ($\text{Chord Spread} = \max(t_{\text{on}}) - \min(t_{\text{on}})$) dao động hỗn loạn. | Kết luận: *"Hợp âm bị xuống phím không đều (lệch 38ms) và lực chạm quá gắt"* $\rightarrow$ Gợi ý thả lỏng cổ tay và tập rơi trọng lượng. |
| **Chuyển thế tay (Position Shift / Thumb-under) bị trễ** | Trước một khoảng nhảy quãng lớn hoặc vắt ngón cái (thumb-under), nốt ngay trước đó bị **cắt cụt trường độ** ($\text{dur}_{\text{perf}} / \text{dur}_{\text{score}} < 0.45$) nhưng nốt đích vẫn bị **vào trễ** ($\Delta t_{\text{onset}} > +65\text{ ms}$). | Chỉ ra chính xác: *"Bạn buông nốt cuối nhịp 7 quá sớm nhưng vẫn nhảy sang nhịp 8 bị trễ 70ms — cần chuẩn bị mắt và vị trí tay sớm hơn."* |
| **Dùng Pedal che lấp lỗi thiếu Legato ngón tay ("Pedal Cheating")** | Trên bản nhạc có dấu *slur* (yêu cầu legato ngón tay), nhưng $\text{KOT} < -80\text{ ms}$ (tay nhấc hoàn toàn khỏi phím tạo khoảng hở lớn) trong khi chân dậm `CC64 = 127` liên tục. | Bắt bài ngay lập tức: *"Bạn đang nối âm bằng Pedal chứ không phải bằng ngón tay (tay nhấc sớm 95ms trước nốt tiếp theo). Hãy tập đoạn này không dùng Pedal trước."* |

$\rightarrow$ **Nguyên tắc thiết kế số 1:** Mọi lời nhận xét của AI Guru đều phải **neo chặt vào chỉ số vật lý đo được từ MIDI (Evidence-Anchored Coaching)**. Tuyệt đối không dùng LLM sinh lời khuyên chung chung không có căn cứ dữ liệu.

---

### Q1.2: Làm sao Model phân biệt được người chơi nâng cao đang dùng *Rubato nghệ thuật* với người mới học đang *Đánh sai nhịp / Vấp nhịp*? Nếu phạt nhầm Rubato thì sao?

**Lập luận & Giải pháp:**
Đây là điểm yếu chí mạng của Simply Piano, Yousician và Piano Marvel. Để giải quyết triệt để, Piaino **tách bạch 2 khái niệm toán học** và thiết kế **Chế độ Đánh giá Thích ứng theo Cấp độ (Stage-Aware Assessment Mode)**:

1. **Sự khác biệt toán học giữa "Lỗi Nhịp (Rhythmic Error)" và "Rubato Nghệ thuật (Expressive Rubato)":**
   - **Lỗi nhịp vô thức (Beginner Mistake):** Biến thiên vận tốc cục bộ (Local Tempo $QPM_i$) có **đạo hàm bậc 2 (Jerk / Độ giật) rất lớn** tại các vị trí khó về kỹ thuật (nhảy quãng, hợp âm dày), hoặc tốc độ trôi một chiều (chỉ nhanh dần lên ở đoạn dễ - *rushing*, rồi khựng đột ngột ở đoạn khó - *stuttering*).
   - **Rubato nghệ thuật (Expressive Timing):** Đường cong tempo $QPM(b)$ biến thiên **mượt (smooth, low-frequency curve)** và **tương quan chặt chẽ với cấu trúc hình thức của câu nhạc (Phrase Arch)**: tăng nhẹ ở đầu câu, giãn nhịp (*ritardando* / *agogic accent*) ở đỉnh câu hoặc tại *cadence* (kết câu cuối mỗi 4 hoặc 8 ô nhịp).
2. **Cơ chế xử lý trong Model:**
   - Thay vì trừ thời gian tuyệt đối $|t_{\text{perf}} - t_{\text{score}}|$, hệ thống tính **Normalized Inter-Onset Interval Ratio (Tỷ lệ trường độ tương đối cục bộ)** so với nhịp trung bình động của 4 phách liền kề, đồng thời chiếu lên **Phân phối Chuẩn Tham chiếu Đa Nghệ sĩ (Multi-Artist Expressive Envelope)** trích từ `PERiScoPe` / `(n)ASAP`.
   - **Phân tách theo Chế độ Luyện tập của User (UX Toggle):**
     - **Chế độ 1 — `Strict Pulse / Metronome Drill` (Dành cho giai đoạn vỡ bài & luyện nhịp):** Khoan dung hẹp ($\pm 35\text{ ms}$), yêu cầu giữ nhịp phẳng tuyệt đối.
     - **Chế độ 2 — `Expressive Guru Mode` (Dành cho giai đoạn hoàn thiện tác phẩm):** Cho phép đường cong tempo co giãn tự do trong biên độ cấu trúc câu nhạc, chỉ phạt khi xảy ra **độ giật cục bộ (local rhythmic jitter)** hoặc **lệch tỷ lệ trường độ nội bộ** (ví dụ nốt móc đơn bị đánh dài bằng nốt đen).

---

### Q1.3: Cùng một tác phẩm, các danh cầm chơi khác nhau. Nếu Model chỉ học một đáp án duy nhất thì có làm nghèo nàn cá tính âm nhạc của người học không?

**Lập luận & Giải pháp:**
- Trong `ASAP v1.2` và `PERiScoPe v1.1`, mỗi tác phẩm cổ điển có từ **5 đến 29 bản biểu diễn của các nghệ sĩ khác nhau**.
- Do đó, với mỗi ô nhịp $m$ và mỗi nốt $i$, ta không lưu một giá trị chuẩn duy nhất $\mu_i$, mà tính **Không gian Biểu diễn Hợp lệ (Valid Expressive Manifold)**:
  - Những yếu tố **Bất biến (Invariant Constraints - Bắt buộc mọi người chơi phải tuân thủ):** Cao độ nốt, thứ tự phách, bè giai điệu phải nổi hơn bè đệm ($\text{Velocity}_{\text{melody}} > \text{Velocity}_{\text{accomp}}$), pedal không được giữ xuyên qua hai hợp âm nghịch nhau gây đục âm.
  - Những yếu tố **Khả biến (Stylistic Degrees of Freedom - Cho phép tự do sáng tạo):** Tempo tổng thể (ví dụ 112–136 BPM), mức độ sâu của *ritardando* cuối câu, độ sáng/tối của âm sắc tổng thể.
- Model chỉ đánh dấu là **Lỗi (Defect)** khi thông số của người chơi rơi ra ngoài phân phối hợp lệ và vi phạm các ràng buộc bất biến về nhạc lý; còn trong vùng hợp lệ, AI Guru sẽ nhận xét theo hướng **Phân tích phong cách (Stylistic Mirroring)**: *"Cách xử lý câu nhạc của bạn thiên về lối chơi giàu kịch tính, tempo nhanh và sắc nét (tương đồng 82% với phong cách Argerich), nhưng lưu ý ở nhịp 15 pedal hơi dày."*

---

### Q1.4: Làm sao đảm bảo Model giải quyết đúng nhu cầu của User mà không biến màn hình thành một bảng số liệu kỹ thuật khô khan?

**Lập luận & Giải pháp (Quy tắc "3-1-1 Pedagogical Funnel"):**
Một người thầy giỏi không bao giờ đổ 50 lỗi lên đầu học trò cùng một lúc. Dù bên dưới Model tính toán hàng trăm đặc trưng trên từng nốt và 19 chiều cảm thụ của `PercePiano`, tầng đầu ra (Output Funnel) luôn tuân thủ **Quy tắc 3-1-1**:
- **Tối đa 3 Điểm sáng / Chỉ số tổng quan (3 Macro Radar Pillars):** *Accuracy & Rhythm* (Độ chuẩn xác & Nhịp), *Touch & Voicing* (Kiểm soát phím & Tách bè), *Pedal & Phrasing* (Pedal & Câu nhạc).
- **Đúng 1 Vấn đề ưu tiên cao nhất cần sửa ngay (1 Top-Priority Bottleneck):** Chọn đoạn 2–4 ô nhịp có mức độ ảnh hưởng lớn nhất (ví dụ: *"Ô nhịp 12–14: Bè tay trái đang đè lấp giai điệu tay phải"*), tô sáng trực tiếp trên khuông nhạc.
- **Đúng 1 Bài tập tương tác nhỏ thực hiện được ngay trong 60 giây (1 Interactive Micro-Drill):** Nút bấm **"Tập sửa ngay đoạn nhịp 12–14 (3 lượt)"** ngay dưới lời nhận xét.

---

## Phần 2: Câu hỏi Trọng tâm về Tốc độ Phản hồi (Latency) & Thiết kế UX/UI

### Q2.1: Khi người học đang tập trung đàn, mắt nhìn bản nhạc, tai nghe âm thanh, tay di chuyển liên tục — nếu AI hiện thông báo chữ hoặc biểu đồ phức tạp lên màn hình thì user sẽ bị phân tâm và vấp ngón ngay lập tức. Giải quyết UX/UI như thế nào?

**Lập luận & Giải pháp (Kiến trúc UX 2 Pha tách biệt theo Tải nhận thức — Dual-Phase Cognitive UX):**

Con người không thể vừa thực thi vận động tinh ở tốc độ mili-giây (Fine Motor Execution) vừa đọc hiểu ngôn ngữ phân tích (Analytical Reading). Do đó, UX/UI của Piaino bắt buộc chia thành **2 Pha hoàn toàn tách biệt**:

| Pha trải nghiệm | Trạng thái của User | Những gì ĐƯỢC PHÉP hiển thị trên màn hình | Những gì TUYỆT ĐỐI CẤM hiển thị | Yêu cầu Độ trễ (Latency SLA) |
|---|---|---|---|---|
| **Pha 1: LIVE PLAY MODE (Đang lướt phím)** | Mắt quét khuông nhạc phía trước, tay đang đàn | 1. Con trỏ khuông nhạc (Smooth Cursor) trượt mượt theo vị trí thực tế.<br>2. Đổi màu nhẹ đầu nốt nhạc vừa đánh (Xanh ngọc: chuẩn; Cam: lệch nhịp; Đỏ: sai cao độ).<br>3. Thanh mảnh hiển thị trạng thái Pedal & cân bằng 2 tay ở rìa dưới tầm mắt phụ (Peripheral Vision). | **CẤM:** Popup chữ, câu nhận xét dài, biểu đồ 19 chiều, âm thanh giọng nói chèn ngang (trừ khi ở chế độ *Stop-on-Repeated-Error*). | **$\le 10\text{ ms}$** từ lúc phím đàn chạm đáy tới lúc UI cập nhật (Chạy 100% trên trình duyệt bằng WebAssembly/JS). |
| **Pha 2: GURU REVIEW MODE (Vừa dừng tay sau 1 câu hoặc hết bài)** | Tay rời phím đàn (> 1.5 giây không có MIDI event hoặc chạm nút Stop/Spacebar) | 1. **Bản đồ nhiệt trên khuông nhạc (Score Overlay Heatmap):** Khoanh vùng đúng 2–3 ô nhịp cần chú ý.<br>2. **Thẻ Guru Chẩn đoán (3-1-1 Card)** kèm biểu đồ so sánh trực quan (Velocity của bạn vs. Mục tiêu; Thời điểm nhấc Pedal).<br>3. **Nút "Nghe so sánh" & "Vào Micro-Drill 60s"**. | Tránh hiển thị bảng log hàng nghìn dòng MIDI thô. | **$< 150\text{ ms}$** cho điểm số & biểu đồ kỹ thuật; **$< 600\text{ ms}$** cho dòng chữ đầu tiên (Time-to-First-Token) của lời khuyên Guru. |

---

### Q2.2: Ngân sách độ trễ (Latency Budget) từng mili-giây được tính toán như thế nào để đảm bảo không bao giờ bị giật/trễ?

**Bảng phân bổ Latency Budget chi tiết:**

```text
================================================================================
LUỒNG A: REAL-TIME IN-FLIGHT LOOP (Chạy 100% tại Client Browser - Không qua mạng)
Mục tiêu: Tổng độ trễ từ lúc bấm phím -> Âm thanh & Đổi màu nốt trên màn hình < 12 ms
================================================================================
[USB/BLE MIDI Keyboard]
   │  (~1.0 - 2.5 ms qua cáp USB-MIDI; ~7.5 ms nếu qua Bluetooth MIDI)
   ▼
[Browser WebMIDI API: onmidimessage(event)]
   │  (~0.2 ms)
   ├──► [Nhánh Audio: WebAudio AudioWorklet Synthesizer] ──► Loa/Tai nghe (~3.0 - 5.0 ms)
   │    (Lưu ý: Nếu dùng tiếng đàn sẵn có của Piano điện thì độ trễ âm thanh = 0 ms!)
   │
   └──► [Nhánh Tracking: Client WASM Online Score Follower (OLTW / Windowed HMM)]
        │  • Cập nhật ma trận xác suất vị trí nốt hiện tại trên cửa sổ ±12 nốt: ~0.4 ms
        │  • Xác định trạng thái (Hit / Pitch Error / Extra Note / Early-Late): ~0.1 ms
        ▼
      [Canvas / WebGL / SVG Score Renderer (requestAnimationFrame 120Hz)]
        │  • Cập nhật màu nốt & vị trí con trỏ: ~4.0 - 8.0 ms
================================================================================
TỔNG ĐỘ TRỄ LUỒNG A: ~5.7 ms (USB) — Mượt tuyệt đối như phần mềm Native!
================================================================================

================================================================================
LUỒNG B: POST-PHRASE DEEP GURU EVALUATION (Kích hoạt ngay khi kết thúc đoạn/bài)
Mục tiêu: Hiện bảng điểm & Heatmap < 150 ms; Lời khuyên sư phạm streaming < 600 ms
================================================================================
[User dừng đàn 1.5s hoặc bấm Finish]
   │
   ▼
[Bước 1 - Client/Edge]: Đóng gói chuỗi MIDI Events + Alignment sơ bộ (~5 KB JSON/Binary)
   │  (Thời gian truyền qua WebSocket / HTTP2: ~25 ms)
   ▼
[Bước 2 - Backend Fast-Path (Rust/Python C-Extension + ONNX Runtime)]:
   │  • Tinh chỉnh Note-level Alignment (Backward-Forward Refinement): ~12 ms
   │  • Trích xuất 24 Score Features + 20 Note Features + 51 Segment Descriptors: ~8 ms
   │  • Suy luận Model Chấm điểm (Hierarchical Score-Aligned Transformer + RCG-DMAF):
   │    Chạy bằng ONNX Runtime / PyTorch FP16 cho đoạn 16–32 ô nhịp (~250 nốt): ~18 ms
   │  • Thuật toán Anomaly Pinpointer (Lọc Top-3 ô nhịp lỗi nặng nhất): ~3 ms
   ├──► [TRẢ VỀ NGAY GÓI 1 (WebSocket Push #1 tại mốc ~66 ms)]:
   │    Gồm: Điểm 19 chiều PercePiano, Cấp độ MAJEPPA, Toạ độ Heatmap trên khuông nhạc!
   │    --> User thấy ngay kết quả chấm điểm và vùng khoanh đỏ trên bản nhạc trong < 100 ms!
   │
   └──► [Bước 3 - Backend Slow-Path (Grounded Pedagogical LLM Coach)]:
        • Đưa JSON Chẩn đoán cấu trúc (chỉ ~300 tokens) vào LLM (Claude Haiku 5.5 / Sonnet 5.5
          với Prompt Caching cho toàn bộ kiến thức nhạc lý của bài nhạc):
        • Time-to-First-Token (TTFT): ~350 - 500 ms (Stream từng chữ lên thẻ Guru).
================================================================================
```

**Tại sao kiến trúc 2 tầng (Two-Stage Push) này giải quyết hoàn hảo bài toán UX?**
- Ngay giây phút người dùng ngẩng mặt lên nhìn màn hình sau khi đánh xong câu nhạc (**~70–100 ms**), toàn bộ **Điểm số, Biểu đồ Radar, Thanh đo Voicing/Pedal và Vùng khoanh lỗi trên khuông nhạc đã hiện ra đầy đủ**.
- Trong lúc mắt người dùng nhìn vào ô nhịp bị khoanh đỏ (mất khoảng 0.5 giây), lời giải thích chi tiết bằng ngôn ngữ tự nhiên từ Guru vừa kịp stream ra mượt mà! Không có bất kỳ cảm giác "chờ loading" nào.

---

### Q2.3: Khi đang tập, người học rất hay đánh sai 1 nốt rồi tự ý dừng lại, quay lui 1–2 ô nhịp đánh lại (Backtracking / Stutter Repeat), hoặc nhảy cóc qua đoạn khó. Nếu Score Follower không hiểu hành vi này thì sẽ chấm sai toàn bộ phần sau?

**Lập luận & Giải pháp:**
- Các thuật toán DTW cổ điển giả định đường căn chỉnh là **đơn điệu tiến (monotonic progression)** — hễ người học quay lui đánh lại ô nhịp trước là thuật toán "vỡ trận".
- **Giải pháp trong Piaino:** Sử dụng **Jump-Aware Hidden Markov Model (Nakamura / Hori-Sagayama HMM) kết hợp Online Time Warping có trạng thái nhảy (`OLTWArzt` / `Partitura`)**:
  - Mỗi nốt kỳ vọng trên bản nhạc là một trạng thái ẩn $s_k \in \{1, \dots, M\}$.
  - Ma trận chuyển trạng thái $A_{j \to k}$ có 3 nhóm xác suất:
    1. **Tiến bình thường (Normal Progression):** $k = j$ (đánh lặp nốt/hợp âm), $k = j+1$ (tiến 1 nốt), $k = j+2$ (bỏ sót 1 nốt).
    2. **Quay lui sửa lỗi (Local Backtrack Transition):** Cho phép nhảy lùi về đầu ô nhịp hiện tại hoặc đầu ô nhịp liền trước ($k \in [\text{bar\_start}(j) - 1, j - 1]$) với xác suất phạt nhỏ $\gamma_{\text{back}}$.
    3. **Nhảy vùng tự do (Global Jump Transition):** Khi phát hiện chuỗi $\ge 4$ nốt liên tiếp không khớp với vùng lân cận nhưng khớp hoàn hảo với một ô nhịp khác trên bản nhạc, tự động dịch chuyển con trỏ (re-anchor) tới ô nhịp mới trong $< 15\text{ ms}$.
  - **Lợi ích kép cho tính năng chấm điểm:** Số lần xảy ra chuyển trạng thái `Local Backtrack` tại một ô nhịp chính là chỉ số định lượng trực tiếp của **"Độ ngập ngừng / Vấp ngón (Stutter / Hesitation Rate)"**!

---

## Phần 3: Câu hỏi Ad-hoc về Dataset & Tiền xử lý Dữ liệu

### Q3.1: Đàn piano điện của người dùng ở nhà (Yamaha, Roland, Casio, MIDI Controller) có độ nặng phím và đường cong Velocity khác hẳn đàn đại dương cầm Steinway Disklavier trong dataset. Làm sao tránh việc chấm oan điểm Dynamics?

**Lập luận & Giải pháp (2-Step Hardware Velocity Calibration):**
1. **Hiệu chuẩn phần cứng 15 giây lần đầu kết nối đàn (15-Second Keyboard Calibration Wizard):**
   - Khi người dùng cắm đàn MIDI lần đầu, ứng dụng yêu cầu bấm 3 lần vào phím Đô giữa (C4):
     - Lần 1: Nhẹ nhất có thể phát ra tiếng ($pp \rightarrow v_{\min}$),
     - Lần 2: Lực vừa phải tự nhiên ($mf \rightarrow v_{\text{mid}}$),
     - Lần 3: Mạnh nhất mà không đập phím ($ff \rightarrow v_{\max}$).
   - Hệ thống khớp một hàm ánh xạ phi tuyến đơn điệu (Piecewise Monotonic Spline / Gamma Curve) để đưa `raw_velocity` của cây đàn đó về phân phối chuẩn $[15, 115]$ của Steinway Disklavier:
     $$\tilde{v} = \text{clip}\left( 15 + 100 \cdot \left(\frac{v_{\text{raw}} - v_{\min}}{v_{\max} - v_{\min}}\right)^{\gamma_{\text{user}}}, \, 1, \, 127 \right)$$
2. **Sử dụng Đặc trưng Tương đối Bất biến theo Thang đo (Scale-Invariant Relative Features):**
   - Trong Model đánh giá Voicing và Phrasing, thay vì chỉ dùng giá trị tuyệt đối $v_i$, ta dùng **Z-score cục bộ theo từng đoạn nhạc** và **Độ chênh lệch tương đối giữa hai bè tại cùng thời điểm**:
     $$\Delta v_{\text{voicing}}(t) = \frac{v_{\text{melody}}(t) - \bar{v}_{\text{accompaniment}}(t)}{\sigma_{v,\text{piece}} + \epsilon}$$
   - Nhờ đó, dù đàn của user có nhạy quá mức hay nặng quá mức, tỷ lệ nổi khối của giai điệu so với phần đệm vẫn được đo chính xác tuyệt đối.

---

### Q3.2: Bản nhạc MusicXML từ cộng đồng (MuseScore) hay có lỗi ký âm (như trong `ASAP v1.2` có 18 nốt duration $\le 0$, thiếu gán voice rõ ràng giữa tay phải/tay trái). Hệ thống xử lý thế nào?

**Lập luận & Giải pháp (Automated MusicXML Sanitization & Voice Separation Pipeline):**
Khi một file MusicXML được nạp vào hệ thống (dù từ dataset hay do user/giáo viên upload):
1. **Quarantine & Sửa lỗi Duration $\le 0$:** Quét toàn bộ phần tử `<note>`. Nếu không phải `<grace>` mà có `<duration> <= 0` (như 14 nốt bằng 0 ở Debussy/Ravel và 4 nốt `-142` ở Schumann trong ASAP v1.2), tự động tính lại duration từ thẻ `<type>` (quarter, eighth, 16th), `<dot>` và `<time-modification>` nhân với `<divisions>`.
2. **Unfold Repeats (Trải phẳng dấu quay lại):** Dùng `partitura.score.unfold_part_maximal` hoặc xây dựng đồ thị kịch bản chơi (Playthrough Graph) cho cả 2 biến thể *With-Repeat* và *No-Repeat* để người học chơi kiểu nào cũng căn chỉnh đúng.
3. **Gán nhãn Vai trò Âm nhạc cho từng Nốt (Musical Role Tagging):**
   - Tự động phân tách từng nốt trong hợp âm/ô nhịp thành 3 nhãn vai trò:
     - `ROLE_MELODY (Soprano / Hauptstimme)`: Nốt cao nhất ở khuông khóa Sol hoặc nốt mang đường giai điệu chính (xác định qua thuật toán Skyline + Voice Continuity).
     - `ROLE_BASS (Bass Line)`: Nốt thấp nhất ở khuông khóa Fa.
     - `ROLE_INNER_ACCOMP (Alberti Bass / Inner Harmony)`: Các nốt đệm nằm giữa.
   - Nhãn vai trò này là chìa khóa để Model biết **nốt nào cần đánh nổi lên (voicing), nốt nào cần đánh êm đi**.

---

## Phần 4: Câu hỏi Ad-hoc về Kiến trúc Modeling

### Q4.1: Tại sao không đưa thẳng chuỗi MIDI vào một LLM (như GPT/Claude) để chấm điểm mà phải thiết kế kiến trúc 3 tầng (Symbolic Alignment $\rightarrow$ Neural Assessment + Domain Features $\rightarrow$ Grounded LLM)?

**Bảng đối chứng kỹ thuật:**

| Tiêu chí | Đưa thẳng MIDI vào LLM thuần túy | Kiến trúc 3 tầng của Piaino (Hybrid Symbolic + Neural + Grounded LLM) |
|---|---|---|
| **Độ chính xác toán học trên số thực (Float precision)** | **Rất kém:** LLM xử lý số qua subword tokens, thường xuyên cộng trừ sai mili-giây, không thể tính đúng tích phân chồng lấp Pedal hay phương sai IOI trên 500 nốt. | **Chính xác 100% tới từng micro-giây:** Tầng 1 & 2 tính toán bằng vector/tensor số thực (`float32`), không bao giờ tính nhầm một mili-giây hay một đơn vị velocity nào. |
| **Độ trễ (Latency) & Chi phí** | Gửi 2.000 sự kiện MIDI (~15.000 tokens) vào LLM tốn **3–8 giây** chờ đợi và chi phí token rất đắt cho mỗi lượt tập. | Tầng 1 (WASM) tốn **0.4 ms**; Tầng 2 (ONNX Model) tốn **18 ms**; Tầng 3 chỉ nhận bảng tóm tắt lỗi (~300 tokens) nên vừa rẻ hơn 50 lần vừa nhanh gấp 10 lần. |
| **Huấn luyện trên chuẩn chuyên gia (`PercePiano` & `MAJEPPA`)** | LLM tổng quát chưa từng được tinh chỉnh (fine-tune) trên 19 chiều cảm thụ âm nhạc của PercePiano nên chỉ đoán mò theo chữ. | Tầng 2 được huấn luyện trực tiếp có giám sát (Supervised + Contrastive) trên `PercePiano`, `MAJEPPA` và `PERiScoPe`, đạt tương quan cao nhất với hội đồng giám khảo. |
| **Rủi ro Hallucination (Bịa lỗi ở ô nhịp không có lỗi)** | Rất cao (LLM hay nói *"Ở nhịp 14 bạn đánh sai"* dù nhịp 14 đánh đúng). | **Bằng 0%:** LLM chỉ được phép diễn đạt lại (verbalize) và kê đơn bài tập dựa trên danh sách lỗi đã được Tầng 1 & 2 chứng minh bằng số liệu. |

---

## Phần 5: Câu hỏi Ad-hoc về Kiểm thử & Đảm bảo Chất lượng (Testing Strategy)

### Q5.1: Làm sao kiểm thử (Test) để chứng minh Model đánh giá đúng chuyên môn âm nhạc chứ không phải học vẹt (Overfitting / Shortcut Learning)?

Hệ thống kiểm thử của Piaino bắt buộc vượt qua **4 Tầng Kiểm định (4-Tier Verification Suite)**:

1. **Tier 1 — Deterministic Unit Tests (Kiểm thử Toán học Tuyệt đối):**
   - Tạo các file MIDI tổng hợp biết trước đáp án toán học (ví dụ: dịch nốt C4 trễ chính xác `+50.0 ms`, đặt velocity bè Soprano `= 80`, bè Alto `= 50`, giữ CC64 chồng lấp `+200.0 ms` sau khi đổi hợp âm C sang G7).
   - Kiểm tra hàm trích xuất đặc trưng trả về sai số $< 10^{-6}$.
2. **Tier 2 — Metamorphic & Monotonicity Invariance Tests (Kiểm thử Tính Đơn điệu & Biến đổi có hướng):**
   - Lấy 100 bản chơi chuẩn từ `(n)ASAP`.
   - Áp dụng phép làm nhiễu tăng dần mức độ $\alpha \in \{0.0, 0.2, 0.4, 0.6, 0.8, 1.0\}$ lên từng chiều riêng biệt (ví dụ: làm tăng dần độ lệch nhịp $\alpha$, hoặc làm tăng dần độ nhòe pedal $\alpha$).
   - **Điều kiện Pass bắt buộc:** Điểm dự đoán của Model trên chiều tương ứng phải **giảm đơn điệu nghiêm ngặt (Strictly Monotonic Decreasing: $\hat{y}(\alpha_1) > \hat{y}(\alpha_2)$ với tỷ lệ $> 95\%$)**, VÀ điểm của các chiều không liên quan (Orthogonal Dimensions) không được biến động quá $\pm 0.05$!
   - Đây là bài test quan trọng nhất để đảm bảo: *Khi học viên sửa được lỗi Pedal, điểm Pedal Clarity phải tăng lên ngay lập tức mà không làm nhảy loạn điểm Timing.*
3. **Tier 3 — Group-Disjoint Generalization Benchmark (Chống Leakage theo Tác phẩm):**
   - Đánh giá $R^2$, Spearman Rank Correlation ($\rho$) và Concordance Correlation Coefficient ($CCC$) trên tập test của `PercePiano` và `MAJEPPA` với quy tắc chia tập nghiêm ngặt theo `work_id = (composer, title)` (như đã thiết lập trong [data/asap-dataset/DEEP-UNDERSTAND.md](../data/asap-dataset/DEEP-UNDERSTAND.md)).
4. **Tier 4 — Real-Time Latency & Pedagogical Grounding Audit:**
   - Benchmark tự động 1.000 luồng MIDI streaming giả lập có chứa lỗi nhảy nhịp/quay lui: cam kết `p99 step latency < 5 ms` trên WASM và `0% Hallucinated Measure Numbers` ở đầu ra của LLM Coach.
