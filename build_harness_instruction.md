## Vào dự án mới

### Nên chia thế này

| Việc | Ai làm | Vì sao |
|---|---|---|
| Đọc Section 0–3 + bảng 15 bước + exit gate | **Bạn** | Đây là phần định hướng, không cần đọc hết |
| Điền Project Profile: commands, critical flows, data dirs, constraints | **Bạn + agent đề xuất** | Agent có thể chạy lệnh để khám phá, nhưng bạn phải xác nhận |
| Quyết định scope session đầu | **Bạn** | Tránh overreach; WIP=1 |
| Tạo/sửa file theo từng bước, chạy check, viết docs draft | **Agent** | Việc cơ học, tốn thời gian |
| Chấm exit gate, đọc evidence, commit | **Bạn review, agent chạy** | Không tin “done” nếu chưa thấy output |
| Feature work sau khi harness xong | **Agent, theo AGENTS.md + feature_list** | Lúc này repo đã tự dẫn đường |

### Quy trình gợi ý cho dự án mới

**Session 0 — bạn chuẩn bị (15–30 phút):**
1. Copy `repo learn-harness-engineering` vào repo mới, ví dụ `archive/learn-harness-engineering`.
2. Đọc nhanh **Section 2.2** (15 bước), **Section 3** (Project Profile), và các **exit gate**.
3. Chọn scope session đầu: thường là **Step 0–3** (audit, `AGENTS.md`, docs map, `init.sh`) hoặc **Step 0–6** nếu repo nhỏ.
4. Viết rõ “Definition of Done của session bootstrap”: `init.sh` chạy được, `AGENTS.md` trả lời được 5 câu hỏi, có feature list, có progress/handoff, chưa làm feature.

**Session 1 — agent làm bootstrap, bạn duyệt từng gate.** Prompt kiểu:

```text
Bạn đang ở root, repo target tại <path>. Hãy đọc `archive/learn-harness-engineering/build_harness.md`,
chỉ Sections 3–5 và Steps 0–3. Không làm gì ngoài scope này.

Trước khi sửa file:
1. Chạy các lệnh hiện có (install/build/test/lint) và ghi kết quả.
2. Điền Project Profile; nếu chưa chắc thì đề xuất, không bịa.
3. Đề xuất danh sách file sẽ tạo/sửa và chờ tôi xác nhận.

Sau khi tôi xác nhận:
- Thực hiện từng step, dừng ở exit gate.
- Chạy đúng lệnh kiểm tra, ghi evidence.
- Commit riêng từng step.
- Báo lại: files changed, commands run, gate pass/fail, next step.

Không làm feature work, không refactor, không tạo CI trong session này.
```

**Các session sau:** làm Step 4–6, rồi 7–9, rồi 10–12, rồi 13–14. Mỗi session một phase, không gộp.

### Vì sao không nên “prompt agent làm hết”?

- Agent sẽ tạo một loạt file template với lệnh **không đúng repo**.
- Nó dễ đánh dấu `passing` khi chưa chạy verification.
- Nó dễ refactor trước khi có test/log — đúng cái bẫy guide muốn tránh.
- Bạn sẽ phải review một đống thay đổi trong một context đã cạn, thay vì review từng gate nhỏ.

### Sau khi bootstrap xong thì sao?

Lúc đó `AGENTS.md` chính là “guide phiên bản repo”. Bạn không cần paste lại `build_harness.md` mỗi lần. Prompt feature session chỉ cần:

```text
Đọc AGENTS.md, feature_list.json và claude-progress.md.
Chọn feature ưu tiên cao nhất chưa xong. WIP=1.
Implement, chạy verification, ghi evidence, cập nhật state, commit.
Không làm feature khác, không refactor ngoài scope.
```

Còn `build_harness.md` dùng cho:
- Setup ban đầu.
- Audit định kỳ (Section 14 + 15).
- Khi repo bị drift hoặc agent bắt đầu “quên” harness.

**Tóm lại:** bạn đọc **bản đồ và các cổng kiểm tra**; agent đọc **từng bước chi tiết** và thi công; bạn nghiệm thu evidence. Đó là cách dùng guide đúng nhất — và cũng chính là triết lý harness: con người giữ contract, agent giữ nhịp thực thi.
