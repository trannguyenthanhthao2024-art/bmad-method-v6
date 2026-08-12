# Hướng dẫn đọc mã nguồn BMAD-METHOD

> Mục tiêu: **đọc hiểu mã nguồn của tool để tái sử dụng mẫu hình ở dự án khác.**
>
> Đây không phải tài liệu API. Nó là **bản đồ + đường đọc + mẫu hình rút ra**, viết cho người muốn xây một công cụ tương tự hoặc mượn kỹ thuật của nó.

---

## Vì sao đáng đọc mã nguồn này

BMAD-METHOD giải một bài toán khó và giải khá sạch: **phân phối nội dung tri thức có cấu trúc tới ~50 công cụ AI khác nhau, cho phép tùy biến nhiều tầng mà không fork, và giữ tính tất định ở những chỗ cần**.

Bảy kỹ thuật trong đó **tái sử dụng được ở dự án hoàn toàn khác**:

| # | Kỹ thuật | Dùng lại được cho |
| --- | --- | --- |
| 1 | Hợp nhất cấu hình nhiều lớp có quy tắc theo kiểu dữ liệu | Bất kỳ tool nào cần override team/user |
| 2 | Kiến trúc hướng-cấu-hình cho N nền tảng | Tool phải hỗ trợ nhiều target khác nhau |
| 3 | Kết xuất snapshot bất biến định danh bằng hash | Bất kỳ chỗ nào cần cache + kiểm chứng |
| 4 | Registry rule-based cho validator | Linter, kiểm tra chất lượng |
| 5 | Nhật ký chỉ-nối-thêm với ghi nguyên tử | Bất kỳ chỗ nào cần audit trail đáng tin |
| 6 | Bọc thư viện bên thứ ba với lazy-load | Giảm thời gian khởi động CLI |
| 7 | Thay thế `fs-extra` bằng `node:fs` thuần | Tránh monkey-patching gây lỗi khó lần |

---

## Bản đồ mã nguồn

```mermaid
graph TB
  subgraph L1["TANG 1 — PHAN PHOI (Node.js, ~13k dong)"]
    CLI["bmad-cli.js<br/>diem vao commander"]
    CMD["commands/<br/>install, status, uninstall"]
    UI["ui.js (2167)<br/>luong prompt"]
    PR["prompts.js (791)<br/>boc @clack"]
    CORE["core/<br/>installer, manifest-generator,<br/>paths, config"]
    MOD["modules/<br/>channel, version, external,<br/>custom, plugin"]
    IDE["ide/<br/>manager + _config-driven<br/>+ platform-codes.yaml"]
  end

  subgraph L2["TANG 2 — RUNTIME (Python, ~1.2k dong)"]
    CU["config_utils.py (119)<br/>THU VIEN"]
    RC["resolve_config.py (74)"]
    RCU["resolve_customization.py (99)"]
    RS["render_skill.py (401)"]
    ML["memlog.py (224)"]
  end

  subgraph L3["TANG 3 — NOI DUNG (Markdown/TOML/CSV)"]
    SK["src/core-skills/<br/>src/bmm-skills/"]
    MY["module.yaml"]
    MH["module-help.csv"]
    CT["customize.toml"]
  end

  subgraph L4["TANG 4 — CHAT LUONG (Node.js, ~2.5k dong)"]
    VS["validate-skills.js (735)"]
    VF["validate-file-refs.js (563)"]
    VD["validate-doc-links.js (413)"]
    BD["build-docs.mjs (471)"]
  end

  CLI --> CMD --> UI --> PR
  CMD --> CORE --> MOD
  CORE --> IDE
  CORE -.cai dat.-> L2
  CORE -.cai dat.-> L3
  L4 -.kiem tra.-> L3
```

---

## Mục lục

| # | Tài liệu | Nội dung |
| --- | --- | --- |
| 01 | **[Bản đồ và đường đọc](./01-ban-do-va-duong-doc.md)** | Đọc theo thứ tự nào tùy mục tiêu; 6 đường đọc có sẵn |
| 02 | **[Tầng phân phối — Installer Node.js](./02-tang-phan-phoi.md)** | `bmad-cli.js` → `ui.js` → `installer.js` → `manifest-generator.js`; quản lý module và IDE |
| 03 | **[Tầng runtime — Script Python](./03-tang-runtime-python.md)** | 5 file, đọc theo thứ tự phụ thuộc; thuật toán hợp nhất và kết xuất |
| 04 | **[Tầng nội dung — Markdown/TOML](./04-tang-noi-dung.md)** | Nội dung là mã: cách BMAD biểu diễn logic nghiệp vụ bằng văn xuôi có cấu trúc |
| 05 | **[Tầng chất lượng — Validator](./05-tang-chat-luong.md)** | Registry rule-based, tích hợp GitHub Actions, kiểm tra tham chiếu |
| 06 | **[Bảy mẫu hình tái sử dụng](./06-mau-hinh-tai-su-dung.md)** | ★ Phần quan trọng nhất — trích mã thật + cách áp dụng nơi khác |
| 07 | **[Áp dụng vào dự án của bạn](./07-ap-dung-vao-du-an-cua-ban.md)** | Ba kịch bản cụ thể kèm mã khởi đầu |

---

## Đọc từ đâu tùy mục tiêu

```mermaid
graph TB
  Q{Ban muon gi?}
  Q -->|"Hieu tong the truoc"| A["01 Ban do -> 02 §1-2 -> 03 §1"]
  Q -->|"Muon xay CLI installer tuong tu"| B["02 toan bo -> 06 §5,6,7"]
  Q -->|"Muon co che config nhieu lop"| C["03 §2 -> 06 §1"]
  Q -->|"Muon cache co kiem chung"| D["03 §4 -> 06 §3"]
  Q -->|"Muon ho tro N nen tang"| E["02 §5 -> 06 §2"]
  Q -->|"Muon viet validator"| F["05 -> 06 §4"]
  Q -->|"Chi muon lay mau hinh"| G["06 truc tiep"]
  Q -->|"Muon ap dung ngay"| H["07 truc tiep"]
```

---

## Số liệu mã nguồn

| Tầng | Ngôn ngữ | Dòng | File lớn nhất |
| --- | --- | --- | --- |
| Phân phối | Node.js (CommonJS) | ~13.000 | `official-modules.js` (2.257) |
| Runtime | Python 3.11 | ~1.220 | `render_skill.py` (401) |
| Chất lượng | Node.js | ~2.500 | `validate-skills.js` (735) |
| Nội dung | Markdown/TOML/CSV | — | `bmad-deep-recon/customize.toml` (212) |

Tổng mã thực thi: **~19.500 dòng**. Đủ nhỏ để đọc hết trong vài ngày, đủ lớn để có mẫu hình đáng học.

---

## Ba quan sát trước khi đọc

### 1. Tỷ lệ mã / nội dung rất thấp

```mermaid
pie showData
  title "Phan bo cong suc"
  "Ma thuc thi (Node + Python)" : 19500
  "Noi dung tri thuc (MD/TOML/CSV)" : 45000
```

BMAD **không phải** một dự án phần mềm có tài liệu kèm theo. Nó là một **thư viện tri thức có công cụ phân phối kèm theo**. Đọc mã mà không đọc nội dung là hiểu nhầm bản chất.

### 2. Mọi thứ tất định đều bị đẩy xuống Python

| Việc | Ai làm | Vì sao |
| --- | --- | --- |
| Prompt, tải module, copy file | **Node** | Việc I/O và tương tác |
| Hợp nhất TOML, tính hash, kết xuất | **Python** | Cần tất định tuyệt đối |
| Phán đoán nghiệp vụ | **LLM** | Không mã hóa được |

Ranh giới này rất rõ trong mã và đáng học.

### 3. Chú thích trong mã là tài liệu thiết kế

Nhiều quyết định quan trọng nhất **chỉ tồn tại dưới dạng chú thích**. Ví dụ `tools/installer/fs-native.js:1-3`:

```js
// Drop-in replacement for fs-extra using native node:fs APIs.
// Eliminates graceful-fs monkey-patching that causes non-deterministic
// file loss during multi-module installs on macOS (issue #1779).
```

Ba dòng này giải thích một quyết định kiến trúc mà không tài liệu nào khác nhắc tới. **Đọc chú thích, đừng lướt qua.**

---

## Quy ước ký hiệu

| Ký hiệu | Nghĩa |
| --- | --- |
| `file.js:42` | Đường dẫn kèm số dòng — mở được trực tiếp |
| 📖 | Đọc file này |
| ⭐ | Điểm mấu chốt, đừng bỏ qua |
| ♻️ | Mẫu hình tái sử dụng được |
| ⚠️ | Cạm bẫy hoặc quyết định gây tranh cãi |

---

## Tài liệu liên quan

| Muốn hiểu | Đọc |
| --- | --- |
| Hệ thống làm cái gì | [Đặc tả](../tai-lieu-he-thong/01-dac-ta-he-thong.md) |
| Kiến trúc tổng thể | [Thiết kế](../tai-lieu-he-thong/02-thiet-ke-he-thong.md) |
| Vận hành | [Vận hành](../tai-lieu-he-thong/03-van-hanh-he-thong.md) |
| Nội dung module core | [Tài liệu core](../tai-lieu-core/index.md) |
| Ví dụ chạy thật | [Demo greenfield](../demo/index.md) · [Demo brownfield](../demo-brownfield/index.md) |

---

**Bắt đầu:** [01 — Bản đồ và đường đọc](./01-ban-do-va-duong-doc.md)
