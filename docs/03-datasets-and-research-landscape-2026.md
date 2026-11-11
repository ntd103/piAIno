# 03. Khảo sát Datasets, Nghiên cứu SOTA & Notebooks Mã nguồn mở (Cập nhật 2026)

> **Mục tiêu tài liệu:** Tổng hợp toàn bộ hệ sinh thái dữ liệu (Datasets), bài báo khoa học (Papers 2020–2026) và kho mã nguồn/Notebooks thực nghiệm cho bài toán **Đánh giá & Huấn luyện Biểu diễn Piano qua MIDI (Automatic Piano Performance Assessment & Alignment)**. Phân tích ưu/nhược điểm của từng bộ dữ liệu (bao gồm `ASAP v1.2` đang có sẵn tại [data/asap-dataset/](../data/asap-dataset/)) và thiết kế chiến lược dữ liệu hợp nhất.

---

## 1. Nhận định về Dataset gợi ý hiện tại (`ASAP v1.2`) và Tại sao cần mở rộng

Tại thư mục [data/asap-dataset/](../data/asap-dataset/) (chi tiết đã phân tích trong [data/asap-dataset/DEEP-UNDERSTAND.md](../data/asap-dataset/DEEP-UNDERSTAND.md)):
- **Những gì `ASAP v1.2` gốc có:** 222 tác phẩm logic, 235 biến thể bản nhạc (`xml_score` + `midi_score`), 1.067 bản biểu diễn MIDI (`midi_performance`), kèm nhãn vị trí `beat`, `downbeat`, `time_signature`, `key_signature` và `downbeats_score_map`.
- **3 Giới hạn chí mạng nếu chỉ dùng `ASAP v1.2` gốc để làm AI Piano Guru:**
  1. **Thiếu liên kết từng nốt (No Note-Level Alignment):** `ASAP v1.2` gốc chỉ căn chỉnh tới cấp độ **Phách (Beat/Downbeat)**. Để đánh giá *độ trễ từng nốt (micro-timing onset deviation)*, *tỷ lệ ngắt/nối âm từng nốt (note articulation ratio)*, hay *cân bằng âm lượng giữa nốt giai điệu và nốt đệm (polyphonic voicing)*, bắt buộc phải biết chính xác nốt thứ $i$ trong `midi_performance` ứng với `<note>` thứ $j$ nào trong `xml_score`.
  2. **Thiếu nhãn đánh giá cảm thụ & sư phạm (No Perceptual / Pedagogical Labels):** `ASAP` chỉ chứa bản chơi của các nghệ sĩ/thí sinh trình độ cao (từ MAESTRO và cuộc thi), không có điểm chấm của chuyên gia về các tiêu chí như *độ sạch của pedal, độ đều của ngón, sự cân bằng hai tay, độ thuyết phục của câu nhạc*.
  3. **Thiên lệch trình độ (Skill-Level Bias):** Hầu hết bản thu trong ASAP là của người chơi chuyên nghiệp. Nếu chỉ train trên ASAP, model sẽ không nhận diện tốt các mẫu lỗi đặc trưng của người mới học và trung cấp (ngập ngừng tìm phím, đánh trượt quãng, vội nhịp ở đoạn dễ, khựng nhịp ở đoạn khó).

$\rightarrow$ **Giải pháp:** Phải kết hợp `ASAP` với **6 bộ dữ liệu thế hệ mới (2022–2026)** dưới đây.

---

## 2. Bản đồ Toàn cảnh các Dataset Piano MIDI Mới nhất (2022–2026)

### 2.1. Bảng so sánh chi tiết 8 Datasets cốt lõi

| Dataset (Năm) | Quy mô | Cấp độ Alignment | Nhãn Đánh giá / Kỹ năng | Nguồn tải / Repo chính thức | Vai trò trong dự án Piaino |
|---|---|---|---|---|---|
| **1. (n)ASAP** *(TISMIR 2023)* | 222 bài, 1.067 perf (~100h), **>7.000.000 nốt** | **Note-level** (đã kiểm duyệt thủ công) + Beat/Downbeat | Không có nhãn điểm; là chuẩn vàng về căn chỉnh nốt-với-bản-nhạc | GitHub: `CPJKU/asap-dataset` | **Nâng cấp trực tiếp** cho `data/asap-dataset` hiện tại để lấy ánh xạ 1-1 từng nốt (`match` files). |
| **2. PercePiano** *(Scientific Reports, 10/2024)* | 1.202 đoạn (4, 8, 16 ô nhịp), 12.652 lượt chấm từ chuyên gia | **Note-level** với MusicXML (bản chuẩn Urtext Henle) | **19 chiều cảm thụ (Perceptual Features)** chia làm 8 nhóm điểm liên tục $[0, 1]$ (Mean + Std) | GitHub: `JonghoKimSNU/PercePiano` <br> HF: `EliMasonTech/PercePiano` | **Dataset quan trọng số 1** để huấn luyện đầu ra chấm điểm chuyên môn đa chiều (Timing, Touch, Pedal Clarity, Voicing Balance, Interpretation). |
| **3. MAJEPPA-Dataset** *(ISMIR 2026)* | ~4.000 bản thu phân đoạn có cặp `(score_midi, perf_midi)` | Segment & Note-aligned | **6 cấp độ người chơi** (`child_beginner` $\rightarrow$ `virtuoso`) $\times$ **6 bối cảnh** (`practice`, `sight_read`, `slow_demo`, `concert`...) | GitHub: `kkwsts/majeppa` <br> HF: `kkwsts/MAJEPPA-Dataset` | **Dataset quan trọng số 2** để model hiểu phổ kỹ năng từ người mới học (vấp, tập chậm, thị tấu) tới nghệ sĩ. |
| **4. PERiScoPe v1.1** *(SyMuPe, 01/2026)* | **35.815 cặp** Score-Performance (2.848 giờ, **88.4 triệu nốt**), 66 nhạc sĩ | **Note-level** (đã làm sạch bằng `parangonar` + nội suy nốt thiếu) | Chuẩn biểu diễn biểu cảm (Expressive Reference Distribution) từ nhiều nghệ sĩ cho cùng 1 bài | HF: `SyMuPe/PERiScoPe` | Huấn luyện **Mô hình Chuẩn Tham chiếu Biểu cảm (Reference Expressive Prior)** — biết một câu nhạc được phép rubato/voicing trong biên độ nào. |
| **5. Aria-MIDI** *(ICLR 2025)* | **1.186.253 file MIDI** (>100.629 giờ solo piano) | Không có score (chỉ Performance MIDI) | Metadata về thể loại, nhạc sĩ, chất lượng bản thu | GitHub: `loubbrad/aria-midi` <br> HF: `loubb/aria-midi` | Dùng **Pretrained Backbone** (`piano-medium-double-1.0.safetensors`) thay vì phải tự pretrain từ đầu. |
| **6. ATEPP** *(ISMIR 2022)* | 11.674 bản thu (~1.000 giờ) của 49 danh cầm trên 1.595 chương nhạc | Tổ chức theo tác phẩm/chương; ~50% có MusicXML | Nhãn danh cầm (49 nghệ sĩ hàng đầu thế giới) | GitHub: `tangjjbetsy/ATEPP` | Học đa dạng phong cách biểu cảm (Multiple Valid Interpretations) cho cùng một tác phẩm. |
| **7. PIG (Piano Fingering)** *(Nakamura et al.)* | 150 bản nhạc có gán nhãn ngón tay đầy đủ | Note-level | Nhãn bàn tay (L/R) và số ngón (`1, 2, 3, 4, 5` / `-1, -2, -3, -4, -5`) | Trang chủ Kyoto Univ / Eita Nakamura | Huấn luyện module **Dự đoán Ngón bấm tối ưu & Chẩn đoán lỗi kẹt thế tay (Biomechanical Fingering Bottleneck)**. |
| **8. SMD / Vienna 4x22 / ConEspressione** | Các tập nhỏ chất lượng phòng thu trên đàn Disklavier / Bösendorfer CEUS | Note-level chính xác tuyệt đối từ cảm biến cơ học phím/pedal | Biểu cảm & độ lệch micro-timing, velocity chuẩn vật lý | Công khai cho nghiên cứu MIR | Dùng làm tập **Golden Benchmark** để kiểm định độ chính xác của thuật toán trích xuất đặc trưng. |

---

## 3. Phân tích sâu 3 Dataset & Nghiên cứu Đột phá Gần đây (2024–2026)

### 3.1. PercePiano (Park et al., *Nature Scientific Reports*, 10/2024) & Các nghiên cứu kế thừa năm 2026
- **Bài toán giải quyết:** Trước PercePiano, các hệ thống đánh giá chỉ chấm điểm tổng quát (Overall grade 1–5) hoặc đếm lỗi nốt. PercePiano xây dựng bộ 19 tiêu chí cảm thụ lưỡng cực (bipolar scales chuẩn hóa về $[0, 1]$ bằng Item Response Theory) thuộc 8 nhóm chuyên môn:
  1. **Timing (Nhịp độ):** `stable_beat_vs_unstable_beat` (Nhịp vững vs. Nhịp chông chênh).
  2. **Articulation (Ngắt/Nối âm):** `short_vs_long` (Staccato/Ngắn vs. Legato/Dài), `cushioned_vs_solid` (Đầu ngón mềm/xốp vs. Chắc/rõ nét).
  3. **Pedaling (Sử dụng Pedal):** `sparse/dry_vs_saturated/wet` (Ít pedal/khô vs. Nhiều pedal/ướt), `clean_vs_blurred` (Sạch/rõ hòa âm vs. Đục/nhòe âm).
  4. **Timbre (Âm sắc quy chiếu qua tương tác phím):** `even_vs_colorful` (Đều/đơn sắc vs. Đa dạng màu sắc), `shallow_vs_rich` (Mỏng/nông vs. Dày/sâu), `mellow_vs_bright` (Ấm/dịu vs. Sáng/anh ánh), `soft_vs_loud` (Nhỏ vs. To).
  5. **Dynamics (Biên độ động):** `little_dynamic_range_vs_large_dynamic_range` (Biên độ hẹp/phẳng vs. Tương phản động rộng).
  6. **Musicality (Tính âm nhạc & Cân bằng):** `fast_vs_slow_tempo` (Tempo nhanh vs. chậm), `cramped_vs_spacious` (Bức bối/vội vs. Thoáng/có khoảng thở câu nhạc), `flat_vs_protruding_balance` (Các bè phẳng/lẫn vào nhau vs. Tách bè nổi khối rõ ràng), `calm_vs_dramatic` (Tĩnh tại vs. Kịch tính).
  7. **Mood (Cảm xúc truyền tải):** `sad_vs_happy` (Valence), `low_energy_vs_high_energy` (Energy), `plain_vs_imaginative` (Đơn điệu vs. Giàu trí tưởng tượng).
  8. **Interpretation (Diễn giải tổng thể):** `unsophisticated_vs_sophisticated` (Thô mộc vs. Tinh tế), `doubtful_vs_convincing` (Thiếu thuyết phục/ngập ngừng vs. Thuyết phục/chắc chắn).
- **Phát hiện kỹ thuật cốt lõi từ bài báo PercePiano (2024) và RCG-DMAF-Net (10/2026):**
  - Nếu chỉ đưa Performance MIDI thuần túy vào `Bi-LSTM` hoặc `MidiBERT-Piano` (không đối chiếu với bản nhạc MusicXML), $R^2$ trên tập Piece-Split chỉ đạt **0.185** (Bi-LSTM) và **0.313** (MidiBERT).
  - Khi **Grounding (neo) Performance MIDI vào MusicXML Score** và mã hóa theo cấu trúc phân cấp **Note $\rightarrow$ Voice (Bè) $\rightarrow$ Beat (Phách) $\rightarrow$ Bar (Ô nhịp)**, điểm $R^2$ tăng vọt thêm **+21.2%**!
  - Nghiên cứu mới nhất **RCG-DMAF-Net (Jin, 10/2026)** chứng minh việc kết hợp **Nhánh chuỗi sự kiện nốt (Sequential Note Events)** với **Nhánh 51 đặc trưng kỹ thuật trích xuất thủ công (51 Engineered Domain Descriptors)** qua cơ chế *Residual Cross-Gating* giúp đẩy $R^2$ trên tập Group-Disjoint (chống rò rỉ dữ liệu tuyệt đối) lên **0.4765 – 0.4979** (RMSE ~ 0.087 trên thang $[0,1]$).
  - Nghiên cứu **Dhiman (01/2026, arXiv:2601.19029)** chỉ ra rằng nếu kết xuất MIDI qua bộ tổng hợp vật lý nhẹ (như Pianoteq) rồi trích đặc trưng tầng 9–12 của **MuQ** kết hợp với đặc trưng Symbolic, các chiều khó như *Pedal Clarity* (+0.279 $R^2$) và *Dynamic Range* (+0.295 $R^2$) được cải thiện rõ rệt.

### 3.2. MAJEPPA (Chang, Dixon, Benetos — Queen Mary Univ. of London, ISMIR 2026)
- **Đột phá:** Lần đầu tiên giải quyết bài toán mô hình hóa đồng thời **Trình độ người chơi** (`child_beginner`, `adult_beginner`, `adult_intermediate`, `child_professional`, `piano_teacher`, `virtuoso`) và **Trạng thái chơi** (`practice`, `sight_read`, `slow_demo`, `demo_class`, `performance`, `concert_performance`) trong một không gian nhúng thống nhất (Joint-Embedding Predictive Architecture).
- **Kiến trúc:** Sử dụng backbone `Aria-medium` gắn `LoRA adapter`, huấn luyện đồng thời với hàm mục tiêu kép:
  1. Causal Next-Token Prediction (NTP) để hiểu ngữ pháp âm nhạc.
  2. Block-diagonal InfoNCE + Supervised Contrastive Loss (`SupCon`) giữa biểu diễn của Bản nhạc (Score) và Bản chơi (Performance).
- **Ứng dụng trực tiếp cho Piaino:** Không chỉ chấm điểm, kiến trúc kiểu MAJEPPA cho phép thực hiện **"Performance Morphing / Target Demonstration"** — tức là từ bản chơi còn lỗi của học viên (`adult_beginner`), model có thể dịch chuyển vector trong không gian nhúng sang trạng thái `piano_teacher` hoặc `slow_demo` của chính đoạn nhạc đó để **phát lại cho học viên nghe sự khác biệt ngay lập tức**!

### 3.3. Matchmaker (`pymatchmaker`, ISMIR 2025/2026) & `partitura` / `parangonar` (CPJKU)
- Hệ sinh thái mã nguồn mở từ nhóm nghiên cứu CPJKU (Áo) và KAIST/QMUL đã chuẩn hóa hoàn toàn tầng xử lý dữ liệu:
  - **`partitura`:** Đọc MusicXML,展开 (unfold) các dấu lặp (repeats), tách voice, xuất danh sách note kèm vị trí metrical chính xác.
  - **`parangonar`:** Cung cấp các thuật toán căn chỉnh nốt-với-nốt (Note-level alignment) chính xác nhất hiện nay (bao gồm Dynamic Time Warping cải tiến, HMM của Nakamura, và Segment-Constrained Alignment).
  - **`pymatchmaker`:** Thư viện **Real-time Score Following** chạy trực tuyến (streaming) với độ trễ mỗi bước suy luận chỉ **0.07 ms (OLTWArzt) đến 3.59 ms (HMM)** — hoàn toàn đáp ứng yêu cầu phản hồi thời gian thực.

---

## 4. Danh mục Kho mã nguồn & Notebooks Thực nghiệm Cần Tham khảo

Dưới đây là các repository và notebook thực tế từ cộng đồng nghiên cứu có thể dùng làm nền tảng đối chiếu và tái sử dụng:

1. **Notebooks có sẵn ngay trong workspace hiện tại:**
   - [data/asap-dataset/util/full_ann_pipeline.ipynb](../data/asap-dataset/util/full_ann_pipeline.ipynb): Quy trình trích xuất beat/downbeat từ MusicXML qua `music21` và đối chiếu với symbolic aligner.
   - [data/asap-dataset/util/check_dataset_quality.ipynb](../data/asap-dataset/util/check_dataset_quality.ipynb): Cách kiểm tra tính toàn vẹn của alignment và phát hiện ngoại lai (outliers).
2. **PercePiano & VirtuosoNet Feature Extraction (`JonghoKimSNU/PercePiano` & `pyScoreParser`):**
   - Cách trích xuất 24 đặc trưng bản nhạc (Score features: vị trí phách, slur, grace note, dynamic marking, fermata...) và các đặc trưng biểu diễn đã căn chỉnh (tempo QPM cục bộ, độ lệch onset, tỷ lệ articulation, trạng thái pedal tại từng nốt).
3. **Note-level Alignment (`sildater/parangonar` & `CPJKU/asap-dataset`):**
   - Cung cấp `NoteMatcher`, `AutomaticNoteMatcher`, `DualDTWNoteMatcher` và giao diện web `Parangonada` để trực quan hóa đường nối giữa nốt kỳ vọng trên bản nhạc và nốt thực tế người dùng đánh.
4. **MAJEPPA (`kkwsts/majeppa`):**
   - Mã nguồn huấn luyện `LLMJEPADataModule`, cách biểu diễn cặp `(score_midi, perf_midi)` và hàm loss đối chiếu đa cấp độ kỹ năng.

---

## 5. Chiến lược Dữ liệu Hợp nhất cho Dự án Piaino (4-Pillar Data Strategy)

Vì không một dataset đơn lẻ nào chứa đủ cả **(1) Lỗi sai của người mới học**, **(2) Chuẩn biểu cảm của nghệ sĩ**, và **(3) Nhãn chấm điểm 19 chiều chi tiết**, Piaino sẽ xây dựng Pipeline dữ liệu gồm **4 trụ cột**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   PIAINO UNIFIED DATA PIPELINE                              │
├──────────────────────┬──────────────────────┬───────────────────────────────┤
│ Trụ cột 1:           │ Trụ cột 2:           │ Trụ cột 3:                    │
│ (n)ASAP + PERiScoPe  │ PercePiano           │ MAJEPPA + PIG                 │
│ (35,815 cặp chuẩn)   │ (1,202 đoạn, 19 nhãn)│ (4,000 đoạn 6 cấp độ + Ngón)  │
└──────────┬───────────┴──────────┬───────────┴──────────────┬────────────────┘
           │                      │                          │
           ▼                      ▼                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Trụ cột 4: PEDAGOGICAL PERTURBATION ENGINE (Bộ sinh dữ liệu lỗi có kiểm soát)│
│  • Tiêm lỗi Cao độ (Pitch slip ±1..2 semitones theo cấu trúc bàn phím)      │
│  • Tiêm lỗi Nhịp (Rushing ở đoạn dễ, Hesitation/Stutter trước hợp âm khó)   │
│  • Tiêm lỗi Voicing (Tăng velocity tay trái, đè chìm bè Soprano)            │
│  • Tiêm lỗi Articulation (Cắt cụt legato hoặc làm dính ngón 3-4-5)          │
│  • Tiêm lỗi Pedal CC64 (Dậm pedal sớm/quên nhấc khi đổi hòa âm -> Blur)     │
│  ==> Tự động gán nhãn Ground-Truth vị trí lỗi & mức độ suy giảm điểm!       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tại sao "Trụ cột 4 (Pedagogical Perturbation Engine)" lại là vũ khí bí mật?
- Trong thực tế, việc thuê chuyên gia chấm điểm từng ô nhịp cho 100.000 bản thu của người mới học là bất khả thi về chi phí.
- Nhưng nếu lấy một bản chơi chuẩn đã được căn chỉnh hoàn hảo từ **(n)ASAP / PERiScoPe**, sau đó áp dụng các **Phép biến đổi suy giảm kỹ thuật có tham số (Parametric Technical Perturbations)** tại các ô nhịp xác định (ví dụ: dịch chuyển sự kiện nhấc Pedal `CC64=0` ở ô nhịp 8 trễ đi `+350ms` qua thời điểm đổi hợp âm), ta biết **chính xác 100% Ground-Truth**:
  - Ô nhịp 8 bị lỗi `pedal_blur = 0.85`,
  - Các ô nhịp 1–7 và 9–16 hoàn toàn sạch (`pedal_blur = 0.05`).
- Phương pháp **Self-Supervised Pedagogical Perturbation** này giúp model học được khả năng **Định vị chính xác lỗi tới từng nốt và từng ô nhịp (Pinpoint Localization)** mà không bị hiện tượng đoán mò theo cảm tính toàn cục!
