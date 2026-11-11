# 06. Kiến trúc Hệ thống, Thiết kế UX/UI, Chiến lược Kiểm thử & Lộ trình Triển khai

> **Mục tiêu tài liệu:** Đặc tả kiến trúc kỹ thuật toàn hệ thống (Frontend, Client Edge WASM, Backend Inference, Teacher-Student Real-time MIDI Relay, Database), thiết kế UX/UI chống quá tải nhận thức và kế hoạch kiểm thử định lượng.

---

## 1. Kiến trúc Hệ thống End-to-End (System Architecture)

Để đảm bảo **Độ trễ khi đang đàn `< 10 ms`** (không phụ thuộc mạng Internet) và **Độ trễ chấm điểm chuyên sâu `< 100 ms`** ngay khi dừng tay, hệ thống áp dụng mô hình **Split-Compute (Tính toán Phân tách Client Edge + Cloud GPU/ONNX)**:

```text
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT BROWSER (Chrome / Edge / iPad)                      │
│                                                                                     │
│  ┌──────────────────┐     ┌──────────────────────────────────────────────────────┐  │
│  │ Đàn Piano Điện   │     │ TẦNG 1: REAL-TIME ENGINE (Chạy 100% Local < 5ms)     │  │
│  │ (USB / BLE MIDI) ├────►│ • WebMIDI Ingress + Hardware Velocity Calibrator     │  │
│  └──────────────────┘     │ • KeyPedalStateMachine (t_on, t_finger_off, t_sound) │  │
│                           │ • WASM Causal Score Follower (OLTW / Jump-HMM)       │  │
│                           │ • Real-time Micro-Drill Rule Verifier                │  │
│                           └───────────┬──────────────────────────┬───────────────┘  │
│                                       │ (< 4ms)                  │                  │
│                                       ▼                          │ (Khi dừng tay    │
│  ┌───────────────────────────────────────────────────────────┐   │  hoặc hết đoạn)  │
│  │ TẦNG HIỂN THỊ & ÂM THANH (UI & Audio Engine)              │   │                  │
│  │ • MusicXML Vector Renderer (OSMD / VexFlow + Canvas 120Hz)│   │                  │
│  │ • Multi-Layer Score Heatmap Overlay (Voicing/Pedal/Timing)│   │                  │
│  │ • Dual-Synth Playback (User Replay vs. Guru Target Morph) │   │                  │
│  └───────────────────────────────────────────────────────────┘   │                  │
└───────────────────────────────────────▲──────────────────────────┼──────────────────┘
                                        │                          │
          WebSocket Push #1 (~70ms):    │                          │ Binary/JSON MIDI
          Điểm 19 chiều + Heatmap       │                          │ Packet (~5 KB)
          WebSocket Push #2 (~450ms):   │                          ▼
          Streaming Lời khuyên Guru     │      ┌──────────────────────────────────────┐
┌───────────────────────────────────────┴──────┤       BACKEND API & AI ENGINE        │
│                                              │       (FastAPI + Rust/C++ + ONNX)    │
│  ┌────────────────────────────────────────┐  │                                      │
│  │ TẦNG 2: FAST INFERENCE (< 45ms Server) │◄─┘                                      │
│  │ • Backward-Forward Note Aligner        │     ┌────────────────────────────────┐  │
│  │ • 44-Dim Note + 51-Dim Global Extractor│────►│ PostgreSQL + Object Storage    │  │
│  │ • Piaino-HCAF-Net (ONNX Runtime FP16)  │     │ • MusicXML Sanitized Corpus    │  │
│  │ • Anomaly Pinpointer & Drill Compiler  │     │ • Student MIDI Session Logs    │  │
│  └────────────────────┬───────────────────┘     │ • Teacher Assignments & Notes  │  │
│                       │                         └────────────────────────────────┘  │
│                       ▼                                                             │
│  ┌────────────────────────────────────────┐     ┌────────────────────────────────┐  │
│  │ TẦNG 3: GROUNDED GURU COACH + TEACHER  │     │ LIVE TEACHER-STUDENT RELAY     │  │
│  │ • LLM Synthesizer (Prompt Cached)      │     │ • WebRTC DataChannel MIDI Sync │  │
│  │ • 2-Min Pre-Lesson Briefing Generator  │     │   (Băng thông < 2 KB/s, 0% mất │  │
│  │ • Auto-Homework Pre-Grader             │     │    dải động velocity/pedal)    │  │
│  └────────────────────────────────────────┘     └────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Thiết kế UX/UI Chuyên sâu: Giải quyết Bài toán Tải nhận thức & Tốc độ

### 2.1. Màn hình 1: `LIVE PLAY MODE` (Tối giản tuyệt đối khi đang đàn)

```text
+-----------------------------------------------------------------------------------+
| [← Bài tập] Nocturne Op. 9 No. 2   | Chế độ: [Biểu cảm Guru ▾] | Tempo: 62 BPM   |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|   KHUÔNG NHẠC CHUẨN (MusicXML Vector - Tự động lật trang sớm 2 ô nhịp trước)      |
|                                                                                   |
|   𝄞  |  ♩.   ♪  |  [Con trỏ mượt màu xanh ngọc đang trượt tại Ô nhịp 12]          |
|   𝄢  |  ♩  ♩  ♩ |  (Chỉ đổi màu nhẹ đầu nốt đã đánh: Xanh = Đúng, Cam = Trễ)      |
|                                                                                   |
+-----------------------------------------------------------------------------------+
| [Thanh Peripheral Vision đáy màn hình - Nhìn bằng khóe mắt, không cần đọc chữ]:   |
|  Cân bằng 2 tay: [Tay trái ████░░░░░░ | ████████░░ Tay phải (Tốt)]                |
|  Độ sạch Pedal : [██████████ Sạch (CC64: 0 -> 127 Syncopated)]                    |
+-----------------------------------------------------------------------------------+
```

### 2.2. Màn hình 2: `GURU REVIEW & MICRO-DRILL MODE` (Hiện ra trong `< 100 ms` ngay khi dừng tay)

```text
+-----------------------------------------------------------------------------------+
| ★ KẾT QUẢ PHÂN TÍCH ĐOẠN (Ô nhịp 1 - 16)                    [Chơi lại] [Tiếp tục] |
+-----------------------------------------------------------------------------------+
| KHUÔNG NHẠC KÈM HEATMAP CHẨN ĐOÁN:                                                |
|  • Ô nhịp 1-11: [Viền Xanh lá - Tốt]                                              |
|  • Ô nhịp 12-13: [Khoanh Đỏ: LỖI ĐỤC PEDAL + TAY TRÁI ĐÈ GIAI ĐIỆU]               |
|    └── Biểu đồ thu nhỏ ngay dưới nhịp 12:                                         |
|        Velocity : ──●── Giai điệu (58)  <  ──■── Đệm tay trái (84) [Bị ngược!]    |
|        Pedal CC64: ███████████████████ (Giữ xuyên qua đổi hợp âm C -> G7: +790ms) |
+-----------------------------------------------------------------------------------+
| 🧙‍♂️ CHẨN ĐOÁN CỦA AI GURU (Quy tắc 3-1-1):                           | 3 TRỤ CỘT:  |
| "Ở ô nhịp 12, khi chuyển sang hợp âm Sol 7, chân bạn vẫn giữ nguyên | • Nhịp : 91 |
|  Pedal thêm 0.79 giây làm tiếng Đô trưởng cũ đè lên hợp âm mới.      | • Ngón : 43 |
|  Đồng thời nốt giai điệu Mi (E5) tay phải đang yếu hơn tay trái."   | • Pedal: 34 |
|                                                                                   |
| [🔊 Nghe bản bạn vừa đàn] vs [✨ Nghe Guru đàn mẫu đoạn nhịp 12-13]               |
| [🎯 VÀO BÀI TẬP SỬA LỖI 60 GIÂY: Tập Nhấc Pedal & Tách bè nhịp 12-13 (0/3 lượt)] |
+-----------------------------------------------------------------------------------+
```

### 2.3. Màn hình 3: `TEACHER STUDIO — 2-MINUTE PRE-LESSON BRIEFING & 1-TAP HOMEWORK REVIEW`

```text
+-----------------------------------------------------------------------------------+
| HỌC VIÊN: Minh Anh (Trình độ: Early Intermediate) | Buổi học bắt đầu sau: 04:12   |
+-----------------------------------------------------------------------------------+
| 📌 TÓM TẮT 6 NGÀY TỰ TẬP TẠI NHÀ (AI Tổng hợp từ 14 phiên MIDI - 185 phút):       |
|  1. Hành vi tập: TỐT (Dành 68% thời gian tập lặp lại đoạn khó nhịp 12-16,         |
|     không đánh lướt vô thức).                                                     |
|  2. Đã tự sửa xong (Nhờ AI Micro-Drill hôm thứ Tư): Lỗi sai nốt F#4 ở nhịp 8.     |
|  3. Điểm nghẽn CẦN THẦY CAN THIỆP HÔM NAY:                                        |
|     ⚠️ Ô nhịp 24-26: Khi tăng tempo lên > 88 BPM, ngón 4-5 tay phải bị dính phím  |
|        (Key Overlap +48ms) và cổ tay trái bắt đầu đập mạnh (vel tăng từ 55 -> 86).|
|                                                                                   |
| [▶ Nghe ngay 15 giây đoạn lỗi nhịp 24-26 (MIDI gốc 100%)]                         |
| [✓ Duyệt nhận xét nháp của AI gửi phụ huynh]  [+ Tạo bài tập tuần tới trong 1-Tap]|
+-----------------------------------------------------------------------------------+
```

---

## 3. Kế hoạch Kiểm thử Toàn diện (Comprehensive Verification & Testing Matrix)

Mọi thành phần từ Parser, Aligner, Feature Extractor tới Neural Model đều phải có bài test tự động chạy bằng `uv run pytest`:

| Mã Test Suite | Mục tiêu kiểm thử | Đầu vào (Input) | Điều kiện Đạt (Pass Assertion) |
|---|---|---|---|
| **`T1: test_key_pedal_state_machine`** | Kiểm tra tính đúng đắn của `t_finger_off` vs `t_sound_off` và `harmonic_mud_sec` | Chuỗi MIDI tổng hợp: bấm C3 tại `0.0s`, nhấc tay tại `1.0s`, giữ `CC64=127` tới `2.5s`, bấm B2-F3 tại `2.0s`. | `t_finger_off(C3) == 1.0`, `t_sound_off(C3) == 2.5`, `harmonic_mud_sec(B2) == 0.500` (sai số $< 10^{-6}$). |
| **`T2: test_musicxml_sanitizer_asap`** | Kiểm tra bộ đọc MusicXML trên toàn bộ 235 score của `data/asap-dataset` | 235 file `.musicxml` trong `ASAP v1.2` (bao gồm 18 nốt có duration $\le 0$ ở Debussy/Ravel/Schumann). | Parse thành công 235/235 file; 100% nốt không phải grace note có `dur_score_beats > 0`; tách đúng `ROLE_MELODY` và `ROLE_BASS`. |
| **`T3: test_jump_aware_aligner`** | Kiểm tra khả năng theo dõi bản nhạc khi người học đánh sai cao độ, bỏ nốt, hoặc dừng lại đánh lại đầu ô nhịp | Lấy 20 bản trong `ASAP`, tiêm 10% lỗi sai cao độ + 3 lần quay lui (backtrack 2 ô nhịp). | Tỷ lệ căn chỉnh đúng các nốt còn lại $\ge 97.5\%$; phát hiện chính xác vị trí 3 điểm `backtrack_count == 1`; thời gian chạy $< 0.8\text{ ms/note}$. |
| **`T4: test_model_monotonic_invariance`** | Kiểm thử tính đơn điệu của Model `Piaino-HCAF-Net` khi lỗi tăng dần | Tạo 5 mức độ suy giảm $\alpha \in \{0.0, 0.25, 0.5, 0.75, 1.0\}$ cho từng loại lỗi (Voicing, Pedal Blur, Timing Jitter, Sticky Fingers). | Điểm `bar_scores` của đúng tiêu chí bị làm nhiễu phải giảm đơn điệu nghiêm ngặt ($\hat{S}(\alpha_k) > \hat{S}(\alpha_{k+1})$ trên $\ge 95\%$ mẫu); các tiêu chí độc lập khác lệch $< 0.05$. |
| **`T5: test_work_disjoint_leakage`** | Chống rò rỉ dữ liệu giữa Train/Val/Test | Manifest chia tập huấn luyện trên `ASAP`, `PercePiano`, `MAJEPPA`, `PERiScoPe`. | Giao của tập `work_id = (composer, title)` giữa Train, Val và Test phải bằng $\emptyset$ tuyệt đối (`len(set_train & set_test) == 0`). |
| **`T6: test_latency_sla_benchmark`** | Đảm bảo tốc độ phản hồi thực tế | Đo thời gian thực thi 1.000 bước trên đoạn nhạc 32 ô nhịp (400 nốt). | `Online Aligner p99 < 2.0 ms`; `Feature Extraction + ONNX HCAF-Net Forward Pass p95 < 45.0 ms` trên CPU/GPU tiêu chuẩn. |

---

## 4. Lộ trình Triển khai Thực tế (4-Phase Implementation Roadmap)

### Phase 1: Core Symbolic Engine & Deterministic Guru MVP (Tuần 1 – 3)
- Xây dựng module `piaino_core` (Python + khả năng biên dịch sang WASM/Rust):
  - `MusicXMLSanitizer` & `RoleTagger` (kiểm thử trực tiếp trên `data/asap-dataset`).
  - `KeyPedalStateMachine` + `JumpAwareScoreAligner`.
  - Bộ trích xuất 44 đặc trưng nốt (`NoteFeatureVector`) + 51 đặc trưng toàn đoạn (`EngineeredGlobalDescriptors`).
  - `PedagogicalPerturbationEngine` để sinh dữ liệu lỗi có nhãn Ground-Truth từ `ASAP v1.2`.

### Phase 2: Huấn luyện Model `Piaino-HCAF-Net` & Chuẩn hóa Đa Dataset (Tuần 4 – 6)
- Tải và tích hợp `(n)ASAP`, `PercePiano` (`EliMasonTech/PercePiano`), `MAJEPPA-Dataset` (`kkwsts/MAJEPPA-Dataset`), và `PERiScoPe v1.1`.
- Huấn luyện `Piaino-HCAF-Net` với hàm loss đa mục tiêu ($\mathcal{L}_{\text{Percep-NLL}} + \mathcal{L}_{\text{Bar-Focal}} + \mathcal{L}_{\text{Monotonic}}$).
- Xuất model sang định dạng `ONNX FP16/INT8` để suy luận cực nhanh (`< 20 ms`).

### Phase 3: Web App Interactive Player & AI Guru Coaching Loop (Tuần 7 – 9)
- Xây dựng giao diện Web (React/Next.js + TypeScript + WebMIDI API + WebAudio Synthesizer + MusicXML Canvas Renderer).
- Triển khai chế độ 2 Pha (`Live Play Mode` và `Guru Review Mode` kèm Heatmap trên khuông nhạc và tính năng nghe so sánh *User vs. Guru Target Morph*).
- Tích hợp `Diagnostic-to-Guru Compiler` và chế độ `Interactive Micro-Drill 60s`.

### Phase 4: Teacher Studio & Real-Time Online MIDI Classroom (Tuần 10 – 12)
- Xây dựng Dashboard cho Giáo viên: `2-Minute Pre-Lesson Briefing`, `Auto-Homework Pre-Screening` và `1-Tap Drill Generator`.
- Tích hợp `WebRTC DataChannel MIDI Relay` để giáo viên và học viên nghe âm thanh MIDI gốc 100% không nén trong giờ học online.
