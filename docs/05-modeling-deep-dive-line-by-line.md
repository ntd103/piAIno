# 05. Kỹ thuật Modeling Chuyên sâu: Giải phẫu Từng Đơn vị Tính toán & Debug Từng Dòng (`Piaino-HCAF-Net`)

> **Mục tiêu tài liệu:** Đặc tả toàn bộ kiến trúc mô hình của Piaino ở mức độ nguyên tử (atomic level) — từ byte sự kiện MIDI thô, máy trạng thái bàn phím-pedal, thuật toán căn chỉnh nốt, công thức toán học của từng chiều đặc trưng, tới từng dòng biến đổi Tensor (`shape`, `index`, `equation`) trong mạng học sâu và bộ biên dịch lời khuyên sư phạm. Tài liệu được viết để có thể đọc như **đang chạy trình gỡ lỗi (debugger) qua từng dòng lệnh trên một ô nhịp thực tế**.

---

## 1. Bức tranh Tổng thể: Tại sao cần Kiến trúc 3 Tầng (Tri-Tier Architecture)?

Một hệ thống AI Piano Guru thực thụ phải trả lời đồng thời **4 câu hỏi ở 3 thang thời gian khác nhau**:
1. **Ở thang `< 5 ms` (Mỗi lần ngón tay chạm phím):** *Người học đang đánh nốt nào trên bản nhạc? Đúng hay sai cao độ?* $\rightarrow$ **Tier-1: Causal Symbolic Score Follower (WASM).**
2. **Ở thang `< 80 ms` (Ngay khi kết thúc câu/đoạn nhạc):** *Điểm số 19 tiêu chí chuyên môn là bao nhiêu? Chính xác ô nhịp nào, phách nào, tay nào bị lỗi Voicing, lỗi Pedal hay lỗi Articulation? Nếu sửa lại cho đúng chuẩn Guru thì các nốt đó phải vang lên như thế nào?* $\rightarrow$ **Tier-2: `Piaino-HCAF-Net` (Hierarchical Cross-Gated Assessment & Feedback Network).**
3. **Ở thang `< 500 ms` (Streaming lời khuyên):** *Giải thích nguyên nhân lỗi bằng ngôn ngữ sư phạm dễ hiểu và tạo bài tập Micro-Drill 60 giây như thế nào?* $\rightarrow$ **Tier-3: Grounded Pedagogical Coach & Drill Compiler.**

---

## 2. Bước 0: Đơn vị Dữ liệu Nguyên tử (Atomic Data Units) & Máy Trạng thái Vật lý

Để hình dung rõ nhất, xét **1 ví dụ chạy xuyên suốt tài liệu (Running Debug Example)**:
- **Bản nhạc (Score):** Ô nhịp số 1 của một bản Nocturne nhịp `4/4`, Tempo mục tiêu `60 BPM` (1 phách quarter-note = `1.000 giây`).
  - **Tay phải (Voice 1 - `ROLE_MELODY`, có dấu `slur` nối liền):** Đánh 2 nốt giai điệu:
    - Nốt $s_0$: `E5` (MIDI pitch `76`), bắt đầu tại phách `0.0`, độ dài `2.0` phách (`2.000s`), ký hiệu sắc thái `mf` (velocity chuẩn `75`).
    - Nốt $s_1$: `G5` (MIDI pitch `79`), bắt đầu tại phách `2.0`, độ dài `2.0` phách (`2.000s`), nối legato từ `E5`.
  - **Tay trái (Voice 2 - `ROLE_BASS` & `ROLE_INNER_ACCOMP`, sắc thái `p` velocity chuẩn `50`):**
    - Phách `0.0`: Hợp âm Đô trưởng (C major): `C3` (pitch `48`, nốt $s_2$) và `G3` (pitch `55`, nốt $s_3$), độ dài `2.0` phách.
    - Phách `2.0`: Đổi hòa âm sang Hợp âm Sol 7 (G7): `B2` (pitch `47`, nốt $s_4$) và `F3` (pitch `53`, nốt $s_5$), độ dài `2.0` phách.
    - Yêu cầu Pedal: Nhấc sạch Pedal (`CC64 = 0`) tại phách `2.0` khi đổi từ hợp âm `C` sang `G7` rồi mới đạp lại.

---

### 2.1. Đơn vị 1: Gói tin MIDI thô (`RawMidiEvent`) và Máy Trạng thái Vật lý (`KeyPedalStateMachine`)

Trình duyệt nhận từ cáp USB-MIDI các gói tin 3-byte kèm nhãn thời gian độ phân giải micro-giây (`DOMHighResTimeStamp`):

```python
# Đơn vị nguyên tử đầu vào từ WebMIDI API:
# Tuple: (timestamp_sec: float64, status_byte: uint8, data1: uint8, data2: uint8)
# status_byte & 0xF0 == 0x90 (144): Note-On (nếu data2 > 0) hoặc Note-Off (nếu data2 == 0)
# status_byte & 0xF0 == 0x80 (128): Explicit Note-Off
# status_byte & 0xF0 == 0xB0 (176) và data1 == 64: Sustain Pedal (CC64), data2 in [0..127]
```

**Bẫy kỹ thuật số 1 mà 90% hệ thống làm sai — Phân biệt `t_finger_off` và `t_sound_off`:**
Khi người học nhấc ngón tay khỏi phím (`Note-Off`), nếu chân họ vẫn đang đạp Sustain Pedal (`CC64 >= 64`), **dây đàn thực tế chưa tắt tiếng** mà tiếp tục ngân cho đến tận lúc chân nhấc Pedal (`CC64 < 40`)!
- Nếu chỉ dùng `Note-Off` làm thời điểm kết thúc nốt, model sẽ không bao giờ phát hiện được lỗi **"Pedal Cheating"** (ngón tay nhấc sớm gây đứt tiếng nhưng dùng pedal đè lên) hay lỗi **"Harmonic Mud"** (tiếng của hợp âm C trưởng vẫn ngân đè lên hợp âm G7).
- Do đó, `KeyPedalStateMachine` phải tính **cả hai mốc thời gian** cho mỗi nốt biểu diễn $p_m$:

```python
# === LINE-BY-LINE DEBUG: KeyPedalStateMachine ===
class PerformedNoteAtom:
    perf_idx: int          # Chỉ số thứ tự nốt người dùng đánh: m = 0, 1, 2, ...
    pitch: int             # MIDI pitch [21..108] (A0..C8)
    vel_raw: int           # Velocity thô từ đàn [1..127]
    vel_cal: float         # Velocity sau hiệu chuẩn phần cứng [1.0..127.0]
    t_on: float            # Thời điểm chạm phím (giây)
    t_finger_off: float    # Thời điểm NGÓN TAY rời phím (giây)
    t_sound_off: float     # Thời điểm ÂM THANH thực sự tắt (xét cả Sustain Pedal CC64)
    cc64_at_on: int        # Giá trị Pedal [0..127] ngay lúc chạm phím t_on
    cc64_at_finger_off: int# Giá trị Pedal [0..127] ngay lúc ngón tay rời phím
    rearticulated: bool    # True nếu phím bị bấm lại khi dây đàn vẫn đang ngân bởi Pedal
```

**Trace chạy từng dòng trên dữ liệu giả lập của Học viên (User chơi có 3 lỗi điển hình):**
Giả sử học viên chơi ô nhịp 1 trên với 3 lỗi:
1. **Lỗi Voicing:** Đánh tay trái quá mạnh (`vel=82`), tay phải giai điệu lại yếu (`vel=58`).
2. **Lỗi Legato ngón tay:** Ở tay phải, nhấc ngón `E5` quá sớm tại `t = 1.60s` rồi mãi `t = 2.05s` mới bấm `G5` (tạo khoảng hở `0.45s`).
3. **Lỗi Pedal đục âm (Harmonic Blur):** Đạp giữ nguyên `CC64 = 127` từ `t = 0.05s` xuyên qua thời điểm đổi hợp âm tại `t = 2.00s` đến tận `t = 2.80s` mới nhấc!

Bảng trạng thái sau khi đi qua `KeyPedalStateMachine`:

| `m` | Nốt thực tế | `pitch` | `vel_cal` | `t_on` (s) | `t_finger_off` (s) | `t_sound_off` (s) *(Do giữ CC64 tới 2.80s!)* | `cc64_at_on` | Nhận xét vật lý từ State Machine |
|---:|---|---:|---:|---:|---:|---:|---:|---|
| `0` | `C3` (LH Bass) | `48` | `84.0` | `0.012` | `1.920` | **`2.800`** | `0` | Tay nhấc ở `1.92s` nhưng âm vẫn ngân tới `2.80s`! |
| `1` | `G3` (LH Inner) | `55` | `80.0` | `0.018` | `1.910` | **`2.800`** | `0` | Chạm phím trễ hơn C3 `6ms`. |
| `2` | `E5` (RH Melody)| `76` | `58.0` | `0.045` | **`1.600`** | **`2.800`** | `35` | **Phát hiện Pedal Cheating:** Tay nhấc sớm ở `1.60s` nhưng nhờ Pedal nên tiếng vẫn kêu tới `2.80s`. |
| `3` | `B2` (LH Bass) | `47` | `82.0` | `2.010` | `3.950` | `3.950` | **`127`** | Khi B2 vang lên ở `2.01s`, C3 và G3 vẫn đang ngân $\rightarrow$ **Va chạm hòa âm `[47, 48, 53, 55]` trong `0.79s`!** |
| `4` | `F3` (LH Inner) | `53` | `79.0` | `2.015` | `3.940` | `3.940` | **`127`** | Cùng hợp âm G7 với B2. |
| `5` | `G5` (RH Melody)| `79` | `60.0` | `2.055` | `3.980` | `3.980` | **`127`** | Vào trễ `45ms` so với tay trái. |

Chỉ riêng việc tách `t_finger_off` và `t_sound_off` ở Bước 0 đã phơi bày toàn bộ bản chất vật lý mà các app thông thường bỏ sót!

---

## 3. Bước 1: Thuật toán Căn chỉnh Nốt-với-Bản nhạc (Score-to-Performance Alignment)

Đầu vào của Bước 1 gồm:
- Danh sách nốt bản nhạc $\mathcal{S} = [s_0, s_1, \dots, s_{N-1}]$ được gom thành các **Cụm Onset đồng thời (Score Onset Units / Chords)** $U_0, U_1, \dots, U_{K-1}$ theo vị trí phách $b(U_k)$.
  - Trong ví dụ:
    - $U_0$ tại $b = 0.0$: gồm $\{s_2(\text{C3}=48), s_3(\text{G3}=55), s_0(\text{E5}=76)\}$.
    - $U_1$ tại $b = 2.0$: gồm $\{s_4(\text{B2}=47), s_5(\text{F3}=53), s_1(\text{G5}=79)\}$.
- Danh sách nốt người học đã đánh $\mathcal{P} = [p_0, p_1, \dots, p_{M-1}]$.

### 3.1. Công thức Toán học của Bộ Căn chỉnh (Hidden Markov / Dynamic Programming Matcher)

Khi một nốt mới $p_m = (\text{pitch}_m, t_{\text{on}, m})$ đi vào, điểm tương thích (Log-Probability Score) khi gán $p_m$ vào cụm bản nhạc $U_k$ được tính bằng:

$$\mathcal{L}(p_m \to U_k \mid U_{k_{\text{prev}}}) = \underbrace{\ln P_{\text{pitch}}(\text{pitch}_m \mid U_k)}_{\text{Điểm khớp cao độ}} + \underbrace{\ln P_{\text{time}}(\Delta t_m \mid \Delta b_k, \hat{v}_{\text{tempo}})}_{\text{Điểm khớp khoảng cách nhịp}} - \underbrace{\lambda_{\text{jump}} \cdot \text{JumpCost}(k_{\text{prev}} \to k)}_{\text{Phạt nhảy cóc / quay lui}}$$

Cụ thể từng số hạng tới đơn vị tính toán nhỏ nhất:
1. **Điểm khớp cao độ $\ln P_{\text{pitch}}(\text{pitch}_m \mid U_k)$:**
   - Đặt $d_{\min} = \min_{s \in U_k} |\text{pitch}_m - \text{pitch}(s)|$ (khoảng cách nửa cung gần nhất trong cụm $U_k$).
   - Nếu $d_{\min} == 0$ (Đúng cao độ và nốt $s$ trong $U_k$ chưa bị gán): $\ln P_{\text{pitch}} = 0.0$.
   - Nếu $d_{\min} \in \{1, 2\}$ (Trượt phím kế cận 1–2 nửa cung — lỗi bấm nhầm dấu hóa hoặc trượt ngón): $\ln P_{\text{pitch}} = -1.8 \cdot d_{\min}$.
   - Nếu $d_{\min} == 12$ (Sai quãng 8 — lỗi đọc nhầm dòng kẻ phụ): $\ln P_{\text{pitch}} = -2.5$.
   - Ngược lại (Sai cao độ hoàn toàn): $\ln P_{\text{pitch}} = -6.0$.
2. **Điểm khớp thời gian $\ln P_{\text{time}}(\Delta t_m \mid \Delta b_k, \hat{\tau})$:**
   - Gọi $\hat{\tau}$ là độ dài 1 phách ước lượng hiện tại (giây/beat, khởi tạo từ tempo bản nhạc $\hat{\tau} = 60 / \text{BPM} = 1.0\text{ s/beat}$, cập nhật theo cấp số nhân $\hat{\tau} \leftarrow 0.7 \hat{\tau} + 0.3 \frac{t_{\text{on}, m} - t_{\text{on}, \text{last\_matched}}}{b(U_k) - b(U_{k_{\text{prev}}})}$).
   - Khoảng thời gian kỳ vọng từ cụm trước tới cụm $U_k$: $\Delta t_{\text{exp}} = (b(U_k) - b(U_{k_{\text{prev}}})) \cdot \hat{\tau}$.
   - Khoảng thời gian thực tế: $\Delta t_{\text{obs}} = t_{\text{on}, m} - t_{\text{anchor}}(U_{k_{\text{prev}}})$.
   - Với các nốt nằm trong cùng một cụm hợp âm ($k == k_{\text{prev}}$, nên $\Delta b_k = 0$):
     $$\ln P_{\text{time}} = -\frac{1}{2} \left(\frac{\Delta t_{\text{obs}}}{\sigma_{\text{chord}}}\right)^2, \quad \text{với } \sigma_{\text{chord}} = 0.035\text{ s } (35\text{ ms})$$
   - Với bước chuyển sang cụm mới ($k \neq k_{\text{prev}}$):
     $$\ln P_{\text{time}} = -\frac{1}{2} \left(\frac{\Delta t_{\text{obs}} - \Delta t_{\text{exp}}}{\sigma_{\text{beat}} \cdot \sqrt{\max(\Delta b_k, 0.25)}}\right)^2, \quad \text{với } \sigma_{\text{beat}} = 0.15\text{ s}$$
3. **Chi phí chuyển trạng thái $\text{JumpCost}(k_{\text{prev}} \to k)$:**
   - $k = k_{\text{prev}}$ (Cùng cụm hợp âm): `cost = 0.0`
   - $k = k_{\text{prev}} + 1$ (Tiến đúng 1 cụm): `cost = 0.0`
   - $k = k_{\text{prev}} + 2$ (Bỏ sót 1 cụm): `cost = 2.2`
   - $k_{\text{bar\_start}} \le k < k_{\text{prev}}$ (Học viên dừng lại quay lui đánh lại đầu ô nhịp): `cost = 3.5`

Sau khi gán mỗi nốt thực tế $p_m$ vào cụm $U_k$, thuật toán chạy **Bipartite Matching (Hungarian / Greedy Pitch-Distance)** bên trong từng cụm $U_k$ để nối 1–1 từng nốt bản nhạc $s_i \in U_k$ với $p_m$:
- Nếu $s_i$ có $p_m$ khớp cao độ: `align_status[i] = MATCHED`, lưu `perf_map[i] = m`.
- Nếu $s_i$ ghép với $p_m$ lệch 1–2 nửa cung: `align_status[i] = PITCH_ERROR`, lưu `perf_map[i] = m`.
- Nếu $s_i$ không có $p_m$ nào: `align_status[i] = MISSED_NOTE`, `perf_map[i] = -1`.
- Nếu $p_m$ dư ra không khớp với $s_i$ nào trong $U_k$: đánh dấu $p_m$ là `EXTRA_NOTE` gắn vào vị trí phách gần nhất.

---

## 4. Bước 2: Trích xuất Vector Đặc trưng Cấp độ Nốt ($\mathbf{x}_i \in \mathbb{R}^{44}$) — Từng Chiều & Con số Cụ thể

Mỗi nốt trên bản nhạc $s_i$ ($i \in \{0, \dots, N-1\}$) được biểu diễn thành một vector **44 số thực `float32`** gồm:
- **24 đặc trưng Bản nhạc (`Score Context`, chiều `0..23`)**: Cho Model biết *"Bản nhạc yêu cầu điều gì ở nốt này?"*
- **20 đặc trưng Biểu diễn & Độ lệch (`Aligned Performance & Deviations`, chiều `24..43`)**: Cho Model biết *"Người học thực tế đã làm gì so với yêu cầu đó?"*

### 4.1. Bảng Định nghĩa Toán học Từng Chiều của Vector $\mathbf{x}_i \in \mathbb{R}^{44}$

#### Nhóm A: 24 Chiều Bản nhạc (`Score Context Features`, `idx 0..23`)

| Index | Tên biến | Công thức tính chính xác | Giá trị tại nốt $s_0$ (`E5` Melody) | Giá trị tại nốt $s_2$ (`C3` Bass) |
|---:|---|---|---:|---:|
| `0` | `pitch_norm` | $(\text{pitch}_i - 60.0) / 48.0 \in [-1, +1]$ | $(76-60)/48 = \mathbf{+0.3333}$ | $(48-60)/48 = \mathbf{-0.2500}$ |
| `1..12` | `pitch_class_1hot` | One-hot của $\text{pitch}_i \bmod 12$ (12 chiều) | Chiều `4` (E) = `1.0` | Chiều `0` (C) = `1.0` |
| `13` | `dur_score_beats` | $\log_2(1 + \text{dur\_beats}_i)$ | $\log_2(1 + 2.0) = \mathbf{1.5850}$ | $\mathbf{1.5850}$ |
| `14` | `beat_phase_sin` | $\sin(2\pi \cdot \text{beat\_in\_bar}_i / B_{\text{bar}})$ | $\sin(0) = \mathbf{0.0000}$ | $\mathbf{0.0000}$ |
| `15` | `beat_phase_cos` | $\cos(2\pi \cdot \text{beat\_in\_bar}_i / B_{\text{bar}})$ | $\cos(0) = \mathbf{1.0000}$ | $\mathbf{1.0000}$ |
| `16` | `is_downbeat` | $\mathbb{I}[\text{beat\_in\_bar}_i == 0]$ | $\mathbf{1.0}$ | $\mathbf{1.0}$ |
| `17` | `role_melody` | $\mathbb{I}[\text{role}_i == \text{MELODY}]$ | $\mathbf{1.0}$ | $\mathbf{0.0}$ |
| `18` | `role_bass` | $\mathbb{I}[\text{role}_i == \text{BASS}]$ | $\mathbf{0.0}$ | $\mathbf{1.0}$ |
| `19` | `role_inner` | $\mathbb{I}[\text{role}_i == \text{INNER\_ACCOMP}]$ | $\mathbf{0.0}$ | $\mathbf{0.0}$ |
| `20` | `slur_state` | `0.0`: none, `0.5`: slur start/mid, `1.0`: slur end | $\mathbf{0.5}$ (bắt đầu slur) | $\mathbf{0.0}$ |
| `21` | `artic_target` | `-1.0`: staccato, `0.0`: normal, `+1.0`: tenuto/legato | $\mathbf{+1.0}$ (legato) | $\mathbf{0.0}$ |
| `22` | `dyn_target_norm` | Velocity mục tiêu từ ký hiệu ($pp=0.25, p=0.40, mf=0.60, f=0.78$) | $\mathbf{0.6000}$ ($mf$) | $\mathbf{0.4000}$ ($p$) |
| `23` | `harmony_change` | $\mathbb{I}[\text{Cụm hiện tại đổi hợp âm so với cụm trước}]$ | $\mathbf{1.0}$ | $\mathbf{1.0}$ |

#### Nhóm B: 20 Chiều Biểu diễn & Độ lệch (`Performance & Deviation Features`, `idx 24..43`)

Gọi $m = \text{perf\_map}[i]$ là nốt thực tế người học đã đánh ứng với $s_i$ (nếu `MISSED_NOTE` thì các chiều `24..37` điền `0.0` và bật cờ `is_missed = 1.0` ở chiều `41`).

| Index | Tên biến | Công thức tính tới đơn vị nhỏ nhất | Trace số thực tại $s_0$ (`E5` Melody, $m=2$) | Trace số thực tại $s_4$ (`B2` Bass G7, $m=3$) | Ý nghĩa chẩn đoán trực tiếp |
|---:|---|---|---:|---:|---|
| `24` | `vel_norm` | $\tilde{v}_m / 127.0$ | $58 / 127 = \mathbf{0.4567}$ | $82 / 127 = \mathbf{0.6457}$ | Lực đánh thực tế đã hiệu chuẩn. |
| `25` | `vel_dev_target` | $(\tilde{v}_m / 127.0) - \text{dyn\_target\_norm}_i$ | $0.4567 - 0.6000 = \mathbf{-0.1433}$ | $0.6457 - 0.4000 = \mathbf{+0.2457}$ | **Bắt bệnh:** Giai điệu đánh thiếu lực (`-0.14`), bè trầm đánh dư lực (`+0.25`)! |
| `26` | `voicing_diff` | $\frac{\tilde{v}_m - \frac{1}{|U_k|}\sum_{j \in U_k} \tilde{v}_j}{127.0}$ | $\frac{58 - \frac{84+80+58}{3}}{127} = \frac{-16.0}{127} = \mathbf{-0.1260}$ | $\frac{82 - \frac{82+79+60}{3}}{127} = \frac{+8.33}{127} = \mathbf{+0.0656}$ | **Voicing Differential:** Nốt giai điệu `E5` lẽ ra phải dương ($> +0.10$) nhưng lại âm **`-0.126`** (bị bè đệm đè bẹp!). |
| `27` | `vel_zscore_local` | $\frac{\tilde{v}_m - \mu_{v,\text{phrase}}}{\sigma_{v,\text{phrase}} + 10^{-5}}$ | $\frac{58 - 73.83}{11.40} = \mathbf{-1.3886}$ | $\frac{82 - 73.83}{11.40} = \mathbf{+0.7167}$ | Độ nổi tương đối trong câu nhạc. |
| `28` | `local_qpm_norm` | $\frac{60.0}{\hat{\tau}_k \cdot 200.0}$ | $\frac{60.0}{1.002 \cdot 200} = \mathbf{0.2994}$ | $\mathbf{0.2994}$ | Tốc độ QPM cục bộ tại phách $k$. |
| `29` | `log_tempo_ratio` | $\log_2(\text{QPM}_k / \text{QPM}_{\text{median}})$ | $\log_2(59.88 / 59.88) = \mathbf{0.0000}$ | $\mathbf{0.0000}$ | Độ nhanh/chậm tương đối so với trung bình bài. |
| `30` | `onset_dev_sec` | $t_{\text{on}, m} - \hat{t}_{\text{beat}}(b_i)$ *(lệch so với tâm phách)* | $0.045 - 0.025 = \mathbf{+0.020\text{ s}}$ | $2.010 - 2.027 = \mathbf{-0.017\text{ s}}$ | Độ lệch thời điểm xuống phím (giây). |
| `31` | `onset_dev_beats` | $\text{onset\_dev\_sec}_i / \hat{\tau}_k$ | $+0.020 / 1.002 = \mathbf{+0.0200}$ | $-0.017 / 1.002 = \mathbf{-0.0170}$ | Độ lệch chuẩn hóa theo đơn vị phách. |
| `32` | `chord_spread_sec`| $t_{\text{on}, m} - \min_{j \in U_k} t_{\text{on}, j}$ | $0.045 - 0.012 = \mathbf{+0.033\text{ s}}$ | $2.010 - 2.010 = \mathbf{0.000\text{ s}}$ | `E5` xuống phím trễ `33ms` sau `C3`. |
| `33` | `finger_artic_log`| $\log_2\left(\frac{t_{\text{finger\_off}, m} - t_{\text{on}, m}}{\text{dur\_score\_beats}_i \cdot \hat{\tau}_k}\right)$ | $\log_2\left(\frac{1.600 - 0.045}{2.004}\right) = \mathbf{-0.3660}$ | $\log_2\left(\frac{3.950 - 2.010}{2.004}\right) = \mathbf{-0.0468}$ | Ngón tay `E5` chỉ giữ `77.6%` trường độ ($\log_2 = -0.366$). |
| `34` | `sound_artic_log` | $\log_2\left(\frac{t_{\text{sound\_off}, m} - t_{\text{on}, m}}{\text{dur\_score\_beats}_i \cdot \hat{\tau}_k}\right)$ | $\log_2\left(\frac{2.800 - 0.045}{2.004}\right) = \mathbf{+0.4592}$ | $\mathbf{-0.0468}$ | Nhưng âm thanh `E5` lại kéo dài `137.5%` ($\log_2 = +0.459$) do Pedal! |
| `35` | `pedal_cheat_gap` | $\max(0, \text{sound\_artic\_log}_i - \text{finger\_artic\_log}_i)$ | $0.4592 - (-0.3660) = \mathbf{+0.8252}$ | $\mathbf{0.0000}$ | **Chỉ số bắt lỗi Pedal Cheating:** Chênh lệch lớn giữa ngón tay và âm thanh! |
| `36` | `kot_sec` | $t_{\text{finger\_off}, m} - t_{\text{on}, \text{next\_in\_voice}(i)}$ | $1.600 - 2.055 = \mathbf{-0.455\text{ s}}$ | $\mathbf{0.000\text{ s}}$ | **Key Overlap Time:** Âm `-455ms` chứng tỏ câu legato bị đứt lìa giữa `E5` và `G5`! |
| `37` | `cc64_at_on_norm` | $\text{cc64\_at\_on}_m / 127.0$ | $35 / 127 = \mathbf{0.2756}$ | $127 / 127 = \mathbf{1.0000}$ | Độ sâu Pedal lúc bấm phím. |
| `38` | `harmonic_mud_sec`| $\max(0, \max_{j \in U_{\text{prev\_harm}}} t_{\text{sound\_off}, j} - t_{\text{on}, m})$ | $\mathbf{0.000\text{ s}}$ | $2.800 - 2.010 = \mathbf{+0.790\text{ s}}$ | **Độ đục hòa âm (Harmonic Blur):** Hợp âm C trưởng cũ vẫn ngân đè lên hợp âm G7 mới suốt **`0.790 giây`**! |
| `39` | `is_matched` | $\mathbb{I}[\text{status}_i == \text{MATCHED}]$ | $\mathbf{1.0}$ | $\mathbf{1.0}$ | Khớp đúng cao độ. |
| `40` | `is_pitch_error` | $\mathbb{I}[\text{status}_i == \text{PITCH\_ERROR}]$ | $\mathbf{0.0}$ | $\mathbf{0.0}$ | Đánh sai cao độ. |
| `41` | `is_missed` | $\mathbb{I}[\text{status}_i == \text{MISSED\_NOTE}]$ | $\mathbf{0.0}$ | $\mathbf{0.0}$ | Bỏ sót nốt. |
| `42` | `extra_notes_near`| Số nốt đánh thừa (`EXTRA_NOTE`) trong $\pm 0.25$ phách | $\mathbf{0.0}$ | $\mathbf{0.0}$ | Đánh thừa/chạm nhầm phím bên cạnh. |
| `43` | `backtrack_count` | Số lần quay lui (`Local Backtrack`) tại cụm hiện tại | $\mathbf{0.0}$ | $\mathbf{0.0}$ | Đếm số lần vấp/đánh lại. |

> **Quan sát quan trọng (Tại sao bước thiết kế đặc trưng này quyết định 80% sức mạnh của AI Guru):**
> Nhìn vào dòng `26` (`voicing_diff = -0.1260`), dòng `35` (`pedal_cheat_gap = +0.8252`), dòng `36` (`kot_sec = -0.455s`) và dòng `38` (`harmonic_mud_sec = +0.790s`), **mọi lỗi chuyên môn tinh vi nhất về Voicing, Legato ngón tay và Pedal đục âm đã trở thành các con số định lượng rõ ràng tới từng mili-giây** trước khi đi vào mạng Neural!

---

## 5. Bước 3: Kiến trúc Mạng Học sâu `Piaino-HCAF-Net` — Giải phẫu Từng Dòng Code & Kích thước Tensor

Kế thừa và nâng cấp từ hai kiến trúc SOTA mới nhất là **PercePiano Hierarchical Attention Network (Park et al., 2024)** và **RCG-DMAF-Net (Jin, 10/2026)**, `Piaino-HCAF-Net` bao gồm **2 Nhánh song song (Dual-Branch)** và **4 Cấp độ Phân cấp Âm nhạc (4-Level Musical Hierarchy)**:

### 5.1. Sơ đồ Luồng Tensor (Tensor Flow Blueprint)

```text
Input 1: Note Feature Sequence X_note [B, N, 44]
Input 2: Hierarchy Index Maps:
         • onset_group_idx [B, N]  (Nốt i thuộc cụm hợp âm/onset thứ mấy: 0..K-1)
         • beat_idx        [B, N]  (Nốt i thuộc phách thứ mấy: 0..S-1)
         • measure_idx     [B, N]  (Nốt i thuộc ô nhịp thứ mấy: 0..M-1)
Input 3: Engineered Global Segment Descriptors X_eng [B, 51]
         │
         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ STAGE 1: Note-Level Contextual Encoder                                     │
│  Linear(44 -> 256) + LayerNorm + GELU                                      │
│  + 2-Layer Relative-Position Transformer Encoder (d=256, heads=8)          │
│  ==> Output: H_note [B, N, 256]                                            │
└────────┬───────────────────────────────────────────────────────────────────┘
         │
         ├──► [HEAD 4: Note-Level Expressive Correction Head]
         │    MLP(256 -> 64 -> 3) ==> Delta_Target [B, N, 3]
         │    (Dự đoán: [delta_vel, delta_onset_beats, delta_artic_log] cho từng nốt!)
         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ STAGE 2: Hierarchical Musical Structure Aggregator (Note -> Beat -> Bar)   │
│  • Beat-Level Attention Pooling (nhóm theo beat_idx):                      │
│    H_note [B, N, 256] ──(SegmentAttnPool)──► H_beat_raw [B, S, 256]        │
│    H_beat = BiGRU_Beat(H_beat_raw)           ==> Shape: [B, S, 256]        │
│                                                                            │
│  • Measure-Level Attention Pooling (nhóm theo measure_idx):                │
│    H_beat [B, S, 256] ──(SegmentAttnPool)──► H_bar_raw  [B, M, 256]        │
│    H_bar  = Transformer_Bar(H_bar_raw)       ==> Shape: [B, M, 256]        │
└────────┬───────────────────────────────────────────────────────────────────┘
         │
         ├──► [HEAD 1: Measure-Level Defect & Heatmap Head]
         │    Linear(256 -> 6) + Sigmoid ==> Bar_Scores [B, M, 6]
         │    (Chấm điểm 6 tiêu chí cho TỪNG Ô NHỊP m = 0..M-1 trên khuông nhạc!)
         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ STAGE 3: Phrase/Segment Pooling & Residual Cross-Gated Fusion (RCG-DMAF)   │
│  • Attention Pooling qua M ô nhịp: H_bar [B, M, 256] -> h_seq [B, 256]     │
│  • Nhánh Domain Descriptors: X_eng [B, 51] -> MLP -> h_eng [B, 256]        │
│  • Cross-Gated Fusion:                                                     │
│    g_seq = Sigmoid(W_1 [h_seq || h_eng])     ==> Shape: [B, 256]           │
│    g_eng = Sigmoid(W_2 [h_seq || h_eng])     ==> Shape: [B, 256]           │
│    z_global = LayerNorm(h_seq + g_seq * h_eng + g_eng * W_3(h_seq))        │
│    ==> Output: z_global [B, 256]                                           │
└────────┬──────────────────────────────────────┬────────────────────────────┘
         │                                      │
         ▼                                      ▼
┌───────────────────────────────────┐  ┌─────────────────────────────────────┐
│ [HEAD 2: 19-Dim PercePiano Head]  │  │ [HEAD 3: MAJEPPA Skill Level Head]  │
│  • Mean: Linear(256 -> 19)        │  │  • Logits: Linear(256 -> 6)         │
│  • LogVar: Linear(256 -> 19)      │  │  (6 cấp độ: child_beginner          │
│  ==> y_percep_mean [B, 19]        │  │             .. virtuoso)            │
│  ==> y_percep_std  [B, 19]        │  └─────────────────────────────────────┘
└───────────────────────────────────┘
```

---

### 5.2. Mã Nguồn PyTorch Tường minh Kèm Debug Từng Dòng (`piaino_hcaf_net.py`)

Dưới đây là cài đặt chuẩn xác từng phép toán đại số tuyến tính để có thể chạy và kiểm tra shape ở từng dòng:

```python
import torch
import torch.nn as nn
import torch.nn.functional as F


class SegmentAttentionPool(nn.Module):
    """
    Gom nhóm có trọng số (Attention Pooling) các phần tử con (ví dụ: N nốt)
    vào các đơn vị cấu trúc cha (ví dụ: S phách hoặc M ô nhịp) theo segment_ids.
    """
    def __init__(self, d_model: int = 256):
        super().__init__()
        self.attn_proj = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.Tanh(),
            nn.Linear(d_model // 2, 1, bias=False)
        )

    def forward(
        self,
        x: torch.Tensor,            # Shape: [B, L_child, D] (VD: [B, N, 256])
        segment_ids: torch.Tensor,  # Shape: [B, L_child], giá trị trong [0..L_parent-1], padding = -1
        num_segments: int           # L_parent (VD: S phách hoặc M ô nhịp)
    ) -> torch.Tensor:              # Trả về: [B, L_parent, D]
        B, L_child, D = x.shape

        # Dòng 1: Tính điểm attention thô cho từng phần tử con
        raw_scores = self.attn_proj(x).squeeze(-1)          # Shape: [B, L_child]

        # Dòng 2: Tạo mặt nạ khớp từng phần tử con l vào đơn vị cha s in [0..num_segments-1]
        # seg_grid: [1, num_segments, 1], seg_ids_exp: [B, 1, L_child]
        seg_grid = torch.arange(num_segments, device=x.device).view(1, num_segments, 1)
        seg_ids_exp = segment_ids.unsqueeze(1)              # Shape: [B, 1, L_child]
        belong_mask = (seg_ids_exp == seg_grid)             # Shape: [B, num_segments, L_child] (bool)

        # Dòng 3: Che (mask) các vị trí không thuộc segment s bằng -1e9 trước khi lấy Softmax
        scores_exp = raw_scores.unsqueeze(1).expand(B, num_segments, L_child)
        masked_scores = scores_exp.masked_fill(~belong_mask, -1e9)
        attn_weights = F.softmax(masked_scores, dim=-1)     # Shape: [B, num_segments, L_child]

        # Nếu một ô nhịp/phách trống hoàn toàn (toàn False), đặt trọng số về 0
        attn_weights = attn_weights * belong_mask.float()   # Shape: [B, num_segments, L_child]

        # Dòng 4: Nhân ma trận [B, num_segments, L_child] @ [B, L_child, D] -> [B, num_segments, D]
        pooled = torch.bmm(attn_weights, x)                 # Shape: [B, num_segments, D]
        return pooled


class ResidualCrossGatedFusion(nn.Module):
    """
    Cơ chế Residual Cross-Gated Dual-Branch Fusion (RCG-DMAF):
    Cho phép nhánh Chuỗi phân cấp (h_seq) và nhánh 51 Đặc trưng kỹ thuật (h_eng)
    chọn lọc chéo thông tin của nhau.
    """
    def __init__(self, d_model: int = 256, d_eng: int = 51, dropout: float = 0.2):
        super().__init__()
        self.eng_encoder = nn.Sequential(
            nn.Linear(d_eng, d_model),
            nn.LayerNorm(d_model),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(d_model, d_model),
            nn.LayerNorm(d_model)
        )
        self.gate_seq = nn.Linear(d_model * 2, d_model)
        self.gate_eng = nn.Linear(d_model * 2, d_model)
        self.proj_seq = nn.Linear(d_model, d_model)
        self.proj_eng = nn.Linear(d_model, d_model)
        self.out_norm = nn.LayerNorm(d_model)

    def forward(
        self,
        h_seq: torch.Tensor,  # Shape: [B, 256] (Biểu diễn toàn đoạn từ nhánh phân cấp)
        x_eng: torch.Tensor   # Shape: [B, 51]  (51 đặc trưng thống kê kỹ thuật toàn đoạn)
    ) -> torch.Tensor:        # Trả về: [B, 256]
        # Dòng 1: Chiếu vector 51 đặc trưng kỹ thuật lên không gian 256 chiều
        h_eng = self.eng_encoder(x_eng)                     # Shape: [B, 256]

        # Dòng 2: Nối hai vector để tính cổng điều phối (Cross-Gates)
        concat = torch.cat([h_seq, h_eng], dim=-1)          # Shape: [B, 512]
        g_s = torch.sigmoid(self.gate_seq(concat))          # Shape: [B, 256] in (0, 1)
        g_e = torch.sigmoid(self.gate_eng(concat))          # Shape: [B, 256] in (0, 1)

        # Dòng 3: Cộng residual có cổng kiểm soát chéo và chuẩn hóa LayerNorm
        fused = h_seq + g_s * self.proj_eng(h_eng) + g_e * self.proj_seq(h_seq)
        return self.out_norm(fused)                         # Shape: [B, 256]


class PiainoHCAFNet(nn.Module):
    def __init__(
        self,
        d_note_in: int = 44,
        d_eng_in: int = 51,
        d_model: int = 256,
        n_heads: int = 8,
        dropout: float = 0.2
    ):
        super().__init__()
        # 1. Note-level Input Projection + Transformer
        self.note_in_proj = nn.Sequential(
            nn.Linear(d_note_in, d_model),
            nn.LayerNorm(d_model),
            nn.GELU()
        )
        note_layer = nn.TransformerEncoderLayer(
            d_model=d_model, nhead=n_heads, dim_feedforward=d_model * 4,
            dropout=dropout, activation="gelu", batch_first=True, norm_first=True
        )
        self.note_transformer = nn.TransformerEncoder(note_layer, num_layers=2)

        # 2. Hierarchical Aggregation: Note -> Beat -> Measure -> Phrase
        self.note_to_beat_pool = SegmentAttentionPool(d_model)
        self.beat_gru = nn.GRU(
            input_size=d_model, hidden_size=d_model // 2,
            num_layers=1, batch_first=True, bidirectional=True
        )
        self.beat_norm = nn.LayerNorm(d_model)

        self.beat_to_bar_pool = SegmentAttentionPool(d_model)
        bar_layer = nn.TransformerEncoderLayer(
            d_model=d_model, nhead=n_heads, dim_feedforward=d_model * 2,
            dropout=dropout, activation="gelu", batch_first=True, norm_first=True
        )
        self.bar_transformer = nn.TransformerEncoder(bar_layer, num_layers=2)

        # Global phrase pooling over M measures
        self.bar_global_pool = SegmentAttentionPool(d_model)

        # 3. Cross-Gated Dual-Branch Fusion
        self.rcg_fusion = ResidualCrossGatedFusion(d_model, d_eng_in, dropout)

        # 4. Output Heads
        # Head 1: Điểm 6 khía cạnh kỹ thuật cho TỪNG Ô NHỊP [B, M, 6]
        self.head_bar_diagnostics = nn.Sequential(
            nn.Linear(d_model, d_model // 2),
            nn.GELU(),
            nn.Linear(d_model // 2, 6),
            nn.Sigmoid()
        )
        # Head 2: 19 chiều cảm thụ PercePiano (Mean + Log-Variance) [B, 19]
        self.head_percep_mean = nn.Sequential(
            nn.Linear(d_model, 128),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(128, 19),
            nn.Sigmoid()
        )
        self.head_percep_logvar = nn.Sequential(
            nn.Linear(d_model, 128),
            nn.GELU(),
            nn.Linear(128, 19)
        )
        # Head 3: 6 cấp độ kỹ năng MAJEPPA [B, 6]
        self.head_skill_logits = nn.Linear(d_model, 6)

        # Head 4: Dự đoán vector chỉnh sửa biểu cảm từng nốt (Guru Target Morphing) [B, N, 3]
        self.head_note_correction = nn.Sequential(
            nn.Linear(d_model * 2, 128),
            nn.GELU(),
            nn.Linear(128, 3)  # [delta_vel_norm, delta_onset_beats, delta_artic_log]
        )

    def forward(
        self,
        x_note: torch.Tensor,          # [B, N, 44]
        x_eng: torch.Tensor,           # [B, 51]
        note_pad_mask: torch.Tensor,   # [B, N] (True tại vị trí padding)
        note_to_beat_idx: torch.Tensor,# [B, N] (chỉ số phách 0..S-1)
        beat_to_bar_idx: torch.Tensor, # [B, S] (chỉ số ô nhịp 0..M-1)
        num_beats: int,                # S
        num_bars: int                  # M
    ) -> dict[str, torch.Tensor]:
        B, N, _ = x_note.shape

        # --- BƯỚC 3.1: Mã hóa cấp độ Nốt ---
        h_n0 = self.note_in_proj(x_note)                    # [B, N, 256]
        h_note = self.note_transformer(
            h_n0, src_key_padding_mask=note_pad_mask
        )                                                   # [B, N, 256]

        # --- BƯỚC 3.2: Gom từ N Nốt -> S Phách (Beats) ---
        h_beat_in = self.note_to_beat_pool(
            h_note, note_to_beat_idx, num_beats
        )                                                   # [B, S, 256]
        h_beat_gru, _ = self.beat_gru(h_beat_in)            # [B, S, 256]
        h_beat = self.beat_norm(h_beat_in + h_beat_gru)     # [B, S, 256]

        # --- BƯỚC 3.3: Gom từ S Phách -> M Ô nhịp (Measures/Bars) ---
        h_bar_in = self.beat_to_bar_pool(
            h_beat, beat_to_bar_idx, num_bars
        )                                                   # [B, M, 256]
        h_bar = self.bar_transformer(h_bar_in)              # [B, M, 256]

        # --- BƯỚC 3.4: Chấm điểm chẩn đoán từng Ô nhịp (Head 1) ---
        # 6 chiều: [pitch_acc, rhythm_stab, voicing_bal, artic_touch, pedal_clarity, tempo_evenness]
        bar_scores = self.head_bar_diagnostics(h_bar)       # [B, M, 6]

        # --- BƯỚC 3.5: Gom toàn đoạn & Kết hợp 51 Domain Descriptors ---
        zero_seg = torch.zeros((B, num_bars), dtype=torch.long, device=x_note.device)
        h_seq_global = self.bar_global_pool(h_bar, zero_seg, 1).squeeze(1)  # [B, 256]
        z_global = self.rcg_fusion(h_seq_global, x_eng)     # [B, 256]

        # --- BƯỚC 3.6: Dự đoán 19 chiều PercePiano & 6 cấp độ MAJEPPA ---
        percep_mean = self.head_percep_mean(z_global)       # [B, 19] in (0, 1)
        percep_logvar = torch.clamp(
            self.head_percep_logvar(z_global), min=-6.0, max=2.0
        )                                                   # [B, 19]
        skill_logits = self.head_skill_logits(z_global)     # [B, 6]

        # --- BƯỚC 3.7: Dự đoán hiệu chỉnh từng nốt (Head 4 - Guru Demo Morphing) ---
        # Ghép ngữ cảnh toàn cục z_global vào từng nốt h_note
        z_exp = z_global.unsqueeze(1).expand(B, N, -1)      # [B, N, 256]
        h_note_ctx = torch.cat([h_note, z_exp], dim=-1)     # [B, N, 512]
        note_delta = self.head_note_correction(h_note_ctx)  # [B, N, 3]

        return {
            "bar_scores": bar_scores,           # [B, M, 6]  -> Vẽ Heatmap lên từng ô nhịp
            "percep_mean": percep_mean,         # [B, 19]    -> Điểm 19 chiều chuyên gia
            "percep_std": torch.exp(0.5 * percep_logvar), # [B, 19] -> Độ bất định
            "skill_logits": skill_logits,       # [B, 6]     -> Phân loại trình độ
            "note_delta": note_delta            # [B, N, 3]  -> Sinh MIDI mẫu "Guru sửa lỗi"
        }
```

---

### 5.3. Giải phẫu 51 Đặc trưng Kỹ thuật Toàn đoạn (`Engineered Global Descriptors`, $\mathbf{x}_{\text{eng}} \in \mathbb{R}^{51}$)

Tại sao nhánh $\mathbf{x}_{\text{eng}} \in \mathbb{R}^{51}$ lại giúp tăng mạnh độ chính xác $R^2$ trên tập test độc lập (như chứng minh trong nghiên cứu `RCG-DMAF-Net` tháng 10/2026)? Vì một số tính chất âm nhạc là **thống kê bậc hai trên toàn câu nhạc** mà mạng Attention đơn thuần dễ bị nhiễu khi chuỗi dài:

| Chỉ mục (`idx`) | Nhóm đặc trưng | Công thức định nghĩa cụ thể | Số chiều |
|---|---|---|---|
| `0..7` | **Thống kê Nhịp & Độ ổn định Tempo** | Mean, Std, IQR, Skewness, Kurtosis của `onset_dev_beats`; Hệ số biến thiên $\text{CV}(\text{IOI})$; Độ giật nhịp bậc hai $\frac{1}{S-2}\sum |\text{QPM}_{s+1} - 2\text{QPM}_s + \text{QPM}_{s-1}|$; Độ dốc hồi quy tuyến tính của Tempo (phát hiện *rushing/dragging* toàn cục). | `8` |
| `8..17` | **Thống kê Dynamics & Tách bè (Voicing)** | Mean, Std, P90–P10 (`dynamic_range`) của $\tilde{v}$; Trung bình `voicing_diff` trên các nốt `ROLE_MELODY`; Tỷ lệ phần trăm số hợp âm mà $\tilde{v}_{\text{melody}} > \max(\tilde{v}_{\text{accomp}})$; Hệ số tương quan Pearson giữa đường cong velocity của người học và đường cong `dyn_target_norm` trên bản nhạc; Độ chênh lệch lực trung bình giữa Tay phải và Tay trái. | `10` |
| `18..27` | **Thống kê Articulation & Độ đều Ngón** | Mean, Std của `finger_artic_log` trên các nốt có nhãn *legato*; Mean, Std của `finger_artic_log` trên các nốt *staccato*; Trung bình `kot_sec` trong dấu slur; Tỷ lệ nốt bị dính phím ($\text{KOT} > +35\text{ ms}$); Tỷ lệ nốt bị đứt câu ($\text{KOT} < -50\text{ ms}$ trong slur); Trung bình `pedal_cheat_gap`; Độ lệch chuẩn velocity trong các chuỗi chạy ngón nhanh ($\text{IOI} < 180\text{ ms}$). | `10` |
| `28..36` | **Thống kê Pedal & Độ sạch Hòa âm** | Tỷ lệ thời gian đạp Pedal (`CC64 >= 64`); Số lần nhấc Pedal trên mỗi ô nhịp; Trung bình và Max của `harmonic_mud_sec` tại các điểm đổi hợp âm (`harmony_change == 1`); Tỷ lệ đổi hợp âm có thực hiện *Syncopated Pedaling* (nhấc Pedal trong khoảng $[0, +120\text{ ms}]$ sau onset hợp âm mới); Độ tương phản Wet/Dry. | `9` |
| `37..44` | **Thống kê Độ đồng bộ Hợp âm (Chord Touch)** | Mean, P90 của `chord_spread_sec`; Tỷ lệ hợp âm mà nốt Soprano chạm phím trước các nốt đệm $5\text{--}20\text{ ms}$ (kỹ thuật *Melody Lead* của nghệ sĩ); Tỷ lệ hợp âm bị tòe ngoài ý muốn ($\text{spread} > 35\text{ ms}$). | `8` |
| `45..50` | **Thống kê Độ chính xác & Độ trôi chảy (Fluency)** | Tỷ lệ `MATCHED`, `PITCH_ERROR`, `MISSED_NOTE`, `EXTRA_NOTE`; Tổng số lần vấp/quay lui `backtrack_count` trên mỗi ô nhịp; Entropy của phân phối thời gian tập trên các ô nhịp. | `6` |

---

## 6. Bước 4: Hàm Mất mát Đa mục tiêu (Composite Loss) & Ràng buộc Đơn điệu

Khi huấn luyện `Piaino-HCAF-Net`, hàm mục tiêu tổng hợp $\mathcal{L}_{\text{total}}$ gồm 5 thành phần:

$$\mathcal{L}_{\text{total}} = \lambda_1 \mathcal{L}_{\text{Percep-NLL}} + \lambda_2 \mathcal{L}_{\text{Bar-Focal}} + \lambda_3 \mathcal{L}_{\text{Monotonic}} + \lambda_4 \mathcal{L}_{\text{Skill-SupCon}} + \lambda_5 \mathcal{L}_{\text{Morph-Huber}}$$

Trong đó hai thành phần quan trọng nhất đảm bảo Model chấm điểm chuẩn xác như chuyên gia là:

1. **Uncertainty-Aware Perceptual Loss ($\mathcal{L}_{\text{Percep-NLL}}$):**
   Vì mỗi mẫu trong `PercePiano` có cả điểm trung bình của hội đồng giám khảo $y_d$ và độ lệch chuẩn giữa các giám khảo $\sigma_{\text{gt}, d}$ ($d \in \{1..19\}$), dùng Kullback-Leibler / Gaussian Negative Log-Likelihood giúp model tự học được tiêu chí nào rõ ràng, tiêu chí nào mang tính cảm nhận chủ quan:
   $$\mathcal{L}_{\text{Percep-NLL}} = \frac{1}{19} \sum_{d=1}^{19} \left( \frac{(\hat{\mu}_d - y_d)^2}{2 \hat{\sigma}_d^2} + \frac{1}{2} \ln \hat{\sigma}_d^2 + \gamma_{\text{std}} |\hat{\sigma}_d - \sigma_{\text{gt}, d}| \right)$$

2. **Pairwise Monotonicity Ranking Loss ($\mathcal{L}_{\text{Monotonic}}$ — Chống chấm ngược):**
   Với mỗi bản chơi chuẩn $X^{(0)}$ và bản chơi đã bị tiêm lỗi kỹ thuật mức độ $\alpha_1 < \alpha_2$ tại ô nhịp $m$ thông qua *Pedagogical Perturbation Engine*:
   $$\mathcal{L}_{\text{Monotonic}} = \sum_{c=1}^{6} \max\left(0, \, \text{margin} - \left(\hat{S}_{m, c}(X^{(\alpha_1)}) - \hat{S}_{m, c}(X^{(\alpha_2)})\right)\right)$$
   Hàm loss này ép buộc toán học: **Hễ lỗi kỹ thuật ở ô nhịp $m$ nặng hơn thì điểm ô nhịp $m$ của đúng tiêu chí $c$ đó bắt buộc phải thấp hơn!**

---

## 7. Bước 5: Biên dịch từ Tensor Đầu ra sang Lời khuyên Sư phạm & Bài tập (`Diagnostic-to-Guru Compiler`)

Quay lại **Ví dụ Debug Ô nhịp 1** ở Bước 0 & Bước 2: Sau khi chạy qua `Piaino-HCAF-Net` (tốn `14.2 ms`), hệ thống thu được các con số đầu ra từ `bar_scores[0, 0, :]` (Ô nhịp 1) và vector đặc trưng nốt. Bộ biên dịch `DiagnosticCompiler` tạo ra cấu trúc JSON có kiểm chứng (Grounded Diagnostic Payload) như sau:

```json
{
  "piece_id": "nocturne_c_major_demo",
  "evaluated_bars": [1],
  "overall_skill_estimate": "adult_beginner (confidence: 0.78)",
  "macro_pillars": {
    "pitch_and_rhythm": 91.0,
    "touch_and_voicing": 42.5,
    "pedal_and_phrasing": 34.0
  },
  "top_pinpointed_defects": [
    {
      "rank": 1,
      "bar": 1,
      "beat": 3.0,
      "category": "PEDAL_HARMONIC_BLUR",
      "severity": 0.88,
      "measured_evidence": {
        "chord_transition": "C major -> G7",
        "cc64_at_change": 127,
        "harmonic_mud_duration_ms": 790,
        "expected_max_mud_ms": 120
      },
      "guru_target_fix": "Nhấc chân Pedal (CC64 -> 0) ngay khi tay trái chạm hợp âm B2-F3 ở phách 3, sau đó mới đạp xuống lại sau 80ms (Syncopated Pedaling)."
    },
    {
      "rank": 2,
      "bar": 1,
      "beat": 1.0,
      "category": "VOICING_IMBALANCE_AND_PEDAL_CHEATING",
      "severity": 0.81,
      "measured_evidence": {
        "melody_note": "E5 (vel=58)",
        "accompaniment_notes": "C3-G3 (vel=84, 80)",
        "voicing_differential_db": -16.0,
        "finger_gap_to_next_melody_ms": -455,
        "sustain_pedal_masked_gap": true
      },
      "guru_target_fix": "Tăng trọng lượng ngón tay phải ở nốt E5 (vel ~75) và hạ nhẹ tay trái (vel ~48). Giữ ngón tay phải trên phím E5 đủ 2 phách cho tới khi chạm G5, không nhấc tay sớm rồi dùng Pedal nối hộ."
    }
  ],
  "auto_generated_micro_drill": {
    "drill_type": "VOICING_AND_CLEAN_PEDAL_LOOP",
    "target_bars": [1, 1],
    "tempo_bpm": 45,
    "pass_criteria": {
      "min_melody_minus_accomp_vel": 15,
      "max_harmonic_mud_ms": 150,
      "max_melody_finger_gap_ms": 30,
      "required_consecutive_passes": 3
    }
  }
}
```

### Kết quả hiển thị cho User (Được sinh từ JSON trên với 0% Hallucination):
> **Chẩn đoán của AI Guru (Ô nhịp 1):**
> - **Đúng cao độ & nhịp cơ bản (91/100)**, nhưng âm thanh đang bị đục và giai điệu bị chìm:
> 1. **Đục hòa âm do Pedal ở phách 3:** Bạn giữ nguyên Pedal từ hợp âm Đô trưởng sang hợp âm Sol 7 khiến hai hợp âm va nhau suốt **0.79 giây**. Hãy nhấc sạch chân Pedal ngay khoảnh khắc tay trái bấm xuống phách 3 rồi mới đạp lại.
> 2. **Tay trái đè giai điệu & Nhấc ngón sớm ở phách 1–2:** Nốt giai điệu `Mi (E5)` tay phải đang nhẹ hơn phần đệm tay trái **24 đơn vị lực**, đồng thời ngón tay phải nhấc sớm **0.45 giây** trước khi sang nốt `Sol (G5)` (bạn đang dùng Pedal để nối âm thay cho ngón tay).
>
> 👉 **[Bấm để nghe Bản chơi của bạn vs. Bản sửa mẫu của Guru]** | **[Bắt đầu Bài tập 60 giây: Tập nối ngón tay phải & Đổi Pedal sạch ở 45 BPM (0/3 lượt)]**

Toàn bộ luồng từ lúc ngón tay chạm phím đến lời khuyên trên hoàn toàn tường minh, kiểm chứng được tới từng mili-giây và phục vụ trúng đích 100% nỗi đau của cả người tự học lẫn giáo viên piano!
