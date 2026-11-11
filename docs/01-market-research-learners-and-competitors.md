# 01. Khảo sát Thị trường: Nhu cầu Người học Piano & Phân tích Đối thủ (Cập nhật 2026)

> **Mục tiêu tài liệu:** Phân tích toàn diện thị trường ứng dụng học piano hiện tại, bóc tách ranh giới giữa những gì công nghệ hiện có đã giải quyết tốt (**Solved Problems**) và khoảng trống chuyên môn sâu mà chưa sản phẩm thương mại nào làm được (**The Guru Gap**).

---

## 1. Phân khúc người tự học Piano & Nỗi đau thực tế theo từng giai đoạn

Người tự học piano qua ứng dụng không đồng nhất. Nhu cầu đối với "AI Guru" thay đổi hoàn toàn sau mốc 3–6 tháng đầu tiên:

| Giai đoạn | Thời gian học | Mục tiêu chính | Vấn đề gặp phải khi tự học không có thầy |
|---|---|---|---|
| **Giai đoạn 1: Onboarding (Nhập môn)** | Tháng 0 – 3 | Làm quen phím đàn, đọc nốt cơ bản, ghép 2 tay đơn giản, duy trì thói quen mỗi ngày | Sợ nhạc lý khô khan, dễ nản nếu bài tập khó, cần phản hồi đúng/sai tức thì kiểu game. |
| **Giai đoạn 2: Early Intermediate (Sơ trung cấp)** | Tháng 3 – 18 | Đọc bản nhạc chuẩn (Standard Staff), độc lập hai tay, kiểm soát nhịp (pulse), bắt đầu dùng sustain pedal, xử lý câu nhạc (phrasing) | **"Bẫy hoàn thành giả" (Illusion of Competence):** Đánh đúng cao độ nốt theo màn hình cuộn nhưng nhịp bị giật, tay trái đập quá to đè mất giai điệu tay phải, giữ pedal lầy lội làm đục hòa âm, ngón tay bị dính (overlapping notes). |
| **Giai đoạn 3: Intermediate – Advanced (Trung – Cao cấp)** | > 18 tháng | Biểu cảm (Expressiveness), Rubato, Voicing đa bè (Bach fugue, Chopin nocturne), Articulation tinh tế (legato/staccato/portato), tốc độ cao | **Bị các app truyền thống "phạt oan":** Chơi có cảm xúc (co giãn nhịp rubato, ngân nốt đầu câu) thì bị máy chấm điểm trừ vì lệch lưới metronome cứng nhắc. Không biết **tại sao** một đoạn chạy ngón bị vấp (do thế tay/chuyển ngón ở ô nhịp trước hay do căng cổ tay). |

---

## 2. Ma trận khảo sát các đối thủ trên thị trường (2026)

Khảo sát trực tiếp 10 nền tảng tiêu biểu nhất hiện nay chia theo 4 nhóm tiếp cận:

### 2.1. Bảng so sánh tổng quan

| Sản phẩm | Nhóm định vị | Input hỗ trợ | Hiển thị bản nhạc | Chấm Cao độ (Pitch) | Chấm Nhịp (Timing) | Chấm Cường độ (Velocity) | Chấm Voicing / Phrasing | Chấm Pedal (CC64) | Chẩn đoán nguyên nhân lỗi |
|---|---|---|---|---|---|---|---|---|---|
| **Simply Piano** | Gamified Beginner | Mic / MIDI | Cuộn ngang (Scrolling tiles) | Có (Binary) | Yếu (Không tách biệt rõ với pitch) | Không | Không | Không | Không |
| **Flowkey** | Song-first Video | Mic / MIDI | Bản nhạc 1 dòng + Video tay | Có (Wait Mode) | Không chấm khi Wait Mode | Không | Không | Không | Không |
| **Yousician** | Game hóa kiểu Guitar Hero | Mic / MIDI | Falling notes / Staff | Có | Cứng nhắc theo lưới | Không | Không | Không | Không |
| **Skoove** | Guided Course | Mic / MIDI | Standard Staff (1 dòng) | Có | Có (Lưới metronome) | Không | Không | Không | Không |
| **Piano Marvel** | Học thuật / Khảo thí (SASR) | MIDI (Core), Mic (Beta) | Standard Staff đầy đủ | Rất chính xác | Chính xác (Lưới cứng) | Có (Ngưỡng thô) | Không | Không | Không (Chỉ tô đỏ/xanh nốt) |
| **Playground Sessions** | Pop/Groove & Co-founder Quincy Jones | MIDI | Standard Staff / Falling | Có | Có (Lưới cứng) | Không | Không | Không | Không |
| **Playsheet / SightPiano** | Luyện đọc bản nhạc với MIDI | MIDI | Standard Staff (MusicXML) | Có | Early / On-time / Late | Hiển thị thô, k chấm | Không | Không | Không |
| **MuseFlow** | Sight-reading sinh tự động | MIDI | Standard Staff | Có | Có (Lưới cứng) | Không | Không | Không | Không |
| **ROLI Learn + Airwave** | Phần cứng phím sáng + Camera hồng ngoại 3D | MIDI + IR Hand Tracking | Falling / Staff | Có | Có | Có (Keystrike) | Không | Không | Một phần về tư thế bàn tay (phụ thuộc phần cứng đắt tiền) |
| **Tonebase** | Masterclass cho người chơi trung-cao cấp | Không (Xem video thụ động) | PDF tĩnh | Không | Không | Không | Giảng giải qua video, k chấm | Giảng giải qua video | Không có AI tương tác |

---

## 3. Phân tích sâu: Đối thủ ĐÃ LÀM ĐƯỢC gì? (Commoditized Features)

Những tính năng dưới đây thị trường đã làm rất tốt — sản phẩm mới **bắt buộc phải có ở mức tương đương hoặc tốt hơn (Table Stakes)**, nhưng không thể chỉ dựa vào chúng để tạo lợi thế cạnh tranh:

1. **Wait Mode & Loop Practice (Flowkey, Piano Marvel, Playsheet):**
   - Dừng con trỏ bản nhạc chờ đến khi người học bấm đúng tập hợp pitch của hợp âm/nốt hiện tại mới nhảy sang nốt tiếp theo.
   - Cắt nhỏ bài nhạc theo từng đoạn (Measure slicing / Looping) và tách riêng tay phải (RH) / tay trái (LH).
2. **Chấm điểm Sight-Reading tiêu chuẩn hóa (Piano Marvel SASR):**
   - Kiểm tra khả năng thị tấu qua 90 cấp độ nhỏ, thích ứng độ khó (Adaptive testing) dựa trên % nốt đúng + nhịp đúng so với metronome.
3. **Onboarding giảm ma sát và tạo thói quen (Simply Piano, Duolingo Music):**
   - Chia bài tập thành các chặng 5 phút (Micro-workouts), nhạc đệm (Backing track) bắt tai giúp người mới cảm thấy mình đang "chơi nhạc" ngay từ tuần đầu tiên.
4. **Import MusicXML/MIDI tùy chỉnh (Piano Marvel, Playsheet, Soundslice):**
   - Cho phép người học tải lên bản nhạc `.musicxml` hoặc `.mid` bất kỳ từ MuseScore để luyện tập.

---

## 4. Phân tích sâu: Đối thủ CHƯA LÀM ĐƯỢC gì? ("The AI Guru Gap")

Đây chính là vùng đại dương xanh (Blue Ocean) cho một **AI Piano Guru** sử dụng tín hiệu MIDI chuyên sâu:

### 4.1. "Ách thống trị của Metronome" (Metronomic Tyranny) vs. Nhịp biểu cảm (Expressive Timing & Rubato)
- **Thực trạng đối thủ:** Mọi app hiện tại (kể cả Piano Marvel) tính điểm timing bằng công thức cứng:
  $$|\text{t}_{\text{user}} - \text{t}_{\text{grid}}| < \tau_{\text{tolerance}}$$
- **Hệ quả:** Khi người học chơi nhạc Cổ điển (Chopin, Debussy, Romantic) hoặc Ballad, việc thực hiện *ritardando* ở cuối câu, *agogic accent* (ngân nhẹ nốt đỉnh câu), hay *rubato* làm toàn bộ các nốt sau đó bị lệch pha so với đồng hồ tuyến tính $\text{t}_{\text{grid}}$. App báo đỏ hàng loạt ("Late"), buộc người học phải chơi cứng đơ như máy đánh chữ để lấy điểm 100%.
- **Điều người học cần ở AI Guru:**
  - Phân biệt được **Lỗi nhịp vô thức (Uncontrolled Rhythmic Instability / Rushing / Dragging)** với **Biến thiên nhịp có chủ đích nghệ thuật (Structured Phrase-Arch Rubato)**.
  - Đánh giá **Tính ổn định nội tại (Local Tempo Smoothness)** và sự co giãn nhịp theo cấu trúc câu nhạc (Phrase boundary).

### 4.2. Mù màu về Cường độ & Phân tách Bè (Dynamics, Polyphonic Voicing & Balance)
- **Thực trạng đối thủ:**
  - Hầu hết các app bỏ qua byte `velocity` (0–127) của gói tin MIDI Note-On, hoặc chỉ kiểm tra `velocity > ngưỡng_tối_thiểu`.
  - Khi người học bấm một hợp âm 4 nốt (ví dụ C4-E4-G4 ở tay trái và C5 là nốt giai điệu ở tay phải) với `velocity = [85, 82, 88, 60]`, Piano Marvel chấm **100% hoàn hảo**. Nhưng một giáo viên piano thực thụ sẽ lập tức dừng lại: *"Giai điệu ở ngón út tay phải (C5 = 60) bị chìm nghỉm dưới phần đệm tay trái (85)!"*
- **Điều người học cần ở AI Guru:**
  - **Melody-to-Accompaniment Voicing Ratio:** Đo độ chênh lệch velocity ($\Delta v$) giữa bè giai điệu (Soprano/Melody voice trong MusicXML) và bè đệm (Alberti bass, chordal accompaniment).
  - **Dynamic Contour Tracking:** Kiểm tra đường cong *crescendo* / *diminuendo* trên 4–8 ô nhịp thông qua hồi quy tuyến tính/đa thức trên chuỗi velocity thay vì chấm từng nốt rời rạc.
  - **Chord Voicing & Micro-Asynchrony:** Phát hiện hiện tượng "bấm hợp âm bị tòe" (unintended chord spread > 30ms) so với việc chủ động làm nổi bật nốt cao nhất (melody lead ~10–18ms).

### 4.3. Điểm mù về Articulation & Độ đồng đều của Ngón (Touch, Legato/Staccato & Finger Evenness)
- **Thực trạng đối thủ:** Chỉ bắt sự kiện `Note-On`, hoàn toàn phớt lờ sự kiện `Note-Off`!
- **Hệ quả:**
  - Người học đánh nốt *staccato* nhưng giữ phím quá lâu, hoặc đánh câu *legato* nhưng nhấc tay sớm tạo khoảng trống (gap) hoặc giữ dính phím quá mức (legato overlap quá dày làm mờ câu chạy ngón).
  - Khi chạy gam (scales) hoặc *trill*, ngón 4 và ngón 5 yếu làm khoảng cách thời gian giữa các nốt (IOI - Inter-Onset Interval) và velocity bị khập khiễng ("chập - nhanh - chập"), nhưng do vẫn nằm trong cửa sổ $\pm 80\text{ ms}$ của app nên vẫn được chấm xanh.
- **Điều người học cần ở AI Guru:**
  - Tính chính xác **Articulation Ratio** từng nốt: $\log_2(\text{Duration}_{\text{played}} / \text{Duration}_{\text{score}})$ để chấm đúng chuẩn *staccato*, *portato*, *non-legato*, *legato*.
  - Đo **Key Overlap Time (KOT)** bằng mili-giây giữa hai nốt liên tiếp trong một slur để phát hiện ngón bị dính ("sticky fingers" ở ngón 3-4-5).
  - Đo **Evenness Index** (độ lệch chuẩn cục bộ của IOI và Velocity) trong các đoạn chạy ngón 16th-notes.

### 4.4. Hoàn toàn bỏ trống Đánh giá Pedal (Sustain Pedal CC64 & Harmonic Clarity)
- **Thực trạng đối thủ:** Không một app phổ thông nào đọc và chấm điểm MIDI Control Change `CC64` (Damper/Sustain Pedal), `CC66` (Sostenuto), `CC67` (Una Corda).
- **Hệ quả:** Lỗi phổ biến nhất của người tự học là **dậm pedal cùng lúc với tay xuống phím** (Direct pedaling thay vì Syncopated/Legato pedaling) hoặc **quên nhấc pedal khi đổi hợp âm**, tạo ra âm thanh đục ngầu (harmonic mud).
- **Điều người học cần ở AI Guru:**
  - Theo dõi liên tục trạng thái `CC64 (0–127)`.
  - Phát hiện **Pedal Bleed / Harmonic Blur:** Khi hòa âm trên bản nhạc đổi từ hợp âm $H_1$ sang $H_2$ (ví dụ từ $G^7$ sang $C$), kiểm tra xem `CC64` có được nhấc xuống $< 20$ trong khoảng $[t_{\text{onset}}(H_2), t_{\text{onset}}(H_2) + 120\text{ ms}]$ (Syncopated pedal change) hay không.
  - Phân biệt **Finger Legato** với **Pedal Cheating:** Phát hiện khi người học nhấc tay quá sớm (thiếu finger legato) rồi dùng pedal để "ăn gian" nối âm.

### 4.5. Thiếu Tư duy Sư phạm & Chẩn đoán Nguyên nhân gốc rễ (Pedagogical Root-Cause & Prescriptive Drills)
- **Thực trạng đối thủ:** Phản hồi mang tính **Mô tả triệu chứng (Descriptive Symptom)**: *"Bạn đạt 84% độ chính xác. Sai 5 nốt ở ô nhịp 12, 13."* Người học nhìn vào con số 84% không biết phải sửa như thế nào ngoài việc... đánh lại từ đầu bài đến cuối bài trong vô vọng.
- **Điều người học cần ở AI Guru:** Phản hồi mang tính **Chẩn đoán nguyên nhân & Kê đơn bài tập (Diagnostic & Prescriptive Coaching)**:
  1. **Chỉ ra nguyên nhân nhân quả:** *"Lỗi trượt hợp âm ở phách 1 ô nhịp 12 xuất phát từ việc ở phách 4 ô nhịp 11 bạn giữ nốt bass quá lâu (98% duration) không kịp chuyển vị trí tay trái nhảy quãng 12."*
  2. **Kê đơn phương pháp luyện tập cụ thể (Actionable Micro-Drill):** Tự động tạo 1 bài tập ngắn 3 bước ngay trên đoạn lỗi (ví dụ: *Bước 1: Tập riêng chuyển vị trí tay trái từ phách 4 nhịp 11 sang phách 1 nhịp 12 với staccato nhanh; Bước 2: Tập nhịp chấm dôi (dotted rhythm) cho tay phải; Bước 3: Ghép 2 tay ở tempo 65%*).

---

## 5. Tổng kết Định vị Sản phẩm (Product Positioning Matrix)

```text
                       ĐỘ SÂU CHUYÊN MÔN & BIỂU CẢM (MUSICAL DEPTH)
                                        ▲
                                        │
                                        │      ★ [PIAINO AI GURU]
                                        │        (MIDI + Score-Aligned AI:
            [Tonebase / Masterclass]    │         Voicing, Rubato, Pedal CC64,
            (Chuyên sâu nhưng thụ động, │         Articulation, Root-Cause Coach)
             không có tương tác/chấm)   │
                                        │
  THỤ ĐỘNG ◄────────────────────────────┼────────────────────────────────► TƯƠNG TÁC
 (PASSIVE)                              │                                (INTERACTIVE)
                                        │      [Piano Marvel]
                                        │      (Mạnh về đọc nốt/SASR,
                                        │       nhưng chấm theo lưới cứng)
                                        │
            [YouTube Synthesia]         │      [Simply Piano / Flowkey / Yousician]
                                        │      (Chỉ chấm đúng/sai nốt cơ bản)
                                        ▼
                         CƠ BẢN / ĐÚNG-SAI NHỊ PHÂN (BINARY HIT/MISS)
```

**Kết luận chiến lược từ Khảo sát 1:**
Thị trường đã bão hòa ở phân khúc *"Game bấm phím đúng nốt cho người mới bắt đầu"*, nhưng hoàn toàn bỏ trống phân khúc **"Huấn luyện viên kỹ thuật & biểu cảm chuyên sâu qua MIDI"** cho người học nghiêm túc (Late Beginner $\rightarrow$ Advanced). Việc tận dụng toàn bộ thông tin vật lý sẵn có trong chuẩn MIDI (`Note-On`, `Note-Off`, `Velocity`, `Micro-timing`, `CC64/66/67`) kết hợp với cấu trúc bản nhạc (`MusicXML`) cho phép xây dựng một AI Guru vượt trội hoàn toàn các đối thủ hiện tại mà không cần phần cứng camera đắt đỏ.
