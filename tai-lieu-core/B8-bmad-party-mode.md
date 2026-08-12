# B8 — `bmad-party-mode`

> [← Chỉ mục](./index.md) · Trước: [B7](./B7-bmad-forge-idea.md) · Tiếp: [B9 — v6-shims](./B9-v6-shims.md)

---

## 1. Nhận diện nhanh

| Thuộc tính | Giá trị |
| --- | --- |
| Tên | `bmad-party-mode` |
| Mã menu | `PM` |
| Pha | `anytime` |
| Bắt buộc | Không |
| File | `SKILL.md`, `customize.toml` (**211 dòng — chứa 9 persona + 2 party dựng sẵn**), 5 `references/`, `scripts/resolve_party.py` (282 dòng) |
| Tạo phẩm | Keepsake HTML (tùy chọn), memlog **theo từng party** |
| Vị trí | `src/core-skills/bmad-party-mode/` |

**Lập trường:**

> *Run a round-table where these agents **talk to each other and to the user like real, distinct people in conversation**. You're the **orchestrator**.*

---

## 2. Bốn nhóm quy ước

### 2.1 Đường dẫn

| Ký hiệu | Phân giải |
| --- | --- |
| Đường dẫn trần (`references/create-party.md`) | Từ `{skill-root}` |
| `{project-root}`-prefixed | Từ thư mục làm việc dự án |
| `{workflow.<tên>}` | Bảng `[workflow]` của `customize.toml` (**override thắng**) |

### 2.2 Script — bốn cái, hai vị trí

| Script | Vị trí | Vai trò |
| --- | --- | --- |
| `resolve_config.py` | `{project-root}/_bmad/scripts/` | Cấu hình trung tâm (4 lớp) |
| `resolve_customization.py` | `{project-root}/_bmad/scripts/` | `{workflow.*}` |
| `memlog.py` | `{project-root}/_bmad/scripts/` | Bộ nhớ theo party |
| **`resolve_party.py`** | **`{skill-root}/scripts/`** | Roster, `party_mode`, `memory_enabled`, scene/`open_cast` |

### 2.3 Vai trò file

| Thứ | Ở đâu |
| --- | --- |
| Bộ nhớ của một party | `{workflow.memory_dir}/<party>/.memlog.md` |
| Thành viên và nhóm tùy biến | Override `customize.toml` của người dùng |
| Cơ chế bộ nhớ | `references/party-memory.md` |
| Cơ chế tạo party | `references/create-party.md` |

### 2.4 Tìm kiếm

> **Web-search, don't guess** — bất cứ gì sau cutoff hoặc không quen. **Subagent cũng vậy.**

---

## 3. Kích hoạt — 7 bước

```mermaid
graph TB
  A["1. resolve_customization.py --key workflow<br/>Chay activation_steps_prepend<br/>Giu persistent_facts (file: / skill: / nguyen van)"] --> B["2. resolve_config.py<br/>-> user_name, communication_language, output_folder, {date}"]
  B --> C{"3. PHAT HIEN Y DINH"}
  C -->|"Muon TAO/CAU HINH party<br/>(bia dan dien vien, them persona,<br/>chung cat du lieu khach hang thanh panel,<br/>dat mac dinh, sua party cu)"| CP["Nap references/create-party.md va theo no"]
  C -->|Chay mot party| D["4. resolve_party.py --project-root --skill"]
  D --> D1["Tra ve: roster dang hoat dong, ten cac group khac,<br/>party_mode, memory_enabled, scene/open_cast"]
  D1 --> E{"5. memory_enabled?"}
  E -->|Co| EM["Theo references/party-memory.md CA PHIEN"]
  E -->|Khong| F
  EM --> F["6. CHAO: hien ai dang trong phong (icon, ten, vai tro mot dong)<br/>Ghi chu co the chuyen sang group khac<br/>Hoi ho muon dao vao gi (tru khi da ro tu cach goi skill)"]
  F --> G["7. Chay activation_steps_append<br/>Xac nhan MOI muc da chay neu hook list khong rong"]
```

### 3.1 Xử lý kết quả `resolve_party.py`

| Trả về | Áp dụng thế nào |
| --- | --- |
| `scene` | Áp dụng ngay, **để nó định hình cách phòng hành xử** |
| `open_cast` | **Cast tại chỗ** — ai hợp khoảnh khắc, **thay đổi khi chủ đề chuyển** |
| `installed_agents_resolved = false` hoặc code trả về `unresolved` | **Nói cho người dùng biết**, tiếp tục với cái đã trả về, và **ứng biến** |

### 3.2 Ba cách override roster

| Cách | Hiệu lực |
| --- | --- |
| **Cast nêu tên inline** | Cast đó **LÀ** roster của phiên — triệu hồi họ, vào thẳng |
| `--party <id>` (alias `--group <id>`) | Ghi đè `default_party` đã cấu hình. **ID không biết ⇒ hiện tên khả dụng và hỏi** |
| `--list-groups` | Chỉ hiện menu |

**Giữa phiên** cùng các đòn bẩy đó: đổi phòng bằng cách chạy lại `resolve_party.py --party <id>` và **mang mạch chuyện sang**, hoặc **triệu hồi bất kỳ thành viên nào của tập thể bằng tên**.

---

## 4. Chín tiêu chí "giữ cho nó là một bữa tiệc"

Đây là **thanh chuẩn** — cố gắng đạt **từng cái, mỗi vòng**. Đó là khác biệt giữa một **bữa tiệc** và một **hội thảo**.

| # | Tiêu chí | Nội dung |
| --- | --- | --- |
| 1 | **Đọc như người nói chuyện, không như báo cáo** | Lượt ngắn, phản ứng thật, đùa cợt, đà. Một **group chat**, không phải chồng memo. **Ngắn gọn theo mặc định** — persona chỉ dài khi được yêu cầu. **Khoảnh khắc nó đọc như câu trả lời được nộp hồ sơ là bữa tiệc đã chết** |
| 2 | **Mỗi giọng không thể nhầm lẫn** | Từ vựng, hài hước, thứ ghét, đạo lý, năng lực. **Giấu nhãn đi bạn vẫn biết ai đang nói.** Giọng **không đều và lập dị**: có người áp đảo, có người cứ kéo về chủ đề ruột. **Đổi ai nổi bật giữa các vòng. Một panel cân bằng thì chán** |
| 3 | **Họ va chạm, và bạn KHÔNG giải quyết** | Thách thức, đẩy lùi mạnh, nóng lên khi đáng. Liên minh và phe phái hình thành. **Bản năng của bạn là hòa giải và thắt nơ — CHỐNG LẠI NÓ.** Đồng thuận sạch sẽ không tốn công là nơi bữa tiệc chết |
| 4 | **Một trao đổi, dệt lại — không bao giờ làm mềm** | Trình bày **một** hội thoại — lượt dạng `{icon} **{name}:**`, nối tiếp — **không phải một dãy câu trả lời**. Thêm dàn cảnh và mô liên kết, nhưng **không bao giờ đổi cái persona đã lập luận**, và **không bao giờ diễn giải lời họ ở ngôi thứ ba** — để họ tự nói. **Dệt phần trình bày, giữ phần thực chất** |
| 5 | **Kéo người dùng vào phòng** | Nhân vật nói **với** họ (và với nhau) — thách thức, trêu, đặt câu hỏi ngược. Họ là **khách bị kéo vào cuộc tranh cãi**, không phải người điều hành panel từ bên ngoài |
| 6 | **Làm cho va chạm xứng đáng** | Đẩy các giọng cho tới khi va chạm của họ **làm lộ một góc mà không giọng nào (kể cả bạn) tự đến được**. Đó là toàn bộ lý do có nhiều hơn một trí óc trong phòng |
| 7 | **Để lịch sử hình thành** | Hiềm khích, liên minh, một trò đùa lặp lại, một callback về ba lượt trước. Để quan hệ tích tụ, để những người này **cảm giác đang trở thành cái gì đó qua phiên**, không reset mỗi lượt |
| 8 | **Cam kết với hư cấu** | Scene và mỗi persona là **ràng buộc** — diễn dàn cảnh, nhân vật, và thế giới quanh bàn **đúng như đã viết**, mang cả hai vào bất kỳ brief nào được sinh ra. **Không bao giờ phá bức tường thứ tư về cơ chế** (không nói "bạn có 4 agent trong phòng"). Nghiêng vào thế giới khi nó nâng khoảnh khắc; đứng ngoài khi scene chỉ là một căn phòng |
| 9 | **Khi nó chùng, ĐỔI cái gì đó — đừng ép** | Lượt nhạt? **Đi tiếp, đừng thử lại.** Trôi vào hỏi-đáp hoặc lòng vòng? Đưa giọng mới vào, pha trò, gọi tên bế tắc, hoặc hỏi họ muốn đưa nó đi đâu. **Không bao giờ chèn tóm tắt hay takeaway** — chúng có sẵn nếu người dùng hỏi |

---

## 5. Bốn chế độ chạy

```mermaid
graph TB
  M{"{workflow.party_mode}<br/>--mode override LUON THANG"}
  M -->|session| S["SESSION - san nha<br/>Long tieng MOI persona noi tuyen<br/>MOT tri oc dang sau moi giong<br/>Khong can chi dan them"]
  M -->|auto| A["AUTO<br/>Long tieng noi tuyen cho qua lai thong thuong<br/>SPAWN agent that CHI khi tu duy doc lap doi ket qua"]
  M -->|subagent| SA["SUBAGENT<br/>Mot agent that sau MOI persona<br/>MOI vong thuc chat"]
  M -->|agent-team| AT["AGENT-TEAM<br/>Dung persona thanh doi BEN VUNG<br/>ho noi TRUC TIEP voi nhau<br/>CHI Claude Code"]

  A -.-> AR["Nap references/mode-auto.md de quyet dinh<br/>Khi no bao spawn -> theo mode-subagent.md"]
  SA -.-> SAR["Nap references/mode-subagent.md<br/>Uu tien model nhanh re hon cho tung subagent"]
  AT -.-> ATR["Nap references/mode-agent-team.md"]

  A -.khong co co che.-> S
  SA -.khong co co che.-> S
  AT -.khong dung duoc team.-> SA
  AT -.van khong duoc.-> S
```

> **Một chế độ hoạt động một lúc; nếu cơ chế của nó không có trong harness, rơi về `session` KHÔNG BÌNH LUẬN GÌ.**

### 5.1 `auto` — khi nào spawn, khi nào lồng tiếng

Spawn agent độc lập khi **tư duy phân kỳ, không bị nhuốm màu, chính là giá trị của vòng đó**:

| Spawn khi | Ví dụ |
| --- | --- |
| Một đánh giá/review/phê bình **thật** — loại sẽ hỏng nếu một trí óc lồng tiếng mọi phía và họ trôi vào đồng ý | Code review, red-team, nhìn kỹ một kế hoạch |
| Các persona **có khả năng đi tới kết luận KHÁC NHAU**, và sự phân kỳ đó **là điểm mấu chốt** | |
| Người dùng **yêu cầu ai đó đào sâu, phân tích, hoặc nghiên cứu** | Chiều sâu do một yêu cầu trực tiếp |

Lồng tiếng nội tuyến cho **mọi thứ khác**: đùa cợt, phản ứng, ý kiến nhanh, phần qua lại liên kết — vốn là **phần lớn** một cuộc hội thoại.

> **Khi phân vân, lồng tiếng** — spawn là ngoại lệ bạn với tới, không phải mặc định.

### 5.2 `subagent` — bốn cơ chế

#### a) Vòng đời — cast **thường trực**

Nơi harness giữ agent sống qua các lượt: spawn **một agent mỗi persona** và **tái dùng chính handle đó** vòng này qua vòng khác — đưa nó lượt mới cộng ngữ cảnh phòng nó cần — thay vì tạo mới mỗi lần.

> **Tính liên tục đó là thứ cho phép hiềm khích, liên minh, và callback của một persona TÍCH TỤ.**

Giữ **roster nhìn thấy được** ánh xạ mỗi persona sang handle sống của nó, và tái dùng nó.

| Quy tắc | Nội dung |
| --- | --- |
| Giữ cast sống **cả phiên** | Thành viên xong việc được giao là **rảnh, không phải xong** |
| **Đừng** đóng/nghỉ hưu/giải tán | Phục vụ ý định mở đầu **không** kết thúc bữa tiệc |
| Chỉ giải phóng agent ở **wrap-up** | |
| Nếu một cái bị đóng nhầm | **Resume nó**; không resume được thì **nói ra và chỉ respawn cái đó** |
| Harness không giữ được agent | Spawn mới mỗi vòng và tái lập brief + mạch chuyện — đó là **fallback, không phải mục tiêu** |

#### b) Một phòng dùng chung

> *It's **one room**, not parallel one-on-ones.*

**Mọi** thành viên thường trực **nghe mọi thứ** nói ra mỗi vòng — lượt của người dùng **và mọi lượt của persona khác** — **kể cả khi không phải lượt họ nói**.

Một persona ngồi ngoài một vòng **vẫn ở trong phòng, đang nghe**, nên khi nó nói lần sau nó **đã bắt kịp**: nó có thể nhặt lại một mạch bị bỏ rơi, giữ hiềm khích, gọi lại chuyện cũ.

> Route **toàn bộ** trao đổi tới **tất cả** họ mỗi vòng; **không bao giờ** đưa một persona chỉ lát cắt nó sắp trả lời. Bỏ qua điều này và họ **trôi lệch pha** — thành **các buổi tư vấn riêng mặc áo của một bữa tiệc**.

#### c) Spawning

Đưa mỗi agent: **mục tiêu**, **persona của nó**, và **phòng cho tới giờ** — người dùng nói gì và những người khác nói gì, dù nó có phản ứng với cái đó hay không.

| Với thành viên tùy biến | Làm gì |
| --- | --- |
| `persona` | Đưa làm **nhân vật** của họ |
| `capabilities` | **Gấp vào brief** |
| `model` | Spawn với model đó (session `--model` pin **thắng cho tất cả**) |

**Luôn mang hai thứ vào brief:**

1. `scene` của nhóm và **mọi chỉ dẫn hành vi trong persona là chỉ đạo RÀNG BUỘC**
2. Bất cứ gì **có thể đã cũ** so với cutoff huấn luyện phải **kiểm bằng web search chứ không đoán**

**Ranh giới quan trọng:**

```mermaid
graph LR
  A["TIN vao TU DUY cua ho"] --> A1["De ho quyet dinh doc gi va di toi quan diem the nao"]
  A1 --> A2["DUNG kich ban hoa noi dung bang checklist nen-va-khong-nen<br/>DO la thu san sinh ra khoi thit vo hon"]
  B["GIU lai HINH THUC"] --> B1["Tran do dai (thuong 1-3 cau)"]
  B1 --> B2["Chi dan PHAN UNG voi cai vua duoc noi<br/>thay vi nop mot bao cao"]
  A2 --> C["Rang buoc do dai va lap truong BAO VE hoi thoai<br/>Rang buoc LAP LUAN cua ho GIET no"]
  B2 --> C
```

**Song song hay tuần tự:**

| Cách | Khi nào |
| --- | --- |
| **Song song** | Muốn ý kiến đầu tiên **độc lập** |
| **Tuần tự** | Muốn họ phản ứng với **lời thật** của nhau |

> Giữ **vài giọng một vòng** — nhiều hơn đọc như một **đám đông**, không phải một cuộc hội thoại.

#### d) Dệt các câu trả lời thành một hội thoại

Vấn đề: một vòng chạy song song nghĩa là **chưa agent nào thấy lượt của người khác trong cùng vòng đó** — nên để nguyên, họ trả lời **cạnh nhau**, không phải **với nhau**.

```mermaid
graph TB
  A["Ket qua tho tu spawn song song"] --> B["1. SAP XEP LAI luot<br/>de mot phan bac roi NGAY SAU cai no phan bac"]
  B --> C["2. THEM cach noi lien ket ma hoi thoai that co<br/>'Hold on, Winston, that's backwards'<br/>'Sally's right about the API, but she's missing the cost'"]
  C --> D["3. De MOT persona nhat lai mach ma nguoi khac bo roi"]
  D --> E["KHONG BAO GIO doi cai mot agent da lap luan<br/>DET phan trinh bay, BAO TON phan thuc chat"]
```

#### e) Chọn model

Khớp model với vòng: **cái gì nhanh cho đùa cợt, cái gì mạnh hơn cho việc sâu**. `model` theo thành viên được dùng khi có; session `--model <name>` **ghi đè cho tất cả**.

### 5.3 `agent-team` — chỉ Claude Code

Dựng persona thành **đội agent bền vững** mà thành viên **nói trực tiếp với nhau**, nên qua lại **xảy ra thật** thay vì được khâu lại sau.

> Vai trò của bạn chuyển từ **dệt** sang **chủ trì**: khởi động chủ đề, giữ lượt ngắn và trong vai, kéo mạch về khi nó lạc, và đưa trao đổi ra cho người dùng. Giọng, sự ngắn gọn, và va chạm **vẫn giữ nguyên**.

**Khác biệt then chốt so với `subagent`:**

> *Messaging is **point-to-point** — there's **no shared feed**, so a member that sat a round out **hasn't seen** what passed while it was idle.*

| Hệ quả | Việc của lead |
| --- | --- |
| Không có feed chung | **Relay** mỗi lượt người dùng tới các thành viên cần nó |
| Thành viên ngồi ngoài không thấy gì | **Bắt kịp** nó về cái nó bỏ lỡ **trước khi** nó nói lại |
| Đồng đội nhắn nhau được bằng tên | Nhưng **chỉ người trong trao đổi đó thấy** |

> *Keeping everyone in sync is **the lead's job, not the channel's**.*

---

## 6. Bộ nhớ theo party — `references/party-memory.md`

### 6.1 Ai có bộ nhớ

```mermaid
graph TB
  A[Loai phong] --> B["Phong MAC DINH (agent da cai)<br/>-> theo {workflow.party_memory}"]
  A --> C["Group CO TEN<br/>-> co CO 'memory' RIENG cua no<br/>KHONG theo party_memory"]
  A --> D["Cast inline ad-hoc<br/>-> KHONG co bo nho, luon phu du<br/>cho toi khi duoc luu thanh mot party"]
```

Cả hai đều được `resolve_party.py` phân giải thành `memory_enabled`.

### 6.2 Vị trí

```
{workflow.memory_dir}/{active}/.memlog.md
```

`{active}` là khóa mà `resolve_party.py` **đã trả về** — id nhóm (ví dụ `code-review-crew`), hoặc **`installed`** cho phòng mặc định.

### 6.3 Đọc khi vào — **chưng cất, đừng đổ**

```mermaid
graph LR
  A["memlog chi-noi-them, LON len moi phien"] -->|"KHONG keo file tho vao party"| X
  A --> B["Dua mot READER SUBAGENT duong dan memlog"]
  B --> C["No tra ve BRIEF GON<br/>vai tram token ve 'moi thu dang o dau BAY GIO'<br/>san sang dien trong vai"]
  C --> D["Brief dinh hinh phong TU NHIP DAU, TRONG VAI"]
```

**Cách brief định hình phòng:**

| Trạng thái | Biểu hiện |
| --- | --- |
| Một cặp lạnh nhạt | **Mở đầu lạnh** |
| Một liên minh | **Mở đầu ấm** |
| Mạch chuyện cũ | **Nhặt lại** |
| Callback | **Rơi đúng chỗ khi nó hợp** — **hữu cơ, không đọc thuộc lòng khi vừa thấy** |

> **KHÔNG BAO GIỜ phá bức tường thứ tư:** phòng **nhớ**; nó **không bao giờ tuyên bố đã nạp cái gì**, và **không ép** thứ gì không hợp.

### 6.4 Khi nào ghi

| Thời điểm | Nội dung |
| --- | --- |
| **Một khoảnh khắc đáng nhớ rơi xuống** | Một va chạm làm đổi nhiệt độ phòng, một liên minh hình thành, một câu đáng callback sau này, một quyết định, một kết cục |
| **Sàn tối thiểu** | Khi đã có **vài trao đổi thật** từ đầu, **kể cả khi không có gì kịch tính**, ghi lại **nó nói về cái gì** và **động lực mở đầu** |
| **Wrap-up** | Nếu người dùng báo xong ⇒ **top up** với kết cục cuối và bất cứ gì đáng nhớ chưa được ghi |

> **Việc ghi là IM LẶNG.** Phòng **không bao giờ** tuyên bố "đã ghi nhận" hay "tôi sẽ nhớ".

### 6.5 Cái gì đáng nhớ

**Phép thử cho mọi mục:** *liệu điều này có tô màu một phiên tương lai, làm một callback rơi đúng chỗ, hoặc cải thiện bữa tiệc không?* **Nếu không, để nó ra ngoài.**

> **Một nắm mục, không bao giờ là bản tóm lược, không bao giờ là bản ghi.** Giữ mỗi mục **ngắn nhất có thể nhưng dùng được bởi LLM tương lai**.

### 6.6 Gương mặt mới

Khi một nhân vật xuất hiện mà **không có trong roster** của party — cast từ scene open-cast, hoặc người dùng thêm vào giữa chừng — **nêu tên họ trong mục ghi lại khoảnh khắc đó** (`"<name> turned up and …"`) để một gương mặt lặp lại có thể **quay lại phiên sau**.

Ở wrap-up, đây là những gương mặt phòng **đề nghị giữ**, lưu vào roster qua `references/create-party.md`. **Cho tới khi được lưu, họ chỉ sống trong memlog**, và phòng **triệu hồi lại họ từ đó**.

### 6.7 Lệnh ghi

```bash
uv run {project-root}/_bmad/scripts/memlog.py append \
  --workspace {workflow.memory_dir}/{active} \
  --type <dynamic|moment|callback|outcome> \
  --text "<một dòng súc tích, theo cách phòng đọc nó>"
```

Thêm `--by <persona-code>` khi một ký ức thuộc về **một nhân vật**.

**Chọn `init` hay `append`:**

> *Choose from the **existence fact you already hold**: the entry-read (and, on a mid-session room switch, that room's read) told you whether the memlog exists — `init` **once before the first append** when it doesn't, plain `append` when it does. (**`init` errors if the file already exists, so don't call it blind.**)*

**Xử lý lỗi:**

> *If `memlog.py` is unavailable or a write errors, **skip it silently** and **never stall the party** on a failed write.*

### 6.8 Quên

| Muốn | Làm |
| --- | --- |
| Xóa **toàn bộ** bộ nhớ một party | **Xóa thư mục** `{workflow.memory_dir}/{active}/` |
| **Sửa** một ký ức sai | **Nối một mục mới thay thế nó** — phòng đọc trạng thái mới nhất |

> Memlog **chỉ-nối-thêm theo thiết kế** — không có xóa phẫu thuật.

> Giữ mục **thưa**. Bản đọc chưng cất giữ **phòng** gọn dù log lớn đến đâu, nhưng **file trên đĩa vẫn lớn dần**.

---

## 7. Tạo party — `references/create-party.md`

### 7.1 Đầu ra là gì

**Mục override `[workflow]` thưa** cho `bmad-party-mode`:

| Mục | Trường |
| --- | --- |
| `[[workflow.party_members]]` | `code`, `name`, `icon`, `title`, `persona`, `capabilities` (tùy chọn), `model` (tùy chọn) |
| `[[workflow.party_groups]]` | `id`, `name`, `scene` (tùy chọn), `members` (**tùy chọn** — bỏ đi ⇒ open-cast), `memory` (`true`/`false`) |
| `default_party` | Chỉ đặt nếu người dùng muốn nhóm này nạp mặc định |

> **`bmad-customize` làm việc ghi thật.** Skill này chỉ sản xuất cấu hình.

### 7.2 `scene` là gì

Một dòng tự do (hoặc vài dòng) dựng sân khấu cho một phòng: **bối cảnh, chuyện gì đang xảy ra, phòng hành xử thế nào, và mọi ghi chú nhân vật tại-thời-điểm** — ai đã ba ly, ai thù địch với ai, ai thử lửa gắt nhất.

> **Đó là cách CÙNG thành viên vận hành NHIỀU phòng khác nhau** (đội cầu tàu đang trực vs. cùng đội đó lúc rảnh ở phòng chờ vs. một panel người mua thù địch).

> **Định nghĩa mỗi thành viên MỘT LẦN; thay đổi `scene` theo nhóm thay vì định nghĩa lại con người.**

Không có từ vựng cố định — viết rõ ràng và model sẽ diễn.

### 7.3 Trường `persona` là toàn bộ trò chơi

> *A flat title produces a flat voice; **the detail you elicit is what makes a member unmistakably themselves at the table**.*

### 7.4 Năm hình dạng party

| Hình dạng | Mô tả | Cấu hình |
| --- | --- | --- |
| **A cast** | Một ensemble theo chủ đề ("đội cầu tàu Star Trek TOS", "hội đồng các nhà đầu tư nổi tiếng") | Nhiều `party_members` + một `party_group` giữ họ |
| **One-offs** | Một hai persona thêm vào tập thể | Chỉ `party_members`, **không cần group** |
| **Distilled from data** | Người dùng đưa vật liệu nguồn (bảng profile khách hàng, export khảo sát, ghi chú phỏng vấn) để nén thành N persona điển hình | **Đây là cách dựng một AI focus group** cho ideation/feedback sản phẩm |
| **A panel of lenses** | Người review chuyên dụng, mỗi người một góc phê bình sắc (security engineer, kẻ hoài nghi đối kháng giả định nó hỏng, thợ săn ca biên, thợ thủ công ghét sự khôn lỏi và trùng lặp, người thực dụng phản lại chủ nghĩa hoàn hảo) | `scene` bảo họ tấn công từ lăng kính của mình **và cãi nhau về cái gì thực sự quan trọng**. **Phòng adversarial-review hoặc red-team tuyệt vời** |
| **Open-cast** | **Không có roster cố định**. `scene` nêu tên một pool hoặc vũ trụ và phòng được cast tại chỗ | **Bỏ `members` đi**; model đã biết vũ trụ và chọn ai hợp khoảnh khắc. Neo một hai gương mặt bằng cách liệt kê họ nếu ai đó phải luôn có mặt |

### 7.5 Bốn tình huống đặc biệt

#### a) Giữ lại cast đang chơi

Khi tới đây **từ một phiên đang sống** — người dùng dựng cast ad-hoc inline và muốn giữ:

> **Đừng thẩm vấn lại** — persona đã được soạn và lồng tiếng rồi. **Ghi lại họ đúng như họ đã diễn**, cho nhóm một `id` và tên, hỏi câu hỏi memory và default, rồi **đi thẳng tới việc ghi**.

#### b) Sửa party đã tồn tại

```bash
uv run {project-root}/_bmad/scripts/resolve_customization.py --skill {skill-root} --key workflow
```

Trả về `party_members`, `party_groups`, `default_party` **đã hợp nhất**.

> **Đọc trạng thái hiện tại TRƯỚC để bạn THAY ĐỔI chứ không ĐẬP NÁT.** Hiện thành viên/nhóm đang được chạm, **chỉ nắm bắt phần delta** với người dùng, và đưa **thay đổi thưa** đó cho `bmad-customize` — nó thay mục có `code`/`id` khớp và nối phần còn lại, nên **một lần sửa chỉ là mục đã đổi, không bao giờ là viết lại toàn bộ**.

#### c) Giữ gương mặt mới từ một phiên

Ở cuối một party có bộ nhớ, phòng đề nghị giữ gương mặt xuất hiện mà không có trong roster.

> Họ **đã được soạn và lồng tiếng**, nên **đừng thẩm vấn lại** — ghi lại mỗi người **đúng như họ đã diễn** (`code`, `name`, `icon`, một dòng `title`, và `persona` rút ra từ cách họ thể hiện), rồi thêm vào `party_members`.

| Loại nhóm | Cũng phải làm gì |
| --- | --- |
| Nhóm **roster cố định** | **Liệt kê code của họ** trong `members` của nhóm để họ quay lại như người thường trực |
| Phòng **open-cast** | **Để `members` rỗng** — liệt kê bất kỳ thành viên nào **biến phòng thành roster cố định và giết chết việc cast tại chỗ**. Persona đã lưu giờ sống trong tập thể, nên scene vẫn nêu tên họ và họ có thể quay lại **mà không khóa phòng lại** |

#### d) Chưng cất từ dữ liệu nguồn

```mermaid
graph TB
  A["Nguoi dung chi vao du lieu:<br/>duong dan file, bang dan vao, profile export"] --> B["Doc va NEN thanh so persona dai dien duoc yeu cau"]
  B --> C["CUM theo cai THUC SU phan biet HANH VI<br/>muc tieu, ngan sach, noi dau, tu the tiep nhan<br/>KHONG chi nhan khau hoc be mat"]
  C --> D["Moi cum thanh MOT persona co ten that va guong mat"]
  D --> E["NEU TEN LY LUAN cua ban:<br/>tim thay phan khuc nao, dac diem nao dan toi cach chia<br/>de ho SUA duong cat TRUOC khi ban to thit persona"]
  E --> F{"Ho co noi bao nhieu persona khong?"}
  F -->|Khong| G["De xuat mot so tu do trai cua du lieu<br/>va de ho dieu chinh"]
```

**Với focus-group panel:**

> *independent answers matter more than banter, so **offer to set `party_mode` to `subagent`** (or remind them `--mode subagent` does it per session) — **otherwise one mind voices every customer and they bleed together**.*

### 7.6 Tô thịt mỗi persona

> **Draft, don't interrogate.** Đề xuất bản cắt đầu tiên của mỗi persona và để người dùng phản ứng — **nhanh hơn nhiều so với một bảng câu hỏi**. Đẩy mỗi cái cho tới khi nó có **một giọng bạn có thể nhận ra khi bịt mắt**.

| Chiều | Nội dung |
| --- | --- |
| **Identity** | Tên, một dòng title, một emoji hợp |
| **Voice & ethos** | Họ nói thế nào, họ coi trọng gì, họ tranh luận thế nào, thứ họ ghét |
| **Agenda** | Họ thực sự theo đuổi gì trong bất kỳ cuộc trò chuyện nào; họ đẩy cho cái gì |
| **Quirks** | Chi tiết cụ thể, con người (một câu cửa miệng, một thiên kiến, một điểm mù) |
| **Likes & dislikes** (focus-group) | Cái gì làm họ **vô địch** hoặc **từ chối** một ý tưởng, và quan hệ của họ với không gian sản phẩm |
| **Capabilities** (tùy chọn) | Nếu persona này nên nghiên cứu hoặc đọc file khi được spawn ⇒ ghi chú; nó thành **hướng dẫn mềm** trong prompt spawn |

**Ví dụ về độ cụ thể:**

| ❌ Placeholder | ✅ Persona |
| --- | --- |
| "CFO hoài nghi" | "không duyệt bất cứ gì không hoàn vốn dưới 18 tháng, và **nói ra điều đó trong ba mươi giây đầu**" |

### 7.7 Đóng lại

| Hỏi thẳng | Ghi chú |
| --- | --- |
| **Còn gì về party này cần chỉ định** trước khi ghi? | Động lực nội bộ, một giọng còn thiếu, một thành viên nên dẫn dắt |
| Party này **có nên nhớ qua các phiên** không? | Có ⇒ `memory = true`; không ⇒ `memory = false`. **One-off không có group thì bỏ qua** — memory là thiết lập cấp group |
| Nhóm này **có nên là party mặc định** từ giờ? | Có ⇒ `default_party = <id>`. One-off không có group **không thể** là mặc định; bỏ qua |

### 7.8 Ghi qua `bmad-customize`

**Trước tiên — kiểm tra va chạm `code`:**

> *A custom member whose `code` matches an installed agent **silently OVERRIDES that agent** in the collective.*

```bash
uv run {skill-root}/scripts/resolve_party.py --project-root {project-root} --skill {skill-root}
```

Kiểm `code` của từng thành viên mới với danh sách trả về. Khi va chạm ⇒ **nêu ra** (*"`analyst` sẽ ghi đè Analyst đã cài — cố ý, hay chọn code khác?"*) và để người dùng xác nhận hoặc đổi tên.

> **Một lần kiểm, không phải một cổng chặn.**

Rồi soạn override **thưa** và đưa cho `bmad-customize`:

| Tham số | Giá trị |
| --- | --- |
| Skill đích | `bmad-party-mode` |
| Bề mặt | `[workflow]` |
| Mặc định | **user** override (`bmad-party-mode.user.toml`) |
| Đề nghị | File **team** khi party dành để chia sẻ |

> **Giữ nó thưa — chỉ mục mới, KHÔNG BAO GIỜ chép base `customize.toml`.** `bmad-customize` hiện TOML, **chờ yes tường minh**, ghi, và verify merge — **đừng tự ghi file**.

Sau khi xong, nói cho người dùng cách dùng: `--party <id>` để triệu hồi nhóm, hoặc nó giờ là mặc định nếu họ chọn vậy.

---

## 8. Wrap-up

```mermaid
graph TB
  A["Nguoi dung bao xong - DOC PHONG, dung cho tu khoa ma thuat<br/>HOAC mot lan chay --non-interactive da phuc vu y dinh cua no"] --> A1["KHONG BAO GIO chi vi prompt mo dau da duoc tra loi"]
  A1 --> B["1. DOC LAI cac takeaway tot nhat"]
  B --> C{"2. Bo nho dang bat?"}
  C -->|Co| C1["TOP UP memlog voi ket cuc cuoi<br/>va bat ky nhip dang nho chua duoc ghi<br/>-> LA MOT TOP-UP; bo nho da tich luy TRUC TIEP"]
  C -->|Khong| D
  C1 --> D["3. DE NGHI keepsake"]
  D --> D1["MOT file HTML tu chua RAT SANG TAO cua phien<br/>bo cuc THEO PERSONA (icon, ten, giong)<br/>vat ky niem thuc su dep<br/>SVG inline / hoat hoa nhe khi no nang tac pham<br/>Ghi ra .html co dau {date} vao {workflow.output_dir}/<br/>hoac bat cu dau ho muon"]
  D1 --> E{"4. Bo nho bat VA co guong mat moi khong trong roster?"}
  E -->|Co| E1["DE NGHI MOT LAN luu ho vao tuy bien party cua nguoi dung<br/>Co -> theo references/create-party.md<br/>(TU CHOI DUOC; dung lam nghen viec dong phien)"]
  E -->|Khong| F
  E1 --> F["5. Chay {workflow.on_complete} neu khong rong"]
  F --> G["6. Roi ve che do binh thuong"]
```

### 8.1 Bữa tiệc là tương tác và mở

> *The opening prompt is **a topic to dig into, not a task that ends the party** once it's answered — it runs **round after round** until the **user** signals done.*

> *A served opening intent means **"what's next?"**, never **"we're finished"**: **don't wrap up, disband the room, or close spawned agents** just because the first ask is satisfied.*

**Ngoại lệ duy nhất:** `--non-interactive` tường minh — chạy party trên ý định đã cho tới một kết thúc tự nhiên, rồi wrap up và giải phóng agent. **Đó là con đường không tương tác duy nhất, và chỉ khi người dùng yêu cầu.**

---

## 9. `customize.toml` — cấu trúc

```toml
[workflow]

activation_steps_prepend = []
activation_steps_append  = []
persistent_facts = ["file:{project-root}/**/project-context.md"]

default_party = ""                    # rỗng = agent đã cài
party_mode    = "session"             # session | auto | subagent | agent-team
output_dir    = "{output_folder}/party-mode"
party_memory  = true                  # CHỈ cho phòng mặc định
memory_dir    = "{output_folder}/party-mode/memories"
on_complete   = ""

[[workflow.party_members]]            # khóa `code`
code = "..."
name = "..."
icon = "..."
title = "..."
persona = "..."
capabilities = "..."                  # tùy chọn
model = "..."                         # tùy chọn

[[workflow.party_groups]]             # khóa `id`
id = "..."
name = "..."
scene = "..."                         # tùy chọn
members = ["code1", "code2"]          # TÙY CHỌN — bỏ đi = open-cast
memory = true                         # mặc định false khi bỏ qua
```

### 9.1 Ghi chú quan trọng về `default_party`

> *Empty = the installed BMAD agents — **exactly the default behavior of a plain install**. Custom members defined below join the **POOL** (usable in groups, and summonable by name) **but do NOT crowd this default room**.*

```mermaid
graph TB
  A[9 party_members shipped] --> B[Vao POOL]
  B --> C["Dung duoc trong group"]
  B --> D["Trieu hoi duoc bang TEN"]
  B -.KHONG.-x E["Phong MAC DINH<br/>chi co agent DA CAI"]
  F["default_party = 'code-review-crew'"] --> G["Ghim mot phong duoc chon lam mac dinh"]
  H["--party <id> luc chay"] --> I["LUON THANG"]
```

> **Chúng không tốn gì cho tới khi được triệu hồi.**

### 9.2 `party_memory` vs `memory` của group

| Cấu hình | Áp cho |
| --- | --- |
| `party_memory = true/false` | **CHỈ** phòng mặc định (agent đã cài) |
| `[[workflow.party_groups]].memory` | **Từng group có tên — KHÔNG theo `party_memory`** |
| Cast inline ad-hoc | **Luôn phù du** cho tới khi được lưu thành party |

---

## 10. Chín persona dựng sẵn

Tất cả đều trong **pool**, không trong phòng mặc định.

| `code` | Tên | Icon | Chức danh | Đặc điểm |
| --- | --- | --- | --- | --- |
| `sec-hawk` | Vex | 🔒 | Security Engineer | Threat-model mọi thứ. Săn injection, authz hỏng, secret rò rỉ, SSRF, rủi ro chuỗi cung ứng. Giả định **mọi input là thù địch và mọi dependency đã bị xâm nhập** cho tới khi chứng minh ngược lại. **Nêu đường khai thác cụ thể** — *"đây là cách tôi chiếm cái máy này"* — không bao giờ khoát tay "có thể không an toàn". *Capabilities: đọc code và lần theo luồng dữ liệu từ input không tin cậy tới sink trước khi phán* |
| `adversary` | Grumbal | 😤 | The Adversary | **Giả định code hỏng và việc của mình là chứng minh điều đó.** Cáu kỉnh, thẳng thừng, **không có bánh mì kẹp khen ngợi**. Bắt đầu từ *"cái này sẽ gọi ai đó lúc 3 giờ sáng"* và lần ngược tới dòng gây ra nó. Dị ứng với lạc quan và "chắc ổn thôi" |
| `edge-hunter` | Boundary | 🌶️ | Edge-Case Hunter | Đi mọi nhánh và biên. Input rỗng, null, off-by-one, payload khổng lồ, gọi đồng thời, tên unicode, timezone, bão retry. **Theo phương pháp, không ác ý**: *"chuyện gì xảy ra khi cái này được gọi hai lần cùng lúc?"* |
| `craftsman` | Yui | 🎯 | The Craftsman | Quan tâm sự đơn giản, cách đặt tên, và tái sử dụng. **Dị ứng với sự khôn lỏi và trùng lặp.** *"Bạn viết lại thứ đã tồn tại"*, *"cái tên này nói dối về việc nó làm"*, *"ba tầng trừu tượng lồng nhau ở chỗ một tầng là đủ"*. Muốn phiên bản **nhàm chán, hiển nhiên, bảo trì được** |
| `shipper` | Dana | 🚢 | The Pragmatist | **Đối trọng với những người cầu toàn để phòng không thành một cuộc đập hội đồng.** *"Cái này có thực sự quan trọng với người dùng không? Ship 80%, ghi phiếu phần còn lại."* Đẩy lùi mạ vàng và rủi ro lý thuyết, **buộc mọi người xếp hạng cái gì thật vs cái gì là bới lông** |
| `option-generator` | Wildcard | 🃏 | Option Generator | Tìm **các phương án phòng chưa xét**. Đề xuất cách phát biểu vấn đề khác, giả định khác, ví dụ đơn giản. **Phải giải thích vì sao mỗi phương án quan trọng bằng ngôn ngữ thường**, và **bỏ ý tưởng nhanh khi chúng không giúp** |
| `claim-checker` | Level | 📏 | Claim Checker | Kiểm tra khẳng định **có được chống lưng không**. Hỏi bằng chứng nào tồn tại, bằng chứng nào thiếu, cái gì sẽ đổi câu trả lời, phòng nên tự tin đến mức nào. **Giữ sự bất định hiển ngôn** và tránh giả vờ một khẳng định yếu là đã ngã ngũ |
| `loop-stopper` | Killjoy | 🛑 | Loop Stopper | **Dừng thảo luận khi nó ngừng tạo ra giá trị.** Gọi tên sự lặp lại, bất đồng giả, phức tạp hóa quá mức, và suy đoán không có cơ sở. Khi phòng lặp lại chính nó, **hỏi câu hỏi chưa giải quyết nào thực sự quan trọng với con người** |
| `consensus-challenger` | Splinter | 🪵 | Consensus Challenger | **Thách thức sự đồng ý dễ dãi.** Tìm giả định ẩn, đánh đổi bị bỏ qua, phản đối yếu, và phương án phòng gạt đi quá nhanh. **Không cãi để mà cãi**; một khi rủi ro đã rõ, **trả quyết định lại cho con người** |

---

## 11. Hai party dựng sẵn

### 11.1 `code-review-crew`

```toml
[[workflow.party_groups]]
id = "code-review-crew"
name = "Code Review Crew"
scene = "Adversarial code review. Each reviewer attacks from their own lens and they argue with each other about what actually matters — security versus shipping, elegance versus pragmatism. No rubber-stamping, no praise sandwiches: surface the real problems before they ship. Point at the line, name the failure mode, and defend it when someone pushes back. Best run with `--mode subagent` so each lens reviews independently before they clash."
members = ["sec-hawk", "adversary", "edge-hunter", "craftsman", "shipper"]
memory = false
```

**Năm lăng kính, hai trục xung đột được thiết kế sẵn:**

```mermaid
graph LR
  SEC[sec-hawk<br/>bao mat] <-->|xung dot| SHIP[shipper<br/>ship duoc chua]
  CRAFT[craftsman<br/>thanh lich] <-->|xung dot| SHIP
  ADV[adversary<br/>gia dinh hong]
  EDGE[edge-hunter<br/>moi bien]
```

`memory = false` — **mỗi review tự đứng một mình**; lật `true` để nhớ các review trước.

### 11.2 `anti-consensus-club`

```toml
[[workflow.party_groups]]
id = "anti-consensus-club"
name = "Anti-Consensus Club"
members = ["option-generator", "claim-checker", "loop-stopper", "consensus-challenger"]
memory = false
```

**`scene` của nhóm này là một ví dụ đặc biệt** — nó chứa **chỉ dẫn vận hành**, không chỉ dàn cảnh:

| Chỉ dẫn trong scene | Nội dung |
| --- | --- |
| **Kiểm tra mode ở đầu phiên** | Trước thảo luận thực chất, kiểm mode hiện tại |
| **Khuyến nghị `--mode subagent` mạnh** | Nếu chưa chạy subagent và nền tảng hỗ trợ — vì **cửa sổ ngữ cảnh riêng làm giảm khả năng một ngữ cảnh chung khiến mọi giọng đồng ý quá nhanh** |
| **Không cằn nhằn sau đó** | Một khi người dùng chọn tiếp tục |
| **Phòng này HỖ TRỢ phán đoán của con người, KHÔNG THAY THẾ nó** | **Không bỏ phiếu, không tuyên bố đồng thuận, không nói như thể phòng có thẩm quyền** |
| Phân vai rõ | Wildcard đề xuất thêm phương án. Level kiểm bằng chứng và độ tin cậy. Killjoy dừng thảo luận lặp lại/không cơ sở. Splinter thách thức đồng ý dễ dãi |
| **Nếu phòng đồng ý quá nhanh** | **Nêu tên giả định ẩn** |
| **Nếu phòng bắt đầu lặp lại** | **Dừng và hỏi con người câu hỏi chưa giải quyết nào quan trọng** |

> Đây là ví dụ điển hình cho thấy `scene` không chỉ là "trang trí" — nó là **kênh chỉ đạo hành vi ràng buộc**.

---

## 12. Vận hành thủ công

```bash
R="$(pwd)"
SK="$R/.claude/skills/bmad-party-mode"

# ── Xem roster đang hoạt động ──────────────────────────────────
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK"

# ── Chỉ xem menu các nhóm ──────────────────────────────────────
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK" --list-groups

# ── Xem một nhóm cụ thể ────────────────────────────────────────
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK" --party code-review-crew

# ── Xem cấu hình đã hợp nhất ───────────────────────────────────
uv run "$R/_bmad/scripts/resolve_customization.py" -s "$SK" -p "$R" -k workflow.party_members
uv run "$R/_bmad/scripts/resolve_customization.py" -s "$SK" -p "$R" -k workflow.party_groups
uv run "$R/_bmad/scripts/resolve_customization.py" -s "$SK" -p "$R" -k workflow.default_party

# ── Xem bộ nhớ của một party ───────────────────────────────────
cat "$R/_bmad-output/party-mode/memories/installed/.memlog.md"
cat "$R/_bmad-output/party-mode/memories/code-review-crew/.memlog.md"

# ── Xóa bộ nhớ một party ───────────────────────────────────────
rm -rf "$R/_bmad-output/party-mode/memories/code-review-crew/"

# ── Ghi tay một ký ức ──────────────────────────────────────────
MEM="$R/_bmad-output/party-mode/memories/installed"
uv run "$R/_bmad/scripts/memlog.py" init --workspace "$MEM" --field party="installed"
uv run "$R/_bmad/scripts/memlog.py" append --workspace "$MEM" \
  --type dynamic --by sec-hawk \
  --text "Vex và Dana đụng nhau về việc có chặn release vì lỗ hổng medium không — chưa ngã ngũ"
uv run "$R/_bmad/scripts/memlog.py" append --workspace "$MEM" \
  --type outcome \
  --text "phòng nghiêng về ship với feature flag, Vex ghi nhận nhưng không đồng ý"
```

### 12.1 Tự tạo một party bằng tay

```bash
mkdir -p "$R/_bmad/custom"
cat > "$R/_bmad/custom/bmad-party-mode.user.toml" <<'TOML'
[workflow]

default_party = "phong-kien-truc"
party_mode = "subagent"

[[workflow.party_members]]
code = "dba"
name = "Hải"
icon = "🗄️"
title = "Database Architect"
persona = "Nghĩ bằng query plan. Hỏi ngay về index, cardinality, và lock contention trước khi bàn về API. Ghét ORM sinh N+1. Nói bằng EXPLAIN ANALYZE. Khi ai đó nói 'cứ thêm cache đi', anh ta hỏi 'invalidate bằng gì?'"
capabilities = "Đọc schema và migration trước khi phán về hiệu năng."

[[workflow.party_members]]
code = "ops"
name = "Trâm"
icon = "📟"
title = "SRE"
persona = "Đo mọi thứ bằng thời gian khôi phục. Câu hỏi đầu tiên luôn là 'khi nó hỏng lúc 2 giờ sáng thì ai bị gọi và họ nhìn vào đâu?'. Ghét thứ không có dashboard. Bình tĩnh đến mức khó chịu."

[[workflow.party_groups]]
id = "phong-kien-truc"
name = "Phòng Kiến trúc"
scene = "Phòng họp cuối ngày, whiteboard đầy chữ. Ai cũng đã cãi nhau về chuyện này một lần rồi. Không ai nhường ai, nhưng ai cũng tôn trọng bằng chứng. Hải và Trâm thường liên minh chống lại mọi thứ 'sẽ ổn thôi'."
members = ["dba", "ops", "craftsman", "shipper"]
memory = true
TOML

# Xác minh
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK" --list-groups
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK" --party phong-kien-truc
```

### 12.2 Kiểm tra va chạm `code`

```bash
# Code nào đã tồn tại?
uv run "$SK/scripts/resolve_party.py" --project-root "$R" --skill "$SK" \
  | python -c "
import json,sys
d = json.load(sys.stdin)
for m in d.get('members', []) + d.get('agents', []):
    print(m.get('code'))
" | sort
```

---

## 13. Cạm bẫy

| Vấn đề | Nguyên nhân | Xử lý |
| --- | --- | --- |
| Phòng đọc như báo cáo | Vi phạm tiêu chí 1 | *"the instant it reads like answers being filed, the party's dead"* |
| Mọi giọng nghe giống nhau | Vi phạm tiêu chí 2 | Giấu nhãn đi vẫn phải phân biệt được |
| Bạn hòa giải xung đột | Vi phạm tiêu chí 3 | **Chống lại bản năng thắt nơ** |
| Trình bày dãy câu trả lời | Vi phạm tiêu chí 4 | Phải là **một** hội thoại, lượt nối tiếp |
| Diễn giải lời persona ở ngôi ba | Vi phạm tiêu chí 4 | Để họ tự nói |
| Phá bức tường thứ tư | Vi phạm tiêu chí 8 | Không nói "bạn có 4 agent trong phòng" |
| Chèn tóm tắt/takeaway | Vi phạm tiêu chí 9 | Chỉ khi người dùng hỏi |
| Kết thúc party khi prompt đầu được trả lời | Hiểu sai §8.1 | *"A served opening intent means 'what's next?'"* |
| Đóng subagent khi nó xong việc | Vi phạm vòng đời | **Rảnh, không phải xong** |
| Persona ngồi ngoài bị mất đồng bộ | Không route toàn bộ trao đổi | Thành "các buổi tư vấn riêng mặc áo party" |
| Memlog bị đổ nguyên vào ngữ cảnh | Vi phạm §6.3 | Dùng **reader subagent** chưng cất |
| Tuyên bố "tôi sẽ nhớ" | Vi phạm §6.4 | **Việc ghi là im lặng** |
| `init` gọi mù ⇒ lỗi | Vi phạm §6.7 | Dùng sự kiện tồn tại bạn đã có từ lần đọc vào |
| Party chết vì lỗi ghi memlog | Vi phạm §6.7 | **Bỏ qua im lặng, không bao giờ làm nghẽn** |
| Thêm `members` vào open-cast room | Vi phạm §7.5c | **Giết chết việc cast tại chỗ** |
| `code` tùy biến ghi đè agent đã cài | Không kiểm va chạm | Chạy kiểm tra ở §12.2 trước |
| Tự ghi file override | Vi phạm §7.8 | **`bmad-customize` làm việc ghi** |

---

**Tiếp:** [B9 — v6-shims](./B9-v6-shims.md) · [← Chỉ mục](./index.md)
