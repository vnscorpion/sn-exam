# Kế hoạch website "Bài tập lớp 2" (phiên bản nạp đề online)

Cập nhật: 2026-10-09

## 0. Mục tiêu

Website có 2 phía:
- **Phía bé**: làm bài tương tác theo 3 chế độ Hàng ngày / Cuối tuần / Ôn thi học kỳ, chấm điểm ngay, thưởng sao, lưu tiến độ.
- **Phía quản trị (phụ huynh/giáo viên)**: **upload PDF/docx đề bài lên web**, hệ thống dùng AI (Claude) tự tách thành câu hỏi + đáp án, người quản trị duyệt rồi bấm "Xuất bản". 89 file hiện có là lô dữ liệu đầu tiên nạp qua chính luồng này.

Không có script chạy riêng trên máy cá nhân. Mọi xử lý nằm trong website.

**Cấu trúc điều hướng** (cố định cho mọi lớp):

```
Lớp (1, 2, 3, 4, 5...)
 └─ Môn (Toán, Tiếng Việt, sau này thêm Tiếng Anh...)
     └─ Chế độ (Hàng ngày / Cuối tuần / Ôn học kỳ)
         └─ Danh sách bài / tuần / đề  →  chọn làm  →  chấm điểm, lưu kết quả
```

Lớp 2 là dữ liệu đầu tiên. Lớp và Môn là **dữ liệu trong DB**, không viết cứng trong code: thêm lớp mới = admin bấm "Thêm lớp" rồi upload tài liệu của lớp đó, không cần lập trình lại. Mỗi bé có "lớp hiện tại"; lên lớp thì đổi, lịch sử lớp cũ vẫn giữ.

## 1. Môi trường phát triển: một máy Ubuntu, Claude Code chạy trên đó

**Quyết định (2026-10-09):** máy Windows hiện tại là máy ảo Hyper-V không bật nested virtualization và không có quyền admin, nên không chạy được Docker Desktop. Thay vì tìm cách lách, **toàn bộ phát triển chuyển sang một máy Ubuntu 24.04**, cài Claude Code trên đó. Dev và máy chủ thật cùng hệ điều hành, cùng Docker, cùng image, không còn đường rẽ Windows nào trong code.

Máy Windows chỉ còn 2 vai trò: mở trình duyệt vào web đang chạy trên Ubuntu để test, và upload 89 file gốc qua chính trang quản trị (file nằm trên OneDrive, không cần chép sang Ubuntu).

### 1.1. Máy Ubuntu cần gì
- Ubuntu 24.04 LTS, 4 CPU, **tối thiểu 8 GB RAM** (Next.js dev + Postgres + Gotenberg + Claude Code), 60 GB đĩa. Máy vật lý, máy ảo, hay VPS đều được: **Docker Engine trên Linux không cần nested virtualization** (khác Docker Desktop trên Windows), chạy được trong mọi máy ảo.
- Cùng mạng LAN với máy Windows/tablet để test, hoặc là VPS có IP công khai (khi đó mở cổng 3000 tạm thời hoặc dùng SSH tunnel).
- Có thể dùng luôn máy này làm máy chủ thật sau này nếu là VPS; nếu là máy nhà thì server thật thuê riêng.

### 1.2. Cài đặt ban đầu trên Ubuntu (chạy một lần, ~15 phút)
```bash
# 1. Gói cơ bản
sudo apt update && sudo apt install -y git curl ca-certificates poppler-utils

# 2. Docker Engine (kho chính thức của Docker, không dùng snap)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER      # đăng xuất/đăng nhập lại để có hiệu lực
docker compose version             # kiểm tra

# 3. Node.js 22 LTS qua nvm (không cần sudo, đổi phiên bản dễ)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc && nvm install 22 && node --version

# 4. Claude Code
npm install -g @anthropic-ai/claude-code
claude                              # lần đầu sẽ hỏi đăng nhập (tài khoản Claude hoặc API key)

# 5. Mã nguồn
mkdir -p ~/dev && cd ~/dev
git clone <repo> bt-lop-2 && cd bt-lop-2   # hoặc Claude Code khởi tạo dự án mới tại đây
```

Lưu ý: Claude Code đăng nhập bằng tài khoản riêng của bạn; **API key cho ứng dụng** (biến `ANTHROPIC_API_KEY` trong `.env`) là một key khác, tạo tại console.anthropic.com, chỉ dùng cho tính năng đọc đề.

### 1.3. Chạy thử local trên Ubuntu
```bash
cp .env.example .env               # điền ANTHROPIC_API_KEY, AUTH_SECRET
docker compose -f docker-compose.dev.yml up -d   # Postgres + Gotenberg, cổng mở ra host
npm install
npx prisma migrate dev
npm run dev                        # ứng dụng chạy trên host để hot-reload nhanh
```
- Trên máy Ubuntu: `http://localhost:3000`.
- Từ máy Windows hoặc tablet cùng LAN: `http://<IP Ubuntu>:3000` (xem IP bằng `ip a`). Next.js dev mặc định lắng nghe mọi địa chỉ nên không cần cấu hình thêm; nếu có `ufw` thì `sudo ufw allow 3000`.
- Dữ liệu dev nằm trong Postgres container (volume `pgdata_dev`), file upload trong `./storage`. Không dùng SQLite nữa: dev và prod cùng một DB engine, bớt một nguồn lỗi.

### 1.4. Bàn giao cho Claude Code trên Ubuntu
Phiên Claude Code mới trên Ubuntu không có lịch sử trò chuyện này. Hai file trong repo là nguồn sự thật:
- `docs/KE_HOACH.md`: chính file này (chép từ OneDrive sang, sau đó chỉ sửa trong repo).
- `CLAUDE.md`: quy ước ngắn cho Claude Code (stack, lệnh chạy, cấu trúc thư mục, quy tắc commit, chỉ dẫn "đọc docs/KE_HOACH.md trước khi làm").
Lệnh mở đầu phiên: `claude` rồi gõ "Đọc docs/KE_HOACH.md và bắt đầu giai đoạn 0".

## 2. Kiến trúc

```
┌──────────────┐  upload PDF/docx   ┌──────────────────────────────────────────┐
│ Trang quản   │ ─────────────────▶ │ Next.js (Node.js)                        │
│ trị (admin)  │ ◀───── duyệt ───── │  ├ API upload, job, duyệt, xuất bản       │
└──────────────┘                    │  ├ Worker xử lý file:                     │
                                    │  │   docx→PDF (Gotenberg)                 │
┌──────────────┐                    │  │   tách trang, render ảnh (pdftoppm)    │
│ Trang bé     │ ◀── bài đã xuất ── │  │   gọi Claude API → JSON câu hỏi        │
│ (tablet/web) │ ──── kết quả ────▶ │  │   kiểm tra đáp án số học bằng code     │
└──────────────┘                    │  │   cắt hình minh họa từ ảnh trang       │
                                    │  └ DB: Postgres (dev + prod, Docker)      │
                                    │     Lưu file: ổ đĩa (dev) / S3 (prod)     │
                                    └──────────────────────────────────────────┘
                                                      │
                                                      ▼
                                             Claude API (claude-opus-5-5)
```

**Công nghệ** (một ngôn ngữ duy nhất: TypeScript):
- Next.js 15 (App Router): vừa giao diện vừa API, một dự án, một lệnh chạy.
- Prisma ORM + Postgres 16 (container) ở cả dev lẫn prod.
- `pdf-lib` tách/gộp trang PDF; `pdftoppm` (poppler-utils) render trang thành PNG; docx → PDF qua Gotenberg (container LibreOffice chạy như dịch vụ HTTP) ở cả dev lẫn prod. Một đường code duy nhất.
- `@anthropic-ai/sdk`: gọi Claude, gửi thẳng PDF (document block), nhận JSON theo schema (structured outputs).
- Hàng đợi job: bảng `Job` trong DB + worker chạy trong tiến trình Next.js (đủ cho 1 gia đình/1 trường nhỏ). Khi lớn hơn thì chuyển BullMQ + Redis.
- Đăng nhập: NextAuth, tài khoản admin/phụ huynh bằng email + mật khẩu; bé chọn avatar + mã PIN 4 số.

## 3. Mô hình dữ liệu

```
Grade       (lớp: số, tên hiển thị, thứ tự, bật/tắt)          ← admin thêm được
Subject     (môn: mã toan|tv|..., tên, thuộc Grade, bật/tắt)   ← admin thêm được
User        (admin / parent)            Child       (tên, avatar, PIN, thuộc User, gradeId hiện tại)
Upload      (file gốc, gradeId, subjectId, loại: daily|weekend|exam, học kỳ, trạng thái, người tải)
Page        (thuộc Upload, số trang, ảnh PNG, text trích được)
Job         (thuộc Upload, loại: convert|render|parse|answer|crop, trạng thái, log lỗi, chi phí token)
Lesson      (bài/tuần/đề: tiêu đề, gradeId, subjectId, loại, học kỳ, thứ tự, trạng thái draft|published, nguồn Upload + trang)
Question    (thuộc Lesson: type, prompt, data JSON, answer JSON, points, needsReview, figureId)
Figure      (ảnh cắt: trang, bbox, đường dẫn file)
Attempt     (Child làm Lesson: bắt đầu, nộp, điểm, thời gian)
Answer      (thuộc Attempt: Question, câu trả lời, đúng/sai)
```

**Các `type` câu hỏi** (giữ nguyên như khảo sát tài liệu): `mcq`, `fill-number`, `fill-text`, `compare`, `match`, `vertical` (đặt tính), `word-problem`, `order`, `free-write`, `handwriting`, `offline` (chỉ hiện đề, làm trên giấy).

Ví dụ `Question.data` + `Question.answer` cho `word-problem`:
```json
{ "text": "Đội văn nghệ của lớp 2B có 8 bạn nam và 3 bạn nữ. Hỏi ... bao nhiêu bạn?",
  "summary": ["Nam: 8 bạn", "Nữ: 3 bạn", "Tất cả: ... bạn?"],
  "sentenceChoices": ["Đội văn nghệ có tất cả số bạn là:", "Số bạn nam nhiều hơn là:", "Số bạn còn lại là:"] }
{ "sentence": 0, "expression": "8 + 3", "result": 11, "unit": "bạn" }
```

## 4. Quy trình nạp đề online (pipeline)

### Bước 1. Upload
Admin vào `/admin/upload`, kéo thả 1 hoặc nhiều file, chọn: **lớp**, môn, loại (Hàng ngày / Cuối tuần / Ôn thi), học kỳ (1 / 2), tên bộ tài liệu. Lớp và môn lấy từ bảng `Grade`/`Subject`; nút "Thêm lớp mới" ngay trong form. Prompt gửi AI được ghép thêm lớp + môn để model hiểu đúng chương trình (vd lớp 3 có nhân chia bảng 6–9, lớp 4 có phân số). Giới hạn 200 MB/file. File lưu vào `storage/uploads/<uploadId>/`.

### Bước 2. Chuẩn hóa (job `convert`)
- docx → PDF bằng `soffice --headless --convert-to pdf`.
- PDF > 30 MB hoặc > 100 trang: tách thành các phần ≤ 25 MB, ≤ 50 trang bằng `pdf-lib` (giới hạn API là 32 MB và 600 trang mỗi lần gửi; file cuối tuần 84 MB/186 trang sẽ thành ~4 phần).

### Bước 3. Render (job `render`)
Mỗi trang → PNG 1200px (`pdfjs-dist` + canvas) → `storage/pages/<uploadId>/p-001.png`. Dùng để (a) hiện cho admin duyệt cạnh kết quả AI, (b) cắt hình minh họa, (c) gửi cho AI nếu PDF là ảnh scan chất lượng thấp.

### Bước 4. Nhận diện ranh giới bài (job `parse`, lần 1)
Gửi cả phần PDF cho Claude với yêu cầu: liệt kê các đơn vị bài (Bài N / Tuần N / Đề số N) kèm trang bắt đầu–kết thúc. Kết quả: danh sách `Lesson` nháp.

### Bước 5. Tách câu hỏi (job `parse`, lần 2, chạy song song theo bài)
Với mỗi bài, gửi đúng các trang của bài đó (PDF cắt bằng `pdf-lib`) + prompt hệ thống mô tả 11 `type` + schema JSON. Model trả về:
- Danh sách câu hỏi với `type`, `prompt`, `data`.
- Với câu có hình: `figure: { page, bbox: [x0,y0,x1,y1] }` (tọa độ 0–1) để server cắt từ ảnh trang. Hình quy luật (đồng hồ, tia số, bảng) trả về `figure: { kind: "clock", h, m }` để vẽ SVG thay vì cắt ảnh.
- Câu không chuyển được (vẽ hình, cắt dán, thực hành cân đo) → `type: "offline"`.
Dùng structured outputs (`output_config.format`) để JSON luôn đúng schema. Model: `claude-opus-5-5`, adaptive thinking, streaming.

### Bước 6. Đáp án (job `answer`)
- Model giải và ghi `answer` ngay trong bước 5.
- Server kiểm tra lại bằng code mọi câu số học (`fill-number`, `vertical`, `compare`, `word-problem`, `order`): tính lại, lệch thì gắn `needsReview`.
- Đề có sẵn đáp án trong file (ôn thi HK2): model được yêu cầu đối chiếu với trang đáp án.
- Tiếng Việt (đọc hiểu, điền từ) mặc định `needsReview: true` cho đến khi admin duyệt.

### Bước 7. Cắt hình (job `crop`)
Với mỗi `figure.bbox`: cắt từ PNG trang → `storage/figures/<lessonId>/<questionId>.png`. Admin có thể kéo lại khung cắt trong màn hình duyệt.

### Bước 8. Duyệt và xuất bản (`/admin/review/<lessonId>`)
Màn hình 2 cột: trái là ảnh trang gốc, phải là từng câu hỏi đã tách (hiện đúng như bé sẽ thấy, có đáp án). Admin sửa text, đổi `type`, sửa đáp án, chỉnh khung hình, xóa/thêm câu, bấm "Chạy lại AI cho câu này" nếu cần. Câu `needsReview` được tô vàng. Bấm **Xuất bản** → `Lesson.status = published`, bé thấy ngay.

### Bước 9. Theo dõi
Trang `/admin/jobs`: tiến độ từng file, lỗi, số token và chi phí ước tính mỗi lần gọi (lấy từ `usage` trong phản hồi API).

### Nhà cung cấp AI: không khóa vào một hãng
Bước 4–6 gọi qua interface `AIProvider` (`parseLessons`, `solve`). Chọn bằng biến môi trường:
```
AI_PROVIDER=anthropic | openai | gemini
AI_MODEL=claude-opus-5-5          # hoặc claude-haiku-5-5, gemini-..., gpt-...
```
- Làm adapter Anthropic trước, Gemini thứ hai (rẻ, đọc PDF và tọa độ hình tốt). OpenAI thêm sau nếu cần.
- Màn hình duyệt có nút "Chạy lại bằng nhà cung cấp khác" để so sánh kết quả trên cùng một bài.
- Chiến lược tiết kiệm (bạn quyết định): lô nạp hàng loạt dùng model rẻ (Haiku 5.5 hoặc Gemini), bài nào bước kiểm tra đáp án bằng code báo lệch thì chạy lại bằng Opus.
- Kiểm tra đáp án bằng code và màn hình duyệt áp dụng cho mọi nhà cung cấp, nên chất lượng cuối không phụ thuộc hãng.

### Chi phí AI (ước tính, giá Opus 5.5: 4 USD/1M token vào, 20 USD/1M token ra)
| Việc | Ước tính |
|---|---|
| 1 phiếu Toán 3 trang | ~6k token vào + ~3k token ra ≈ 0,09 USD |
| 1 đề thi 4 trang | ≈ 0,12 USD |
| Toàn bộ 89 file (~1.100 trang, ~450 bài) | ≈ 40–60 USD, kể cả chạy lại |
Dùng Batch API (giảm 50%) cho lô nạp hàng loạt ban đầu; upload lẻ sau này chạy ngay để admin duyệt trong vài phút.

## 5. Phía bé

### Trang chủ
Chọn avatar → nhập PIN → vào thẳng lớp hiện tại của bé (có nút đổi lớp cho phụ huynh). Trong lớp: chọn môn (Toán / Tiếng Việt) → chọn chế độ (Hàng ngày / Cuối tuần / Ôn thi) → danh sách bài. Đường dẫn dạng `/lop-2/toan/hang-ngay`, `/lop-2/tieng-viet/on-thi/de-03`. Trang chủ cũng có "Bài hôm nay" (1 Toán + 1 TV theo tiến độ), chuỗi ngày học và số sao.

### Hàng ngày
Danh sách bài theo học kỳ, làm từng câu, phản hồi ngay (đúng: âm thanh + sao; sai: thử lại 1 lần rồi hiện đáp án). Cuối bài: tổng kết, 1–3 sao, "Làm lại câu sai".

### Cuối tuần
Phiếu theo tuần, dài hơn, đồng hồ đếm thời gian, điểm cuối phiếu, so với tuần trước.

### Ôn thi
Chọn đề, đếm ngược 40 phút, không báo đúng/sai từng câu, nộp → điểm /10 theo điểm từng câu, xem lại câu sai. "Đề ngẫu nhiên": trộn câu từ ngân hàng theo ma trận 6 trắc nghiệm + 4 tự luận.

### Trải nghiệm cho bé 7 tuổi
Chữ ≥ 20px, nút ≥ 56px, bàn phím số riêng trên màn hình, nút loa đọc đề (Web Speech API tiếng Việt), PWA để thêm vào màn hình chính tablet.

### Phụ huynh (`/parent`)
Tiến độ từng bé, điểm theo thời gian, dạng bài hay sai, bài viết đoạn chờ xem. Khóa bằng mật khẩu tài khoản.

## 6. Cấu trúc dự án

```
bt-lop-2/                      (thư mục mới, nên đặt ngoài OneDrive, vd C:\dev\bt-lop-2)
├── prisma/schema.prisma
├── src/app/
│   ├── (kid)/                 trang bé: /, /[grade]/[subject]/[mode], /[grade]/[subject]/[mode]/[lessonId]
│   ├── parent/
│   ├── admin/                 grades (lớp, môn), upload, jobs, review/[lessonId], lessons
│   └── api/                   upload, jobs, lessons, attempts, auth
├── src/lib/
│   ├── ingest/                convert.ts, render.ts, split.ts, parse.ts, answer-check.ts, crop.ts
│   ├── ai/                    client.ts, prompts/, schemas/ (zod)
│   ├── queue/                 worker.ts
│   └── grading/               chấm từng type
├── src/components/qtypes/     11 component câu hỏi
├── storage/                   uploads, pages, figures (gitignore)
├── docker-compose.yml         app + postgres + libreoffice (cho deploy)
└── .env                       DATABASE_URL, ANTHROPIC_API_KEY, AUTH_SECRET
```

## 7. Lộ trình

| Giai đoạn | Việc | Kết quả kiểm tra được | Thời gian |
|---|---|---|---|
| 0 | Tạo dự án Next.js + Prisma + SQLite, đăng nhập admin, schema DB | `npm run dev` chạy, đăng nhập được | 1–2 ngày |
| 1 | Pipeline nạp đề: upload → convert → render → parse → answer → crop → màn hình duyệt → xuất bản. Thử với "Bài 7. Phép cộng qua 10" và 1 đề ôn thi HK2 | Upload 1 PDF, 2–3 phút sau duyệt được câu hỏi | 1,5 tuần |
| 2 | Phía bé: 3 chế độ, 11 dạng câu hỏi, chấm điểm, sao, tiến độ, âm thanh, đọc đề | Bé làm bài 7 trên tablet | 1,5 tuần |
| 3 | Nạp 89 file hiện có qua Batch API, duyệt. Claude Code duyệt vòng 1 (so ảnh gốc với câu hỏi), bạn duyệt vòng 2 phần Tiếng Việt | Toàn bộ 3 chế độ có dữ liệu | 2–3 tuần (phần lớn là thời gian duyệt) |
| 4 | Phụ huynh, nhiều bé, PWA, Docker, deploy (VPS hoặc Railway), backup DB | Dùng chính thức | 1 tuần |

## 8. Triển khai máy chủ

### 8.1. Chọn hệ điều hành: **Ubuntu Server 24.04 LTS**
| | Ubuntu 24.04 LTS | Debian 12 | AlmaLinux 9 |
|---|---|---|---|
| Docker | gói chính thức, cài 1 lệnh | gói chính thức | gói chính thức |
| LibreOffice / poppler / fonts | đầy đủ trong apt, phiên bản mới | đầy đủ, hơi cũ hơn | phải thêm EPEL, LibreOffice thường thiếu gói phụ |
| Hỗ trợ đến | 2029 | 2028 | 2032 |
| Khuyến nghị | **Chọn** | Thay thế tốt nếu bạn quen Debian | Chỉ khi hạ tầng công ty chuẩn hóa RHEL |

Vì toàn bộ ứng dụng chạy trong Docker, OS chủ chỉ cần Docker và SSH. Chọn Ubuntu vì tài liệu nhiều, image gốc Debian-based nên dev và prod cùng họ.

### 8.2. Docker hay cài trần: **Docker Compose**, không cài trần
Lý do:
- LibreOffice kéo theo ~400 MB gói và font; đóng trong image thì OS chủ sạch, nâng cấp/rollback bằng cách đổi tag image.
- Cùng một image chạy trên máy dev Ubuntu và máy chủ Ubuntu, hết lỗi "máy tôi chạy được".
- Postgres, Caddy, worker tách container, giới hạn RAM/CPU từng cái được.
Chỉ cài trần khi VPS dưới 1 GB RAM, lúc đó LibreOffice trong container có thể bị OOM khi chuyển file docx lớn.

### 8.3. Cách đóng gói LibreOffice: dùng **Gotenberg** (LibreOffice chạy như một dịch vụ HTTP)
Thay vì nhét LibreOffice vào image ứng dụng, chạy container `gotenberg/gotenberg:8` bên cạnh. Ứng dụng gửi docx qua HTTP `POST /forms/libreoffice/convert`, nhận về PDF.
- Image chính thức, đã kèm LibreOffice + font, tự quản tiến trình `soffice` (vốn hay treo), có timeout.
- Image ứng dụng nhỏ, build nhanh.
- Dev và prod đều dùng Gotenberg, chỉ khác URL trong `.env` (`GOTENBERG_URL=http://localhost:3001` khi dev, `http://gotenberg:3000` trong compose prod). Không còn adapter `soffice` cài trần.

Render trang PDF → PNG: dùng `pdftoppm` (gói `poppler-utils`, ~30 MB) cài trong image ứng dụng. Nhanh, ổn định, không cần module native của Node.

Font tiếng Việt: thêm `fonts-noto-core`, `fonts-liberation` (thay thế Times New Roman đúng khổ chữ), `fonts-dejavu` vào image. Gotenberg đã có sẵn.

### 8.4. Cấu hình máy chủ
- VPS: 2 vCPU, 4 GB RAM, 40 GB SSD (chuyển docx và render trang có lúc dùng 1–1,5 GB). Đủ cho vài trăm bé dùng đồng thời.
- Mở cổng 22, 80, 443. Tên miền trỏ về IP.

### 8.5. Hai file compose: dev và prod

`docker-compose.dev.yml` (máy dev Ubuntu: chỉ dịch vụ phụ, ứng dụng chạy bằng `npm run dev` trên host):
```yaml
services:
  db:
    image: postgres:16
    ports: ["5432:5432"]
    environment: { POSTGRES_USER: bt, POSTGRES_PASSWORD: bt, POSTGRES_DB: bt_dev }
    volumes: [pgdata_dev:/var/lib/postgresql/data]
  gotenberg:
    image: gotenberg/gotenberg:8
    ports: ["3001:3000"]
    command: ["gotenberg", "--libreoffice-auto-start=true", "--api-timeout=120s"]
volumes: { pgdata_dev: {} }
```

`docker-compose.yml` (máy chủ thật: tất cả trong container):
```yaml
services:
  app:            # Next.js + worker, image build từ Dockerfile
    build: .
    env_file: .env
    volumes: [storage:/app/storage]
    depends_on: [db, gotenberg]
    deploy: { resources: { limits: { memory: 1500M } } }
  db:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]
    env_file: .env
  gotenberg:
    image: gotenberg/gotenberg:8
    command: ["gotenberg", "--libreoffice-auto-start=true", "--api-timeout=120s"]
    deploy: { resources: { limits: { memory: 1000M } } }
  caddy:          # HTTPS tự động (Let's Encrypt)
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes: [./Caddyfile:/etc/caddy/Caddyfile, caddy_data:/data]
volumes: { storage: {}, pgdata: {}, caddy_data: {} }
```

Dockerfile ứng dụng (phác thảo):
```dockerfile
FROM node:22-bookworm-slim AS build
WORKDIR /app
COPY package*.json ./ && RUN npm ci
COPY . . && RUN npx prisma generate && npm run build

FROM node:22-bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils fonts-noto-core fonts-liberation fonts-dejavu \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/prisma ./prisma
CMD ["node", "server.js"]
```

### 8.6. Quy trình đưa lên máy chủ
1. Trên server: cài Docker giống mục 1.2 (bước 1–2), clone repo, tạo `.env` (DATABASE_URL, ANTHROPIC_API_KEY, AUTH_SECRET, GOTENBERG_URL).
2. `docker compose up -d --build`, rồi `docker compose exec app npx prisma migrate deploy`.
3. Cập nhật sau này: `git pull && docker compose up -d --build` (hoặc GitHub Actions build image, server chỉ `docker compose pull`).
4. Backup hằng đêm: `pg_dump` + nén thư mục `storage/` → đẩy lên S3/Backblaze bằng `rclone`.

### 8.7. Bảo mật và vận hành
- API key chỉ nằm trong `.env` trên server, không bao giờ gửi xuống trình duyệt.
- Upload chỉ cho tài khoản admin; kiểm tra MIME, giới hạn dung lượng, quét số trang trước khi gửi AI.
- Mỗi lần gọi AI ghi lại token + chi phí; đặt trần chi phí/ngày trong `.env`.
- Caddy tự gia hạn HTTPS; `ufw` chỉ mở 22/80/443; fail2ban cho SSH.
- Bản quyền: dùng nội bộ; nếu mở công khai cần xin phép đơn vị phát hành tài liệu.

## 9. Phần việc của bạn
1. Chuẩn bị máy Ubuntu 24.04 (≥ 8 GB RAM) và chạy các lệnh ở mục 1.2: Docker, Node.js, Claude Code. Kiểm tra bằng `docker compose version`, `node --version`, `claude --version`.
2. Tạo repo git (GitHub/GitLab riêng tư) hoặc để Claude Code khởi tạo tại `~/dev/bt-lop-2`.
3. Chép file này vào repo thành `docs/KE_HOACH.md`.
4. Tạo API key Anthropic cho ứng dụng, nạp tiền, điền vào `.env` trên máy Ubuntu (không dán vào chat, không đưa vào git).
5. Mở phiên Claude Code trên Ubuntu: "Đọc docs/KE_HOACH.md và bắt đầu giai đoạn 0".
6. Khi có bản giai đoạn 1: từ máy Windows mở `http://<IP Ubuntu>:3000/admin`, upload thử file từ OneDrive, duyệt, phản hồi.
7. Duyệt đáp án Tiếng Việt ở giai đoạn 3.
