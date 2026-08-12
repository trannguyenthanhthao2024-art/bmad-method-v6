# 02 — Thiết kế hệ thống (System Design)

> Thiết kế kiến trúc và chi tiết của **BMAD-METHOD v6.10.0**.
> Trả lời câu hỏi *làm như thế nào*. Quay lại [mục lục](./README.md) · Xem [đặc tả](./01-dac-ta-he-thong.md).

---

## Mục lục

1. [Nguyên lý thiết kế](#1-nguyên-lý-thiết-kế)
2. [Kiến trúc tổng thể](#2-kiến-trúc-tổng-thể)
3. [Cấu trúc mã nguồn](#3-cấu-trúc-mã-nguồn)
4. [Thiết kế tầng phân phối (Installer)](#4-thiết-kế-tầng-phân-phối-installer)
5. [Thiết kế tầng nội dung (Skill/Agent/Workflow)](#5-thiết-kế-tầng-nội-dung-skillagentworkflow)
6. [Thiết kế tầng runtime (Script Python)](#6-thiết-kế-tầng-runtime-script-python)
7. [Thiết kế hệ thống cấu hình](#7-thiết-kế-hệ-thống-cấu-hình)
8. [Thiết kế bộ máy kết xuất (Render Engine)](#8-thiết-kế-bộ-máy-kết-xuất-render-engine)
9. [Thiết kế tích hợp IDE](#9-thiết-kế-tích-hợp-ide)
10. [Thiết kế phiên bản và kênh phát hành](#10-thiết-kế-phiên-bản-và-kênh-phát-hành)
11. [Thiết kế luồng dữ liệu](#11-thiết-kế-luồng-dữ-liệu)
12. [Quyết định kiến trúc (ADR)](#12-quyết-định-kiến-trúc-adr)
13. [Ma trận truy vết](#13-ma-trận-truy-vết)

---

## 1. Nguyên lý thiết kế

Bảy nguyên lý xuyên suốt mọi thành phần:

| # | Nguyên lý | Biểu hiện cụ thể trong mã |
| --- | --- | --- |
| P1 | **Tất định hóa cái gì có thể tất định** | Mọi phép hợp nhất cấu hình, kết xuất, tính hash, chuyển trạng thái sprint đều do script Python làm, không giao cho LLM |
| P2 | **Ngữ cảnh là tài nguyên khan hiếm** | Catalog phục vụ qua script; file bước nạp just-in-time; spec giới hạn 900–1600 token; mỗi workflow một cửa sổ ngữ cảnh |
| P3 | **Con người giữ quyền quyết định** | Checkpoint HALT bắt buộc; không tự áp bản vá; major upgrade mặc định N |
| P4 | **Tùy biến không được fork** | 3–4 lớp TOML; file mặc định mang chú thích `DO NOT EDIT` |
| P5 | **Tương thích ngược bằng shim** | 19 skill shim chuyển tiếp ID cũ; `aliases` cho mã module |
| P6 | **Bất biến hơn khả biến** | Snapshot kết xuất định danh bằng hash; memlog chỉ nối thêm |
| P7 | **Suy thoái duyên dáng** | Mọi lệnh script đều có nhánh fallback đọc file trực tiếp; thiếu subagent thì chạy nội tuyến |

---

## 2. Kiến trúc tổng thể

### 2.1 Sơ đồ tầng

```mermaid
graph TB
  subgraph L4["Tang 4 - Nguoi dung"]
    HUMAN[Nguoi dung]
    AITOOL[Cong cu AI + LLM]
  end

  subgraph L3["Tang 3 - Runtime trong du an"]
    SKILLS[Skill da cai: SKILL.md, references, assets]
    PYS[Script Python: resolve_config, resolve_customization, render_skill, memlog]
    CFG[Cau hinh TOML 4 lop]
    SNAP[Snapshot ket xuat _bmad/render]
    ART[Tao pham: prd, spec, sprint-status, code]
  end

  subgraph L2["Tang 2 - Phan phoi"]
    CLI[bmad-cli.js]
    INST[core/installer.js]
    MANGEN[core/manifest-generator.js]
    IDEMGR[ide/manager.js + _config-driven.js]
    MODMGR[modules/*.js]
  end

  subgraph L1["Tang 1 - Nguon"]
    SRC[src/core-skills, src/bmm-skills]
    SCRIPTS[src/scripts]
    REG[bmad-modules.yaml]
    EXT[(Repo module ngoai)]
  end

  HUMAN --> AITOOL
  AITOOL --> SKILLS
  SKILLS --> PYS
  PYS --> CFG
  PYS --> SNAP
  AITOOL --> ART
  CLI --> INST
  INST --> MANGEN
  INST --> IDEMGR
  INST --> MODMGR
  SRC --> INST
  SCRIPTS --> INST
  REG --> MODMGR
  EXT --> MODMGR
  INST --> SKILLS
  INST --> CFG
```

### 2.2 Ranh giới trách nhiệm

| Ranh giới | Bên trái | Bên phải | Hợp đồng |
| --- | --- | --- | --- |
| Installer ↔ Dự án | Node.js | Hệ thống file | `_bmad/` + thư mục skill IDE + manifest |
| Skill ↔ Script | Markdown (LLM đọc) | Python | Lệnh `uv run ...` + JSON/stdout |
| Script ↔ Cấu hình | Python | TOML | `load_central_config`, `load_customization` |
| Renderer ↔ Snapshot | Python | Hệ thống file | `manifest.json` + hash SHA-256 |
| LLM ↔ Workflow | Suy luận | Markdown | Token thay thế + `[[bmad-snapshot:...]]` |

### 2.3 Vì sao LLM là bộ thực thi

Thiết kế cố ý đặt **logic nghiệp vụ trong văn xuôi có cấu trúc**, không phải trong mã. Lý do:

1. Nghiệp vụ ở đây là *phán đoán* (phạm vi này có đa mục tiêu không? kiến trúc này có phân kỳ không?) — mã không mã hóa được.
2. Văn xuôi cho phép người dùng đọc, hiểu và sửa mà không cần biết lập trình.
3. Cùng một nội dung chạy được trên ~50 công cụ AI khác nhau mà không cần adapter.

Hệ quả: mọi thứ *có thể* tất định thì **phải** đẩy xuống script (nguyên lý P1), vì LLM không đáng tin cho việc hợp nhất TOML hay tính hash.

---

## 3. Cấu trúc mã nguồn

```
BMAD-METHOD-main/
├── src/                          # TẦNG NỘI DUNG
│   ├── core-skills/              # module core — 8 skill + 6 shim
│   │   ├── module.yaml           # định nghĩa module + prompt cấu hình
│   │   ├── module-help.csv       # catalog trợ giúp của module
│   │   ├── bmad-<name>/          # một skill = một thư mục
│   │   │   ├── SKILL.md          # điểm vào (bắt buộc)
│   │   │   ├── customize.toml    # bề mặt tùy biến (tùy chọn)
│   │   │   ├── references/       # file nạp just-in-time
│   │   │   ├── assets/           # template, CSV, HTML
│   │   │   ├── types/            # biến thể (deep-recon)
│   │   │   └── scripts/          # script Python + tests/
│   │   └── v6-shims/             # chuyển tiếp ID cũ
│   ├── bmm-skills/               # module bmm
│   │   ├── module.yaml           # + khai báo roster agent
│   │   ├── agents/               # 5 persona
│   │   ├── plan/                 # skill pha 1–3
│   │   ├── ship/                 # skill pha 4
│   │   └── v6-shims/             # 13 shim
│   └── scripts/                  # TẦNG RUNTIME dùng chung
│       ├── config_utils.py       # nạp TOML + hợp nhất cấu trúc
│       ├── resolve_config.py     # CLI 4 lớp
│       ├── resolve_customization.py  # CLI 3 lớp
│       ├── render_skill.py       # bộ máy kết xuất
│       ├── memlog.py             # nhật ký chỉ-nối-thêm
│       └── tests/                # unit test (KHÔNG cài vào dự án)
├── tools/                        # TẦNG PHÂN PHỐI + công cụ dev
│   ├── installer/
│   │   ├── bmad-cli.js           # điểm vào commander
│   │   ├── commands/             # install / status / uninstall
│   │   ├── core/                 # installer, manifest-generator, paths, config
│   │   ├── ide/                  # manager + _config-driven + platform-codes
│   │   ├── modules/              # channel, version, external, custom, plugin
│   │   ├── ui.js                 # luồng prompt (~86 KB)
│   │   ├── prompts.js            # bọc @clack/prompts
│   │   └── set-overrides.js      # bản vá TOML hậu cài đặt
│   ├── validate-skills.js        # 13 quy tắc tất định
│   ├── skill-validator.md        # 26 quy tắc (tất định + suy luận)
│   ├── validate-file-refs.js     # kiểm tra tham chiếu
│   ├── validate-doc-links.js     # kiểm tra liên kết tài liệu
│   ├── validate-sidebar-order.js # kiểm tra thứ tự sidebar
│   └── build-docs.mjs            # dựng website
├── test/                         # test cấp hệ thống (Node)
├── docs/                         # nguồn tài liệu (5 ngôn ngữ)
├── website/                      # Astro + Starlight
├── web-bundles/                  # gói cho Gemini Gems / Custom GPTs
├── bmad-modules.yaml             # REGISTRY module chính thức
├── .claude-plugin/marketplace.json  # đăng ký plugin Claude Code
└── package.json                  # v6.10.0, scripts, deps
```

### 3.1 Bảng phụ thuộc runtime

| Gói | Vai trò |
| --- | --- |
| `commander` | Phân tích lệnh CLI |
| `@clack/core`, `@clack/prompts` | Prompt tương tác |
| `chalk`, `picocolors` | Màu terminal |
| `js-yaml`, `yaml` | Đọc/ghi YAML (module.yaml, manifest.yaml, platform-codes) |
| `csv-parse` | Đọc `module-help.csv`, `files-manifest.csv` |
| `glob`, `ignore` | Duyệt file, tôn trọng `.gitignore` |
| `semver` | So sánh phiên bản, phân loại patch/minor/major |
| `xml2js` | Phân tích cấu hình XML của một số IDE |
| `@kayvan/markdown-tree-parser` | Phân tích cây markdown |

---

## 4. Thiết kế tầng phân phối (Installer)

### 4.1 Sơ đồ lớp

```mermaid
classDiagram
  class BmadCLI {
    +program: Commander
    +checkForUpdate()
    +loadCommands()
  }
  class InstallCommand {
    +options[]
    +action(options)
  }
  class UI {
    +promptInstall(options) Config
  }
  class Config {
    +directory
    +modules[]
    +ides[]
    +coreConfig
    +moduleConfigs
    +channelOptions
    +setOverrides
    +build(userInput)$
  }
  class InstallPaths {
    +srcDir, projectRoot, bmadDir
    +configDir, coreDir, scriptsDir, customDir
    +create(config)$
    +manifestFile()
    +centralConfig()
    +helpCatalog()
  }
  class Installer {
    +install(config)
    +quickUpdate(config)
    +uninstall(dir)
    -_installSharedScripts()
    -_installOfficialModules()
    -_setupIdes()
    -_restoreUserFiles()
    -generateModuleConfigs()
    -mergeModuleHelpCatalogs()
  }
  class ManifestGenerator {
    +generateManifests(bmadDir, modules, files, options)
    +collectSkills()
    +collectAgentsFromModuleYaml()
    +writeMainManifest()
    +writeSkillManifest()
    +writeCentralConfig()
    +writeFilesManifest()
    +ensureCustomConfigStubs()
  }
  class IdeManager {
    +ensureInitialized()
    +setup(ide, projectDir, bmadDir)
    +setupBatch(ideList, ...)
    +cleanup(projectDir)
    +detectInstalledIdes(projectDir)
  }
  class ConfigDrivenIdeSetup {
    +setup(projectDir, bmadDir, options)
  }

  BmadCLI --> InstallCommand
  InstallCommand --> UI
  UI --> Config
  InstallCommand --> Installer
  Installer --> InstallPaths
  Installer --> ManifestGenerator
  Installer --> IdeManager
  IdeManager --> ConfigDrivenIdeSetup
```

### 4.2 Trình tự cài đặt

```mermaid
sequenceDiagram
  actor U as Nguoi dung
  participant CLI as bmad-cli.js
  participant UI as ui.js
  participant I as Installer
  participant P as InstallPaths
  participant M as ModuleManagers
  participant MG as ManifestGenerator
  participant IDE as IdeManager

  U->>CLI: npx bmad-method install
  CLI->>CLI: checkForUpdate (async, timeout 5s)
  CLI->>CLI: checkWindowsNodeFromWsl
  CLI->>UI: promptInstall(options)
  UI->>U: hoi thu muc / module / kenh / IDE / cau hinh
  U-->>UI: cau tra loi
  UI-->>CLI: Config (immutable, frozen)
  CLI->>I: install(config)
  I->>P: InstallPaths.create(config)
  P->>P: tao _bmad, _config, core, scripts, custom
  I->>I: _installSharedScripts (copy src/scripts, loai tests)
  I->>M: giai quyet + tai module (stable/next/pinned)
  M-->>I: file module
  I->>I: createModuleDirectories cho tung module
  I->>MG: generateManifests(...)
  MG->>MG: collectSkills (quet SKILL.md de quy)
  MG->>MG: collectAgentsFromModuleYaml
  MG->>MG: writeCentralConfig (config.toml + config.user.toml)
  MG->>MG: writeMainManifest / writeSkillManifest / writeFilesManifest
  MG->>MG: ensureCustomConfigStubs
  MG-->>I: manifestFiles[]
  I->>I: applySetOverrides (neu co --set)
  I->>I: mergeModuleHelpCatalogs -> bmad-help.csv
  I->>IDE: _setupIdes(config, modules, paths)
  IDE->>IDE: copy skill vao target_dir cua tung IDE
  IDE->>IDE: sinh file lenh neu co commands_target_dir
  I->>I: _cleanupSkillDirs (xoa skill khong con)
  I->>U: renderInstallSummary + post-install messages
```

### 4.3 Chiến lược bảo toàn dữ liệu người dùng khi cập nhật

```mermaid
graph TB
  A[Bat dau update] --> B[readFilesManifest: doc files-manifest.csv cu]
  B --> C[detectCustomFiles: so hash tung file]
  C --> D{File thuoc loai nao?}
  D -->|Co trong manifest, hash khop| E[File goc chua sua -> ghi de tu do]
  D -->|Co trong manifest, hash khac| F[File bi nguoi dung sua -> sao luu]
  D -->|Khong co trong manifest| G[File nguoi dung tu them -> giu nguyen]
  F --> H[_backupUserFiles]
  G --> H
  H --> I[Cai dat moi]
  I --> J[_restoreUserFiles]
  J --> K[File tu them: khoi phuc nguyen ven]
  J --> L[File bi sua: khoi phuc duoi dang .bak]
```

`_bmad/custom/**` **không bao giờ** nằm trong `files-manifest.csv`, nên installer không có lý do chạm vào — đây là bảo đảm cấu trúc, không phải quy ước.

### 4.4 Thiết kế `set-overrides.js`

Cố ý **tách rời** khỏi luồng prompt/template/schema. Lý do và đánh đổi:

| Khía cạnh | Quyết định | Hệ quả |
| --- | --- | --- |
| Thời điểm | Bản vá **sau** khi `writeCentralConfig` chạy xong | Hoạt động giống hệt nhau cho install, update, quick-update |
| Định tuyến | Tìm khóa trong `config.user.toml` trước; thấy thì vá ở đó, không thì vá `config.toml` | Khóa user-scope tự về đúng file |
| Giá trị | Ghi **nguyên văn**, không render template `result:` | Muốn `{project-root}/research` thì phải truyền đầy đủ |
| Bền vững | Khóa có khai báo trong `module.yaml` được ghi thêm vào `_bmad/<mod>/config.yaml` | Sống sót qua lần cài kế tiếp |
| Khóa không khai báo | Chỉ tồn tại ở lần cài hiện tại | Phải truyền `--set` lại, hoặc sửa tay `config.toml` |
| Kiểm tra | **Không** validate `single-select`, không từ chối khóa lạ | Người dùng chịu trách nhiệm |

---

## 5. Thiết kế tầng nội dung (Skill/Agent/Workflow)

### 5.1 Ba khuôn mẫu skill

```mermaid
graph TB
  subgraph T1["Khuon mau A - Skill don"]
    A1[SKILL.md: toan bo huong dan]
    A2[references/ nap just-in-time]
    A3[scripts/ phuc vu catalog]
    A1 --> A2
    A1 --> A3
  end

  subgraph T2["Khuon mau B - Agent persona"]
    B1[SKILL.md: 8 buoc kich hoat]
    B2[customize.toml: agent]
    B1 --> B2
    B2 --> B3[menu: dispatch sang skill khac]
  end

  subgraph T3["Khuon mau C - Workflow ket xuat"]
    C1[SKILL.md toi gian: goi render_skill.py]
    C2[workflow.md: kien truc + kich hoat]
    C3[step-NN-*.md: tung buoc]
    C4[customize.toml: workflow]
    C1 --> C2
    C2 --> C3
    C4 -.token.-> C2
  end
```

| Khuôn mẫu | Nhận biết | Ví dụ |
| --- | --- | --- |
| **A — Skill đơn** | `SKILL.md` chứa toàn bộ hướng dẫn; có thể có `customize.toml` với `[workflow]` | `bmad-help`, `bmad-review`, `bmad-brainstorming`, `bmad-customize` |
| **B — Agent persona** | `customize.toml` có mục `[agent]`; chỉ 2 file | `bmad-agent-dev`, `bmad-agent-pm`, … |
| **C — Workflow kết xuất** | `SKILL.md` chỉ ~10 dòng gọi `render_skill.py`; có `workflow.md` | `bmad-build`, `bmad-build-auto` |

### 5.2 Giao thức kích hoạt agent (8 bước)

Trích từ `src/bmm-skills/agents/bmad-agent-dev/SKILL.md` — mọi persona đều theo cùng khung này:

```mermaid
graph TB
  S1[B1: Resolve agent block<br/>resolve_customization.py --key agent] --> S1F{Script that bai?}
  S1F -->|Co| S1B[Doc tay 3 file: customize.toml -> team -> user<br/>ap dung dung quy tac hop nhat]
  S1F -->|Khong| S2
  S1B --> S2[B2: Chay activation_steps_prepend theo thu tu]
  S2 --> S3[B3: Nhap vai - Overview + role + identity<br/>+ communication_style + principles]
  S3 --> S4[B4: Nap persistent_facts<br/>file: = duong dan/glob, con lai = su that nguyen van]
  S4 --> S5[B5: Nap config tu _bmad/bmm/config.yaml<br/>user_name, ngon ngu, planning_artifacts, project_knowledge]
  S5 --> S6[B6: Chao nguoi dung, tien to bang agent.icon<br/>nhac co the goi bmad-help bat cu luc nao]
  S6 --> S7[B7: Chay activation_steps_append<br/>xac nhan moi buoc da chay]
  S7 --> S8[B8: Dispatch hoac hien menu]
  S8 --> D{Y dinh ban dau da ro?}
  D -->|Ro| DISPATCH[Bo qua menu, dispatch thang]
  D -->|Chua| MENU[Render menu danh so: Code, Description, Action<br/>DUNG va cho input]
```

Điểm thiết kế đáng chú ý:

- **Bước 1 có fallback đầy đủ**: nếu script chết, SKILL.md mô tả *chính xác* quy tắc hợp nhất để LLM tự làm. Đây là biểu hiện của P7.
- **Bước 8 tránh nghi thức xác nhận**: nếu ý định đã rõ thì dispatch thẳng; chỉ hỏi khi ≥ 2 mục thực sự gần nhau, và chỉ hỏi *một* câu.
- **Persona bám dính**: khi user gọi skill khác, persona vẫn hoạt động, không "vỡ vai".

### 5.3 Kiến trúc file-bước (Step-File Architecture)

Áp dụng cho khuôn mẫu C. Năm đặc tính:

| Đặc tính | Cơ chế |
| --- | --- |
| **Micro-file Design** | Mỗi bước tự chứa, tuân theo đúng nguyên văn |
| **Just-In-Time Loading** | Chỉ nạp file bước hiện tại |
| **Sequential Enforcement** | Hoàn tất theo thứ tự, không nhảy cóc |
| **State Tracking** | Trạng thái nằm ở frontmatter spec + biến trong bộ nhớ |
| **Append-Only Building** | Tạo phẩm dựng dần, không viết lại |

`bmad-build` có 5 bước chính + 3 file phụ trợ:

```mermaid
graph LR
  W[workflow.md] --> S1[step-01-clarify-and-route.md]
  S1 -->|one-shot| SO[step-oneshot.md]
  S1 -->|plan-code-review| S2[step-02-plan.md]
  S2 --> S3[step-03-implement.md]
  S3 --> S4[step-04-review.md]
  S4 --> S5[step-05-present.md]
  S1 -.compile.-> CE[compile-epic-context.md]
  S3 -.sync.-> SS[sync-sprint-status.md]
  S4 -.tham chieu.-> DC[references/deletion-check.md]
  S4 -.subagent.-> RP[review-prompts/edge-case-hunter.md<br/>review-prompts/verification-gap.md]
```

### 5.4 Định tuyến của `step-01-clarify-and-route.md`

Đây là bộ định tuyến trung tâm của toàn bộ pha 4:

```mermaid
graph TB
  START[Bat dau step-01] --> C1{1. Co doi so tuong minh?}
  C1 -->|spec_folder + story_id| SF[Doc stories.yaml<br/>khong co/parse loi -> HALT<br/>khong khop id -> HALT]
  SF --> SFM{Khop file stories/id-*.md?}
  SFM -->|>1 file| HALT1[HALT - khong tu chon]
  SFM -->|dung 1| SETSPEC[spec_file = file do]
  SFM -->|0 file| DERIVE[Suy slug tu title -> spec_file moi]
  C1 -->|duong dan file| C1B{Co frontmatter status hop le?}
  C1B -->|draft| E2[EARLY EXIT -> step-02-plan]
  C1B -->|ready-for-dev / in-progress| E3[EARLY EXIT -> step-03-implement]
  C1B -->|in-review| E4[EARLY EXIT -> step-04-review]
  C1B -->|done| ING[Nap lam ngu canh, KHONG resume]
  C1B -->|khong| ING
  C1 -->|khong| C2{2. Hoi thoai gan day da ro y dinh?}
  C2 -->|Co| C1B
  C2 -->|Khong| C3[3. Quet tao pham va HOI]
  C3 --> LIST[Liet ke spec dang hoat dong + HALT cho nguoi dung chon]

  SETSPEC --> SK[Story-key resolution]
  DERIVE --> SK
  ING --> INS[INSTRUCTIONS]
  LIST --> INS
  SK --> INS

  INS --> I1[1. Nap ngu canh: liet ke planning + implementation artifacts]
  I1 --> PATH{La story cua epic?}
  PATH -->|Co| A[Path A: epic story]
  PATH -->|Khong| B[Path B: freeform]
  A --> A2[Kiem tra cache epic-N-context.md hop le?]
  A2 -->|Hop le| A5[Nap cache, BO QUA tai lieu tho]
  A2 -->|Khong| A3[Bien dich qua subagent hoac noi tuyen<br/>roi VERIFY: ton tai, khong rong, dung tieu de]
  A3 --> A5
  A5 --> A6[Nap spec done co so thu tu thap hon cung epic<br/>lay Code Map, Design Notes, Change Log, task list]
  B --> B2[Quet PRD, architecture, ux, epic, brief theo lien quan]
  A6 --> I2
  B2 --> I2[2. Lam ro y dinh - hoi dang danh so, kiem tra da tra loi HET]
  I2 --> I3[3. Kiem tra VCS: cay sach? nhanh hop ly?]
  I3 --> I4{4. Kiem tra da muc tieu}
  I4 -->|Da muc tieu| SPLIT[HALT: S = tach / K = giu<br/>S -> ghi deferred-work.md]
  I4 -->|Don muc tieu| I5
  SPLIT --> I5[5. Dinh tuyen]
  I5 -->|Blast radius = 0| ONESHOT[step-oneshot.md]
  I5 -->|Con lai| PLAN[step-02-plan.md]
```

Nguyên tắc quan trọng: **khi không chắc blast radius có thực sự bằng 0 hay không, luôn chọn đường plan-code-review**.

### 5.5 Kiến trúc lens của `bmad-review`

```mermaid
graph TB
  IN[Noi dung dau vao] --> R1[1. Resolve customization<br/>workflow.lenses]
  R1 --> R2[2. Phan loai noi dung<br/>diff/file/function/document + code hay docs]
  R2 --> R3{3. Nguoi dung co chi dinh lens?}
  R3 -->|Co| SEL1[Chay DUNG cac lens do<br/>applies_to va when KHONG loc]
  R3 -->|Khong| SEL2[Chay moi lens enabled<br/>khop applies_to va when]
  SEL1 --> R4[4. Cong bo ke hoach 1 dong]
  SEL2 --> R4
  R4 --> R5[5. Chay lens doc lap - khong co after]
  R5 --> SUB{Co subagent?}
  SUB -->|Co| PAR[Spawn 1 subagent/lens, chay song song<br/>chi tra ve findings]
  SUB -->|Khong| SEQ[Chay tuan tu, xong cai nay moi sang cai khac]
  PAR --> R6[6. Chay lens phu thuoc - co after<br/>nhan findings cua lens dich]
  SEQ --> R6
  R6 --> R7[7. Gop thanh MOT mang JSON<br/>trung lap giua lens la TIN HIEU, khong dedupe]
  R7 --> OUT[Output theo output_format: json / markdown / both]
```

Thiết kế lens là **dữ liệu, không phải mã**: mỗi lens là một mục `[[workflow.lenses]]` với `code`, `name`, `applies_to`, `when`, `instruction`, `after`. Người dùng thêm lens mới bằng cách thêm bảng vào override TOML — không cần đụng đến `SKILL.md`.

Trường `instruction` rỗng ⇒ lens bị tắt. Đây là cách tắt một lens shipped mà không cần cơ chế "disable" riêng.

### 5.6 Kiến trúc shim (v6-shims)

```
_bmad/core/v6-shims/bmad-editorial-review/SKILL.md   (6 dòng)
        │
        └──> chuyển tiếp sang bmad-review với customization đã phân giải sẵn
```

Đặc điểm thiết kế:

- Shim mang **hợp đồng đầu ra riêng**: skill đích phải tôn trọng nó (ví dụ "trả về JSON thô, không có gì khác").
- Shim có thể truyền **customization đã phân giải trước** — skill đích dùng nguyên văn cho các trường được nêu tên, chỉ tự phân giải phần còn lại.
- 19 shim: 6 trong `core` (editorial-review ×3, review-adversarial-general, review-edge-case-hunter, review-verification-gap), 13 trong `bmm`.

---

## 6. Thiết kế tầng runtime (Script Python)

### 6.1 Phân lớp

```mermaid
graph TB
  CU[config_utils.py<br/>THU VIEN DUNG CHUNG]
  RC[resolve_config.py<br/>CLI]
  RCU[resolve_customization.py<br/>CLI]
  RS[render_skill.py<br/>CLI]
  ML[memlog.py<br/>CLI doc lap]

  CU --> RC
  CU --> RCU
  CU --> RS

  CU -.- F1[load_toml: nghiem ngat]
  CU -.- F2[structural_merge]
  CU -.- F3[_merge_arrays: keyed by code/id]
  CU -.- F4[load_central_config: 4 lop]
  CU -.- F5[load_customization: 3 lop]
```

`memlog.py` **cố ý không** phụ thuộc `config_utils` — nó chỉ cần Python ≥ 3.8, không cần `tomllib`, để chạy được ở môi trường tối thiểu nhất.

### 6.2 Thuật toán hợp nhất cấu trúc

```python
def structural_merge(base, override):
    # 1. Hai table  -> deep merge đệ quy
    if isinstance(base, dict) and isinstance(override, dict):
        result = dict(base)
        for key, value in override.items():
            result[key] = structural_merge(result[key], value) if key in result else value
        return result
    # 2. Hai mảng   -> _merge_arrays
    if isinstance(base, list) and isinstance(override, list):
        return _merge_arrays(base, override)
    # 3. Còn lại    -> override thắng
    return override
```

`_merge_arrays` phát hiện **khóa hợp nhất** theo thứ tự `("code", "id")`:

```mermaid
graph TB
  A[_merge_arrays base, override] --> B[_detect_keyed_merge_field]
  B --> C{Moi phan tu deu la dict<br/>VA deu co truong code?}
  C -->|Co| D[Khoa = code]
  C -->|Khong| E{Moi phan tu deu co truong id?}
  E -->|Co| F[Khoa = id]
  E -->|Khong| G[Khong co khoa -> NOI THEM don thuan]
  D --> H[Validate: gia tri khoa phai la chuoi khong rong<br/>sai -> ConfigError]
  F --> H
  H --> I[Duyet base: xay index_by_key]
  I --> J[Duyet override:<br/>khoa da co -> THAY THE tai cho<br/>khoa moi -> NOI vao cuoi]
```

Đây là điểm tinh tế nhất của hệ thống cấu hình: nó cho phép người dùng **thay một mục menu cụ thể** hoặc **thay một lens cụ thể** mà không phải chép lại toàn bộ mảng.

### 6.3 Bảng hành vi hợp nhất

| Loại dữ liệu | Ví dụ trường | Hành vi | Ý nghĩa thực tế |
| --- | --- | --- | --- |
| Scalar chuỗi | `icon`, `role`, `style_guide`, `on_complete` | Lớp sau thắng | Đổi icon, đổi style guide |
| Scalar chuỗi rỗng | `instruction = ""` | Lớp sau thắng (thành rỗng) | **Tắt** một lens |
| Table | `[agent]`, `[workflow]` | Deep merge | Chỉ ghi trường muốn đổi |
| Mảng chuỗi | `persistent_facts`, `principles`, `activation_steps_*` | Nối thêm | Bổ sung quy tắc tổ chức |
| Mảng bảng có `code` | `[[agent.menu]]`, `[[workflow.lenses]]` | Trùng `code` → thay; mới → nối | Sửa một mục menu, thêm lens |
| Mảng bảng có `id` | `review_layers` | Như trên | |

### 6.4 Thiết kế `memlog.py`

Ba bất biến bảo đảm tính đáng tin:

| # | Bất biến | Thực thi |
| --- | --- | --- |
| 1 | **Chỉ nối thêm, theo thời gian** | Không có subcommand `edit`/`delete`; mục luôn ghi ở cuối |
| 2 | **Chỉ ghi / mù** | Mỗi lệnh nguyên tử, không cần ngữ cảnh, echo trạng thái mới dạng JSON một dòng — bên gọi không đọc lại file giữa phiên |
| 3 | **Không có trạng thái vòng đời** | "Xong/blocked/tạm dừng" cũng là *sự kiện đã xảy ra* nên ghi thành mục log, không phải trường frontmatter |

Ghi nguyên tử: `temp file → write → flush → fsync → os.rename` đè lên đích. Sự cố nguồn điện không bao giờ để lại mục ghi dở.

Lần **duy nhất** file được đọc là khi resume — và bên gọi tự đọc, không qua script.

---

## 7. Thiết kế hệ thống cấu hình

### 7.1 Hai trục cấu hình

```mermaid
graph TB
  subgraph TRUC1["Truc 1 - Cau hinh TRUNG TAM (4 lop)"]
    T1[_bmad/config.toml<br/>installer sinh - nhom]
    T2[_bmad/config.user.toml<br/>installer sinh - ca nhan]
    T3[_bmad/custom/config.toml<br/>NGUOI DUNG - nhom, commit]
    T4[_bmad/custom/config.user.toml<br/>NGUOI DUNG - ca nhan, gitignore]
    T1 --> T2 --> T3 --> T4
  end

  subgraph TRUC2["Truc 2 - Tuy bien SKILL (3 lop)"]
    S1[skill/customize.toml<br/>mac dinh - DO NOT EDIT]
    S2[_bmad/custom/skill.toml<br/>NGUOI DUNG - nhom, commit]
    S3[_bmad/custom/skill.user.toml<br/>NGUOI DUNG - ca nhan, gitignore]
    S1 --> S2 --> S3
  end
```

### 7.2 Vì sao 4 lớp mà không phải 2

| Lớp | Ai sở hữu | Vòng đời | Vấn đề nó giải quyết |
| --- | --- | --- | --- |
| `config.toml` | Installer | Ghi đè mỗi lần cài | Nguồn sự thật về *câu trả lời cài đặt* cấp nhóm |
| `config.user.toml` | Installer | Ghi đè mỗi lần cài | Tách câu trả lời cá nhân (tên, ngôn ngữ, trình độ) khỏi cấu hình chung |
| `custom/config.toml` | Người dùng | Vĩnh viễn | Ghim giá trị *bất kể* câu trả lời cài đặt; thêm agent tổ chức |
| `custom/config.user.toml` | Người dùng | Vĩnh viễn | Sở thích cá nhân không muốn commit |

Hệ quả kiến trúc: **installer có thể tái sinh 2 lớp đầu bất cứ lúc nào mà không phá gì**. Đây là thứ cho phép `quick-update` chạy an toàn không cần hỏi lại.

### 7.3 Phân vùng theo scope

`writeCentralConfig` áp dụng thuật toán phân vùng:

```mermaid
graph TB
  A[moduleConfigs] --> B[Doc scope tu module.yaml cua tung module]
  B --> C{Module la core?}
  C -->|Co| D[Moi khoa deu giu]
  C -->|Khong| E[LOAI BO moi khoa trung ten voi khoa core<br/>chong o nhiem tu config.yaml legacy]
  E --> F{Biet schema cua module?}
  F -->|Co| G[Chi giu khoa duoc khai bao]
  F -->|Khong - module ngoai/marketplace| H[Giu het, coi la team]
  D --> I{scope cua khoa}
  G --> I
  H --> J[team]
  I -->|user| K[config.user.toml]
  I -->|con lai| J
```

Xử lý đặc biệt cho khối `[agents.*]`: khi một module được **bảo toàn** (ví dụ trong `quickUpdate` mà nguồn không sẵn), `module.yaml` của nó không được đọc ⇒ agent của nó không có trong `this.agents` ⇒ sẽ *biến mất* khỏi roster. Giải pháp: đọc `config.toml` cũ, trích các khối `[agents.<code>]` thuộc module bảo toàn và ghi lại nguyên văn.

### 7.4 Cấu trúc `customize.toml` chuẩn

```toml
# DO NOT EDIT -- overwritten on every update.
#
# Override files (not edited here):
#   {project-root}/_bmad/custom/<skill>.toml         (team)
#   {project-root}/_bmad/custom/<skill>.user.toml    (personal)

[workflow]                      # hoặc [agent] cho persona

activation_steps_prepend = []   # chạy TRƯỚC luồng kích hoạt
activation_steps_append  = []   # chạy SAU luồng kích hoạt

persistent_facts = [            # ngữ cảnh giữ suốt phiên
  "file:{project-root}/**/project-context.md",
]

on_complete = ""                # chỉ thị chạy sau khi xong
output_format = "both"          # json | markdown | both
report_path = ""                # rỗng = chỉ hiện trong chat

[[workflow.lenses]]             # mảng bảng — hợp nhất theo `code`
code = "adversarial"
name = "Adversarial"
applies_to = "any"              # code | docs | any
when = "always"
instruction = "Load `references/lens-adversarial.md` from the skill root and follow it."
```

Ba tiền tố có ngữ nghĩa đặc biệt trong giá trị chuỗi:

| Tiền tố | Ý nghĩa | Dùng ở |
| --- | --- | --- |
| `file:` | Đường dẫn hoặc glob — nạp **nội dung** file làm sự thật/chỉ thị | `persistent_facts`, `review_guidance`, `style_guide` |
| `skill:` | Một skill cần tham vấn | `persistent_facts` của `bmad-party-mode` |
| *(không có)* | Câu chữ nguyên văn | Mọi nơi |

---

## 8. Thiết kế bộ máy kết xuất (Render Engine)

### 8.1 Vấn đề cần giải

Workflow cần biết những giá trị chỉ tồn tại lúc chạy: đường dẫn tuyệt đối của `planning_artifacts`, ngôn ngữ giao tiếp, danh sách lens sau khi hợp nhất, đường dẫn tuyệt đối của file bước kế tiếp. Ba cách tiếp cận và lý do chọn:

| Cách | Vấn đề |
| --- | --- |
| Để LLM tự thay thế biến | Không đáng tin, dễ ảo giác đường dẫn |
| Thay thế lúc cài | Không phản ánh override thay đổi sau đó |
| **Kết xuất lúc chạy thành snapshot bất biến** ✅ | Đúng lúc, tất định, kiểm chứng được, cache được |

### 8.2 Luồng kết xuất

```mermaid
sequenceDiagram
  participant L as LLM
  participant S as SKILL.md
  participant R as render_skill.py
  participant C as config_utils
  participant FS as He thong file

  L->>S: goi skill bmad-build
  S-->>L: chay `uv run render_skill.py --project-root X --skill Y`
  L->>R: thuc thi lenh
  R->>FS: _load_sources: doc moi *.md tru SKILL.md
  FS-->>R: sources{name: content}
  R->>R: kiem tra bat buoc co workflow.md
  R->>C: load_central_config (4 lop)
  C-->>R: central
  R->>R: quet token {workflow.*} trong sources
  alt Co token customization
    R->>C: load_toml(customize.toml, required=True) -> defaults
    R->>C: load_customization (3 lop) -> customization
  end
  R->>R: _resolve_replacements: giai 4 loai token
  R->>R: tinh source_sha256 tung file
  R->>R: tinh renderer_sha256 cua chinh no
  R->>R: identity = {project_root, renderer_sha, resolved_values, source_sha}
  R->>R: generation_hash = sha256(canonical_json(identity))[:20]
  R->>R: destination = _bmad/render/<skill>/<slug>-<root12>/<gen20>
  R->>R: _render_sources: thay token, chen duong dan snapshot
  alt destination da ton tai
    R->>FS: _verify_existing: so manifest + hash tung file
    FS-->>R: khop -> tai dung / lech -> RenderError
  else chua ton tai
    R->>FS: ghi vao staging dir tam
    R->>FS: os.rename(staging, destination) - NGUYEN TU
  end
  R-->>L: "read and follow <abs>/workflow.md"
  L->>FS: doc workflow.md da ket xuat
```

### 8.3 Bốn loại token

| Token | Regex | Nguồn | Ví dụ |
| --- | --- | --- | --- |
| `{{config.<path>}}` | `\{\{config\.([A-Za-z0-9_.-]+)\}\}` | Cấu hình trung tâm, đường dẫn đầy đủ | `{{config.modules.bmm.planning_artifacts}}` |
| `{{.<key>}}` | `\{\{\.([A-Za-z0-9_]+)\}\}` | Tra cứu **ngắn**, tìm đệ quy toàn cây config | `{{.communication_language}}` |
| `{workflow.<field>}` | `\{workflow\.([A-Za-z0-9_.-]+)\}` | `customize.toml` đã hợp nhất | `{workflow.persistent_facts}` |
| `[[bmad-snapshot:<file>.md]]` | `\[\[bmad-snapshot:([A-Za-z0-9_./-]+\.md)\]\]` | Đường dẫn tuyệt đối trong snapshot | `[[bmad-snapshot:step-02-plan.md]]` |

**Quy tắc chống mơ hồ**: `{{.key}}` tìm mọi vị trí trong cây config có tên trùng và **không phải** dict/list. Nếu tìm thấy > 1 ⇒ `RenderError: ambiguous config value 'key' found at: a.b, c.d`. Không đoán.

**Quy tắc chống đệ quy**: `_render_sources` xây **một** regex gộp mọi token rồi thay thế trong *một lượt* duy nhất. Văn bản được chèn vào (đường dẫn, văn xuôi customization) **không bao giờ** được quét lại làm token. Đây là điều ngăn một `persistent_facts` chứa chuỗi `{{.user_name}}` gây thay thế ngoài ý muốn.

### 8.4 Định dạng giá trị khi chèn

`_resolve_customization_value` chọn cách render theo **kiểu của giá trị mặc định**, không phải kiểu của giá trị override:

| Kiểu mặc định | Cách render | Kết quả trong markdown |
| --- | --- | --- |
| `str` | Nguyên văn | Chuỗi |
| `list[str]` | `_format_markdown_list` | `- mục 1`\n`- mục 2`, hoặc `_None._` nếu rỗng |
| `list[dict]` (có `id`) | `_format_review_layers` | Mục `#### Name (id)` + "Run only when: …" + instruction; nếu không có layer active ⇒ chỉ thị HALT |

Nhờ vậy `workflow.md` chỉ cần viết `{workflow.persistent_facts}` và nhận về một danh sách markdown đúng định dạng — kể cả khi rỗng thì cũng có `_None._` để hướng dẫn "bỏ qua".

### 8.5 Định danh generation

```
generation_hash = sha256(canonical_json({
    "project_root":     "<đường dẫn tuyệt đối>",
    "renderer_sha256":  "<hash của chính render_skill.py>",
    "resolved_values":  { "config.core.user_name": "...", "customization.workflow.lenses": [...] },
    "source_sha256":    { "workflow.md": "...", "step-01-...md": "..." }
}))[:20]
```

Bốn thành phần này bao phủ **mọi** thứ có thể làm đầu ra khác đi:

| Thành phần | Bắt được thay đổi nào |
| --- | --- |
| `project_root` | Cùng skill chạy ở dự án khác ⇒ snapshot khác |
| `renderer_sha256` | Nâng cấp BMad làm đổi logic renderer ⇒ kết xuất lại |
| `resolved_values` | Người dùng đổi override hoặc cấu hình ⇒ kết xuất lại |
| `source_sha256` | Nâng cấp module làm đổi nội dung bước ⇒ kết xuất lại |

Ngược lại: **không có gì đổi ⇒ hash không đổi ⇒ `destination.exists()` ⇒ chỉ verify rồi tái dùng**. Đây chính là cơ chế cache, và nó miễn phí.

### 8.6 Đường dẫn snapshot

```
_bmad/render/<skill-name>/<project-slug>-<root-hash-12>/<generation-hash-20>/
                                                        ├── manifest.json
                                                        ├── workflow.md
                                                        ├── step-01-clarify-and-route.md
                                                        └── ...
```

| Đoạn | Cách tính | Vì sao cần |
| --- | --- | --- |
| `<skill-name>` | `skill_dir.name` | Tách theo skill |
| `<project-slug>` | Tên thư mục project, kebab-case, ≤ 80 ký tự, fallback `project` | Đọc được bằng mắt người |
| `<root-hash-12>` | `sha256(str(project_root))[:12]` | Phân biệt hai dự án cùng tên thư mục |
| `<generation-hash-20>` | Như 8.5 | Định danh nội dung |

---

## 9. Thiết kế tích hợp IDE

### 9.1 Kiến trúc hướng cấu hình

Toàn bộ ~50 nền tảng chia sẻ **một** class `ConfigDrivenIdeSetup`. Không có class riêng cho từng IDE. Sự khác biệt được biểu diễn hoàn toàn bằng dữ liệu trong `platform-codes.yaml`:

```yaml
<platform-code>:
  name: "Tên hiển thị"
  preferred: true|false
  suspended: "lý do chặn cài"           # tùy chọn
  installer:
    target_dir: .claude/skills           # thư mục skill trong dự án
    global_target_dir: ~/.claude/skills  # tùy chọn — cài toàn cục
    ancestor_conflict_check: true        # tùy chọn — từ chối nếu thư mục cha có file BMAD
    commands_target_dir: .github/agents   # tùy chọn — sinh file lệnh
    commands_extension: .agent.md         # tùy chọn
    commands_body_template: "LOAD the FULL {project-root}/{target_dir}/{canonicalId}/SKILL.md, ..."
    commands_filter: agents-only          # tùy chọn — chỉ persona
```

Thêm một IDE mới = thêm ~6 dòng YAML. Không sửa mã.

### 9.2 Ba nhóm đích

```mermaid
graph TB
  A[~50 nen tang] --> B[Nhom 1: chuan lien cong cu<br/>.agents/skills]
  A --> C[Nhom 2: thu muc rieng<br/>.claude/skills, .kiro/skills, .qoder/skills ...]
  A --> D[Nhom 3: chi du an, khong global<br/>Ona, Replit, Trae]
  B --> B1[Cursor, Codex, Gemini CLI, Windsurf, Warp,<br/>Roo, GitHub Copilot, OpenCode, Goose, Crush ...]
  C --> C1[Claude Code, Kiro, Junie, Factory Droid,<br/>IBM Bob, Snowflake Cortex ...]
```

Nhiều nền tảng dùng chung `target_dir` — chuẩn `.agents/skills/` đang trở thành mẫu số chung liên công cụ. Installer viết **một** lần vào thư mục dùng chung là nhiều công cụ cùng thấy.

### 9.3 Lọc `agents-only` — thiết kế đáng chú ý

GitHub Copilot chỉ nên hiện **persona agent** trong bộ chọn Custom Agents, không hiện workflow/tool. Cách phát hiện:

```
Đọc `customize.toml` nguồn của skill → có mục [agent]?
```

Vì sao **không** dùng quy ước đặt tên:

| Trường hợp | Tên | Có `[agent]`? | Đúng/Sai nếu lọc theo tên |
| --- | --- | --- | --- |
| `bmad-tea` | Không chứa `-agent-` | Có | ❌ Lọc theo tên sẽ **bỏ sót** persona |
| `bmad-agent-builder` | Chứa `-agent-` | Không (là `[workflow]`) | ❌ Lọc theo tên sẽ **thêm nhầm** |

Kết luận thiết kế: **tín hiệu cấu hình luôn thắng quy ước đặt tên**.

### 9.4 Sao chép, không symlink

| Phương án | Ưu | Nhược | Quyết định |
| --- | --- | --- | --- |
| Symlink | Cập nhật tự động, tiết kiệm đĩa | Windows cần quyền cao; nhiều IDE không theo symlink; git khó chịu | ❌ |
| **Copy** | Chạy mọi nơi, mọi công cụ; ổn định | Phải cài lại khi cập nhật; tốn đĩa | ✅ |

Kèm theo: **làm sạch đích trước khi copy** để file cũ của phiên bản trước không tồn dư, và `_cleanupSkillDirs` xóa các thư mục skill không còn trong manifest.

---

## 10. Thiết kế phiên bản và kênh phát hành

### 10.1 Hai trục độc lập

```mermaid
graph TB
  subgraph AX1["Truc 1 - Kenh module ngoai"]
    S[stable: tag semver cao nhat, loai prerelease]
    N[next: HEAD cua nhanh main]
    P[pinned: tag chi dinh]
  end
  subgraph AX2["Truc 2 - Phien ban binary installer"]
    L["npx bmad-method install (@latest)"]
    NX["npx bmad-method@next install"]
    LOC["node local-checkout/tools/installer/bmad-cli.js"]
  end
  AX2 -->|quyet dinh| CB[core + bmm<br/>DONG GOI KEM, khong co kenh rieng]
  AX1 -->|quyet dinh| EXT[bmb, cis, gds, tea, cong dong]
```

### 10.2 Thuật toán phân giải kênh

```mermaid
graph TB
  A[Bat dau phan giai module M] --> B{Co --pin M=TAG?}
  B -->|Co| C[channel=pinned, ref=TAG]
  B -->|Khong| D{Co --next=M?}
  D -->|Co| E[channel=next, ref=main HEAD]
  D -->|Khong| F{Co --channel / --all-*?}
  F -->|Co| G[Ap dung cho moi module ngoai]
  F -->|Khong| H[Doc default_channel trong bmad-modules.yaml]
  H --> I{Co khai bao?}
  I -->|Co| J[Dung gia tri do]
  I -->|Khong| K[Mac dinh cung: stable]
  C --> L[Clone/checkout]
  E --> L
  G --> L
  J --> L
  K --> L
  L --> M[Ghi manifest.yaml: name, version, channel, sha, source, repoUrl]
```

### 10.3 Phân loại nâng cấp

| Loại | Ví dụ | Mặc định tương tác | Dưới `--yes` |
| --- | --- | --- | --- |
| Patch | v1.7.0 → v1.7.1 | Y | Tự áp dụng |
| Minor | v1.7.0 → v1.8.0 | Y | Tự áp dụng |
| Major | v1.7.0 → v2.0.0 | **N** | **Đóng băng** — phải `--pin` mới nhận |

Lý do major mặc định N: *"thay đổi phá vỡ thường xuất hiện dưới dạng 'không ổn định' khi người dùng không lường trước"*. Prompt kèm URL release notes để đọc trước khi đồng ý.

### 10.4 Tái lập cài đặt (reproducibility)

Cài `stable` phân giải tag cao nhất **tại thời điểm cài** ⇒ lặp lại cùng lệnh sau 3 tháng cho kết quả khác. Quy trình tái lập đúng:

```mermaid
graph LR
  A[May A: doc _bmad/_config/manifest.yaml] --> B[Trich version cua tung module]
  B --> C[Sinh lenh voi --pin cho tung module]
  C --> D[May B: chay lenh do]
  D --> E[Byte-for-byte giong may A]
```

```bash
npx bmad-method install --yes --modules bmb,cis \
  --pin bmb=v1.7.0 --pin cis=v0.4.2 --tools claude-code
```

---

## 11. Thiết kế luồng dữ liệu

### 11.1 Luồng ngữ cảnh qua 4 pha

```mermaid
graph LR
  B[brief.md<br/>prfaq.md<br/>research.md] --> PRD[prd.md]
  PRD --> UX[DESIGN.md<br/>EXPERIENCE.md]
  PRD --> ARCH[ARCHITECTURE-SPINE.md]
  UX --> ARCH
  ARCH --> EPICS[epics + stories]
  EPICS --> SPRINT[sprint-status.yaml]
  SPRINT --> BUILD[spec-*.md]
  ARCH --> BUILD
  BUILD --> CODE[Ma nguon]
  CODE --> REVIEW[findings]
  REVIEW --> RETRO[retrospective + action items]
  RETRO -.dieu chinh.-> PRD
  PCX[AGENTS.md<br/>project-context] -.persistent_facts.-> BUILD
  PCX -.persistent_facts.-> REVIEW
```

Mỗi tài liệu **trở thành ngữ cảnh cho pha kế tiếp**: PRD nói cho kiến trúc sư biết ràng buộc nào quan trọng; kiến trúc nói cho dev agent biết theo mẫu nào; spec cho ngữ cảnh tập trung và đầy đủ để thực thi.

### 11.2 Cơ chế cache ngữ cảnh epic

Vấn đề: mỗi story trong một epic đều cần cùng bộ ngữ cảnh planning (PRD + architecture + UX + epic file). Nạp lại mỗi lần thì tốn ngữ cảnh khủng khiếp.

```mermaid
graph TB
  A[step-01: la story cua epic N] --> B{Ton tai epic-N-context.md?}
  B -->|Khong| E[Bien dich]
  B -->|Co| C{Hop le?}
  C --> C1[Khong rong?]
  C --> C2[Bat dau bang '# Epic N Context:' dung so?]
  C --> C3[Khong file nao trong planning_artifacts moi hon?]
  C1 & C2 & C3 -->|Tat ca dung| VALID[HOP LE: nap lam ngu canh chinh<br/>KHONG nap tai lieu tho]
  C1 & C2 & C3 -->|Bat ky sai| E
  E --> E1{Runtime co subagent?}
  E1 -->|Co| E2[Spawn subagent DONG BO<br/>prompt = compile-epic-context.md]
  E1 -->|Khong| E3[Doc compile-epic-context.md va tu lam noi tuyen]
  E2 --> V[VERIFY: ton tai + khong rong + dung tieu de]
  E3 --> V
  V -->|Dat| VALID
  V -->|Khong dat| HALT[HALT va bao loi]
```

Điều kiện "không file nào trong `planning_artifacts` mới hơn" là một **cache invalidation dựa trên mtime** — đơn giản, không cần hash, đủ chính xác cho mục đích này.

### 11.3 Tính liên tục giữa các story

Sau khi có ngữ cảnh epic, `step-01` quét thêm:

```mermaid
graph LR
  A[Quet implementation_artifacts] --> B[Tim spec cung epic, status=done, story_num thap hon]
  B --> C{Tim thay?}
  C -->|Co| D[Lay spec co story_num CAO NHAT duoi hien tai]
  D --> E[Trich: Code Map, Design Notes,<br/>Spec Change Log, task list]
  E --> F[Dua vao step-02 lam ngu canh lien tuc]
  C -->|Khong| G{Co spec in-review cung epic, story_num thap hon?}
  G -->|Co| H[Bao nguoi dung va HOI co nap khong]
  G -->|Khong| I[Bo qua]
```

Đây là cách hệ thống chuyển **bài học từ story trước** sang story sau mà không cần con người nhắc lại.

### 11.4 Đồng bộ trạng thái sprint

```mermaid
sequenceDiagram
  participant B as bmad-build
  participant SS as sync-sprint-status.md
  participant Y as sprint-status.yaml

  B->>B: step-01 - Story-key resolution
  B->>Y: tim key khop {epic_num}-{story_num}<br/>so sanh SO HOC 2 doan dau
  Note over Y: 1-1 khong bao gio va cham 1-10
  Y-->>B: dung 1 khop -> story_key<br/>0 hoac >1 khop -> bo trong (canh bao neu >1)
  B->>SS: buoc 03 bat dau thuc thi
  SS->>Y: story_key -> in-progress
  SS->>Y: neu la story dau tien: epic-N -> in-progress
  B->>SS: buoc 04 review xong
  SS->>Y: story_key -> review roi -> done
  SS->>Y: cap nhat last_updated
```

---

## 12. Quyết định kiến trúc (ADR)

### ADR-01 — Logic nghiệp vụ nằm trong Markdown, không trong mã

**Bối cảnh:** Cần biểu diễn quy trình phát triển phần mềm cho LLM thực thi.

**Quyết định:** Viết quy trình bằng văn xuôi có cấu trúc trong Markdown; mã chỉ lo phân phối và tính toán tất định.

**Đánh đổi:**

| Được | Mất |
| --- | --- |
| Người dùng đọc/sửa được không cần biết code | Không kiểm tra kiểu tĩnh được |
| Chạy trên mọi công cụ AI không cần adapter | Chất lượng phụ thuộc mô hình |
| Thay đổi quy trình = sửa văn bản | Cần validator riêng (26 quy tắc) để giữ chất lượng |

---

### ADR-02 — Kết xuất snapshot bất biến thay vì thay thế biến lúc chạy

**Bối cảnh:** Workflow cần giá trị chỉ biết lúc chạy.

**Quyết định:** `render_skill.py` sinh snapshot bất biến định danh bằng hash trước khi workflow bắt đầu.

**Hệ quả:**

- LLM chỉ làm việc với **đường dẫn tuyệt đối đã chốt** ⇒ không có cơ hội ảo giác đường dẫn.
- Cache miễn phí: đầu vào không đổi ⇒ hash không đổi ⇒ tái dùng.
- Kiểm chứng được: bất kỳ lúc nào cũng có thể so `manifest.json` với hash thực tế.
- Chi phí: một lần gọi `uv run` ở đầu mỗi workflow; thư mục `render/` phình theo số generation (được gitignore).

---

### ADR-03 — Nhiều lớp TOML thay vì fork nội dung

**Bối cảnh:** Người dùng cần tùy biến; nhóm cần chuẩn hóa; cá nhân cần sở thích riêng.

**Quyết định:** 4 lớp cho cấu hình trung tâm, 3 lớp cho từng skill, với quy tắc hợp nhất cấu trúc rõ ràng.

**Hệ quả:**

- Nâng cấp BMad **không** phá tùy biến (lớp mặc định bị ghi đè, lớp `custom/` không bị chạm).
- Override **thưa**: chỉ ghi trường muốn đổi, không chép cả file.
- Mảng bảng khóa theo `code`/`id` cho phép thay *một mục* trong menu/lens.
- Chi phí: người dùng phải hiểu 3 quy tắc hợp nhất (có `bmad-customize` để khỏi phải tự viết TOML).

---

### ADR-04 — Kiến trúc file-bước với nạp just-in-time

**Bối cảnh:** Workflow phức tạp nếu nạp hết vào ngữ cảnh sẽ đẩy LLM vào "context rot".

**Quyết định:** Mỗi bước một file; chỉ nạp bước hiện tại; cấm nạp nhiều bước cùng lúc.

**Hệ quả:**

- Ngân sách ngữ cảnh giữ ổn định bất kể workflow dài bao nhiêu.
- Bắt buộc trình tự: LLM không thấy bước sau nên không thể "tối ưu" bỏ bước.
- Chi phí: nhiều thao tác đọc file; trạng thái phải được ghi ra ngoài (frontmatter spec) thay vì giữ trong ngữ cảnh.

---

### ADR-05 — Catalog phục vụ qua script, không nạp nguyên khối

**Bối cảnh:** Thư viện phương pháp elicitation và kỹ thuật brainstorming có hàng trăm mục.

**Quyết định:** Script CLI phục vụ theo lát cắt: `categories` (bản đồ rẻ) → `list --category` (chỉ mục nhóm đã chọn) → `show <name>` (chi tiết mục cụ thể) → `random -n 5 --spread` (rút ngẫu nhiên đa dạng).

**Hệ quả:**

- Catalog **không bao giờ** vào ngữ cảnh nguyên khối, trừ đúng một trường hợp: người dùng bấm `[a]` để xem hết.
- Kỹ thuật do người dùng thêm (`additional_methods`) là công dân hạng nhất: truyền `--extra` là nó xuất hiện trong menu, reshuffle, và listing.

---

### ADR-06 — Shim thay vì breaking change

**Bối cảnh:** v6 gộp nhiều skill (6 skill review → 1 `bmad-review`; 3 skill research → `bmad-deep-recon`).

**Quyết định:** Giữ ID cũ dưới dạng skill shim chuyển tiếp, mang theo hợp đồng đầu ra riêng.

**Hệ quả:** Bản cài cũ, tài liệu cũ, thói quen cũ tiếp tục hoạt động. Chi phí: 19 thư mục shim phải bảo trì; mỗi shim phải nêu rõ hợp đồng đầu ra để skill đích tôn trọng.

---

### ADR-07 — Copy skill vào IDE thay vì symlink

Xem [§9.4](#94-sao-chép-không-symlink).

---

### ADR-08 — `--set` là bản vá hậu cài đặt, không tích hợp luồng prompt

**Bối cảnh:** CI cần đặt bất kỳ tùy chọn module nào không tương tác, kể cả module chưa tồn tại khi installer được viết.

**Quyết định:** `--set` chạy **sau** khi installer hoàn tất, vá thẳng vào file TOML.

**Đánh đổi:**

| Được | Mất |
| --- | --- |
| Mở rộng vô hạn cho mọi module hiện tại và tương lai | Không validate `single-select`, không từ chối khóa lạ |
| Hoạt động giống nhau cho install/update/quick-update | Giá trị ghi nguyên văn, không render template `result:` |
| Không phải sửa installer khi module thêm khóa mới | Khóa không khai báo trong `module.yaml` không sống sót lần cài sau |

---

### ADR-09 — Memlog không có trạng thái vòng đời

**Bối cảnh:** Cần biết một phiên đã xong hay chưa khi resume.

**Quyết định:** Không có cờ `complete` trong frontmatter. "Xong" là một *sự kiện đã xảy ra* ⇒ ghi thành mục log.

**Hệ quả:** Chuỗi thời gian là nguồn sự thật **duy nhất**; file không bao giờ phải mutate frontmatter; resume học trạng thái bằng cách đọc các mục cuối — đúng cách nó học mọi thứ khác.

*Lưu ý:* một vài skill (ví dụ `bmad-brainstorming`) vẫn dùng `memlog.py set --key status --value complete` để lật cờ khi wrap-up — đây là trường hợp skill chủ chọn dùng frontmatter cho mục đích riêng của nó, không phải yêu cầu của memlog.

---

## 13. Ma trận truy vết

Ánh xạ yêu cầu → thành phần thiết kế → file mã nguồn.

| Yêu cầu | Thành phần thiết kế | File hiện thực |
| --- | --- | --- |
| FR-INST-01…03 | CLI + Command loader | `tools/installer/bmad-cli.js`, `commands/install.js` |
| FR-INST-04 | Phát hiện bản cài cũ + quickUpdate | `core/existing-install.js`, `core/installer.js:1331` |
| FR-INST-05 | Phân loại nâng cấp | `modules/version-resolver.js` |
| FR-INST-06 | Gỡ cài đặt | `commands/uninstall.js`, `installer.js:1543` |
| FR-INST-08 | Kiểm tra tiền điều kiện | `core/uv-check.js`, `core/wsl-node-check.js` |
| FR-INST-10 | Bảo toàn file người dùng | `installer.js:_backupUserFiles`, `_restoreUserFiles`, `detectCustomFiles` |
| FR-MOD-01…11 | Quản lý module | `bmad-modules.yaml`, `modules/official-modules.js`, `external-manager.js`, `channel-resolver.js`, `plugin-resolver.js`, `custom-module-manager.js` |
| FR-CFG-01…03 | Cấu hình trung tâm 4 lớp | `src/scripts/config_utils.py:load_central_config`, `manifest-generator.js:writeCentralConfig` |
| FR-CFG-04…06 | Tùy biến skill 3 lớp | `config_utils.py:load_customization`, `installer.js:_installSharedScripts` (tạo `.gitignore`) |
| FR-CFG-07…08 | `--set` / `--list-options` | `tools/installer/set-overrides.js`, `list-options.js` |
| FR-SKILL-01…09 | Chuẩn skill | `tools/validate-skills.js`, `tools/skill-validator.md` |
| FR-RENDER-01…10 | Bộ máy kết xuất | `src/scripts/render_skill.py` |
| FR-IDE-01…08 | Tích hợp IDE | `ide/platform-codes.yaml`, `ide/_config-driven.js`, `ide/manager.js` |
| FR-MANIFEST-01…06 | Sinh manifest | `core/manifest-generator.js` |
| FR-MEM-01…06 | Nhật ký phiên | `src/scripts/memlog.py` |
| FR-DOC-01…04 | Tài liệu + bundle | `tools/build-docs.mjs`, `tools/bundle-web-bundles.js`, `website/` |
| NFR-DET-01…05 | Tính tất định | `render_skill.py:_publish`, `_verify_existing`, `memlog.py` |
| NFR-MAINT-03…04 | Validator | `tools/validate-skills.js`, `validate-file-refs.js` |
| Quy tắc 10.1–10.4 | Chuẩn build | `src/bmm-skills/ship/bmad-build/workflow.md`, `step-01-clarify-and-route.md` |
| Quy tắc 10.5 | Epistemics research | `src/core-skills/bmad-deep-recon/SKILL.md` |
| Quy tắc 10.6 | Bộ nhớ phiên | `src/core-skills/bmad-brainstorming/SKILL.md`, `src/scripts/memlog.py` |

---

**Tiếp theo:** [03 — Vận hành hệ thống](./03-van-hanh-he-thong.md) · Quay lại [01 — Đặc tả](./01-dac-ta-he-thong.md)
