# Báo cáo Lab: Self evolving Agentic

> Sao chép tệp này thành `report/REPORT.md` (đã làm ở Phần 0) và điền dần qua các Phần của lab. Xóa các dòng hướng dẫn dạng trích dẫn (bắt đầu bằng `>`). Văn phong kỹ thuật, ngắn gọn, mọi nhận định đi kèm số liệu hoặc bằng chứng. Trong buổi học: điền mục 1 đến 7 (bản nháp). Sau buổi học: hoàn thiện mục 8 đến 10.

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Đỗ Trung Tuyến | 2A202602427 | Toàn bộ (cài đặt harness, chạy thí nghiệm, báo cáo) |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `LAB_MODEL=openai:gpt-6-luna`; `LAB_TEMPERATURE=1` (mô hình này từ chối `temperature=0` với lỗi 400 "Only the default (1) value is supported", nên mọi lần chạy đều có tính ngẫu nhiên); `recursion_limit=60` (mặc định của runner).
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`; Windows 11, chạy trực tiếp trong WSL2 Ubuntu (Python 3.12.3, venv riêng), không dùng Docker.
- Số lần chạy tác vụ đã dùng / ngân sách:
- Commit của tag `freeze`:

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

Dự đoán chung: `skills-auto` đạt điểm trung bình cao nhất trên tác vụ đánh giá; `subagents` không hơn `baseline` nhưng tốn khoảng 3 lần token.

- H1 (subagents so với baseline): Trên tác vụ đánh giá, `subagents` có điểm **bằng** `baseline` (chênh lệch tối đa 1 check mỗi tác vụ, trong mức nhiễu) và tốn khoảng **2,5-4,5 lần token**. Căn cứ: trên tác vụ học, cả hai cùng đạt kỹ thuật 18/18 và quy ước 0/9; token trung bình 129.619 so với 40.034. Lỗi duy nhất là nhóm E (quy ước không có trong đề); thêm explorer/implementer/reviewer không tạo ra thông tin mà không tác tử nào có, và reviewer chỉ kiểm theo đề (mục 5).
- H2 (skills-auto so với baseline): `skills-auto` đạt điểm **cao hơn** `baseline` trên tác vụ đánh giá, nhưng mức tăng **nhỏ hơn** trên tác vụ học. Phần tăng chỉ đến từ các check quy ước mà tác vụ đánh giá **dùng chung** với tác vụ học cùng họ (code: type hints, `tests/test_regressions.py`, `CHANGELOG.md`; logs: `schema_version`/`generated_by`, tên service, thứ tự). Dự đoán: không tăng ở họ data (không có skill data vì skill đó bị `validate_skill` chặn), không đạt các quy ước **mới** chỉ có ở tác vụ đánh giá, và có thể có chuyển giao âm (skill sai họ được áp nhầm như ở data-learn). Căn cứ: Phần 3.4 tăng quy ước từ 0/9 lên 6/9 đúng ở hai họ có skill; SkillsBench ghi nhận skill do mô hình tự sinh trung bình không có lợi, nên lợi ích chỉ kỳ vọng ở nơi skill chứa đúng quy tắc.
- H3 (tác vụ học so với tác vụ đánh giá): Mức cải thiện của `skills-auto` so với `baseline` trên tác vụ học (Phần 3.4: 24/27 so với 18/27 check, +6 check) **lớn hơn** trên tác vụ đánh giá. Đây là dấu hiệu quá khớp: skill mã hóa đúng các quy ước đã thấy ở tác vụ học, và SkillEvolBench ghi nhận lợi ích trên tác vụ học thường không chuyển sang tác vụ mới. Dự đoán điểm tác vụ học sau đóng băng của `skills-auto` dao động ±1 check quanh Phần 3.4 do `temperature=1`.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Theo `python scripts/tour.py`, tác tử mặc định có 9 công cụ: công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; shell `execute`; và `task` để giao việc cho subagent. Chỉ `execute` cho phép chạy lệnh (shell thật, thư mục làm việc là gốc sandbox).
2. Mô tả của `task` cho biết `general-purpose` là subagent đa năng "has access to all tools as the main agent", dùng cho việc tìm kiếm hoặc tác vụ nhiều bước. Subagent **không** thấy ngữ cảnh của tác tử chính: "Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report." Vì vậy tác tử chính phải đưa đủ thông tin vào lời giao việc.
3. System prompt mặc định rỗng (`''`). Câu hướng dẫn hành vi trong mô tả `task`: "Put full detail in the prompt and state exactly what it should return". Câu trong mô tả `execute`: "Use absolute paths and avoid `cd` so the working directory stays stable". Câu này mâu thuẫn với quy ước đường dẫn tương đối của lab, nên `BASE_PROMPT`/`PATHS_NOTE` phải ghi đè bằng "every path is relative to the sandbox root and never starts with '/'".

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Kết quả `baseline` trên tác vụ học (`results/baseline/*/run.json`): data-learn 5/8 (25.022 token, 6 tool call), code-learn 7/10 (58.632 token, 22 tool call), logs-learn 6/9 (36.448 token, 6 tool call). Không lần chạy nào có `error`, và `skills_modified = false` ở cả ba.

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| data-learn | `rule_money_in_cents` | E | "RULE: money values in answer.json are integer cents (1606.67 USD is written 160667)." Tác tử ghi USD dạng số thực theo đề ("`north_q1_revenue` (number)"). |
| data-learn | `rule_meta_block` | E | "RULE: answer.json has an object `meta` = {source, rows_in, rows_used}". `answer.json` chỉ có 5 khóa của đề. |
| data-learn | `rule_clean_csv` | E | "RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents ...". Tác tử không tạo `clean.csv`; câu trả lời cuối chỉ nêu "Created `workspace/answer.json`". |
| code-learn | `rule_type_hints` | E | "RULE: every public function ... has type annotations on all parameters and on the return value." Tác tử chỉ sửa logic trong `pricing.py`, `export.py`, `report.py`. |
| code-learn | `rule_regression_tests` | E | "RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)". Vết không có `write_file` nào vào `tests/`. |
| code-learn | `rule_changelog` | E | "RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' ...". Vết cho thấy tác tử **đã đọc** `workspace/CHANGELOG.md` (có sẵn mục `## Unreleased` rỗng) nhưng không ghi gì vào đó. |
| logs-learn | `rule_service_names` | E | "RULE: service names ... are lower-case with '-' replaced by '_' (payment-service -> payment_service)". Tác tử giữ nguyên `payment-service` như ví dụ trong đề. |
| logs-learn | `rule_sorted_errors` | E | "RULE: `errors` is sorted by service, then by timestamp_utc, ascending." Đề không nêu thứ tự; phần vết hiển thị (bị cắt ở 1500 ký tự mỗi mục) không có bước sắp xếp nào. |
| logs-learn | `rule_schema_header` | E | "RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage"." Đề chỉ đưa cấu trúc `{errors, counts_by_service}`. |

Nhận xét:

- **Nhóm E chiếm 9/9 check thất bại.** Theo `python scripts/check_breakdown.py`: check kỹ thuật `18/18`, check quy ước `0/9`. Đây là **bằng chứng phủ định** cho các nhóm A-D. Tác tử có đọc đặc tả (vết của cả ba tác vụ đều có `read_file workspace/README.md`), có kiểm chứng (2 lệnh `execute` ở code-learn chạy `pytest`, 3 ở logs-learn), xử lý đúng dữ liệu bẩn (trùng lặp, `-999`, ba định dạng ngày, múi giờ), và không có báo cáo sai (nhóm F): các tệp nêu trong câu trả lời cuối đều có thật.
- Nguyên nhân gốc: cả ba đề chỉ nói "checked by Acme's review bot against the Acme ... conventions", nhưng quy ước **không có** trong đề hay workspace. Tác tử không thể tự suy ra các quy ước cụ thể như "integer cents" hay `schema_version: 2`. Một số quy ước còn trái với gợi ý trong đề (đề ghi "number" cho tiền, ví dụ dùng `payment-service`).
- **Skill có thể phòng ngừa nhóm E**, vì `detail` của tác vụ học phát biểu chính quy tắc. Curator chép các quy ước này thành skill thì tác tử áp dụng được cho tác vụ cùng loại. Rủi ro: quy ước **mới** chỉ có ở tác vụ đánh giá thì skill không thể biết, và skill có thể quá khớp vào chi tiết của tác vụ học.
- Ghi chú môi trường: lần chạy baseline đầu tiên bị hủy. Git trên Windows (`core.autocrlf=true`) checkout `tasks/` với CRLF, làm hash của `tests/test_report.py` không khớp và check kỹ thuật `tests_not_modified` trượt giả (code-learn 6/10). Sau khi checkout lại `tasks/` với LF (khớp từng byte với commit), toàn bộ baseline và subagents được chạy lại; mọi số liệu trên là của lần chạy lại.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa (tên, vai trò, lý do thiết kế) (`src/lab/subagents.py`; `build_agent` nối `PATHS_NOTE` vào prompt của từng subagent):
  - `explorer`: chỉ đọc README, quy ước, docstring, mẫu dữ liệu và báo cáo quy tắc/điểm bất thường. Thiết kế để chống nhóm lỗi A (bỏ qua đặc tả) và D (dữ liệu bẩn).
  - `implementer`: thực hiện thay đổi, ghi tệp đầu ra đúng tên/định dạng, chạy test. Prompt nhấn mạnh "quy ước trong workspace/ ghi đè thói quen" để chống nhóm E.
  - `reviewer`: kiểm tra độc lập, không sửa; tự tính lại số liệu và trả về checklist PASS/FAIL. Thiết kế để chống nhóm B (không kiểm chứng) và F (báo cáo sai).
- `subagent_calls` ở từng tác vụ và nhận xét (kể cả trường hợp bằng 0):
  - code-learn: 3 lần (`explorer` → `implementer` → `reviewer`), đúng quy trình thiết kế. Điểm 7/10.
  - data-learn: 3 lần (`explorer` → `implementer` → `reviewer`). Điểm 5/8.
  - logs-learn: 2 lần (`explorer` → `reviewer`). Tác tử chính tự viết `errors.json`, không gọi `implementer`. Điểm 6/9.
  - Tác tử chính luôn giao việc (không có trường hợp bằng 0), nhưng điểm **giống hệt baseline** ở cả ba tác vụ: kỹ thuật `18/18`, quy ước `0/9`.
- Thông tin thiếu hoặc thừa khi giao việc (nếu có giao việc):
  - Lời giao việc chép đủ các quy tắc **của đề**. Ví dụ ở code-learn, lời giao cho `implementer` liệt kê từng định dạng giá ("'(12.00)' -> Decimal('-12.00')"), `ROUND_HALF_UP`, "Do not edit existing tests".
  - Thông tin thiếu là quy ước Acme, vì chính tác tử chính cũng không có. Ở data-learn, lời giao việc còn chủ động **gạt bỏ** quy ước: "Need honor any Acme reporting conventions found, though README is only task documentation and adds none beyond above ... write valid JSON with exactly these keys". Câu này đẩy `implementer` tránh xa khối `meta`.
  - `reviewer` chỉ được yêu cầu kiểm "against the user task, function docstrings" (code-learn) và "exact key set" (data-learn), tức là kiểm chứng **theo đề**. Vì vậy reviewer xác nhận kết quả đúng, và câu trả lời cuối ghi "Independent checks confirmed the reported totals and counts", dù 3 check quy ước vẫn trượt. Đa tác tử tăng độ chắc chắn về những gì đã biết, không giúp phát hiện điều chưa biết.
- Ảnh hưởng đến token và thời gian: token trung bình 129.619 so với 40.034 của baseline (**gấp ~3,2 lần**). Theo từng tác vụ: code-learn 153.469 so với 58.632 (2,6×), data-learn 109.909 so với 25.022 (4,4×), logs-learn 125.480 so với 36.448 (3,4×). Thời gian: 249,3/132,7/126,9 giây so với 47,7/23,7/28,0 giây. Cùng điểm số nên hiệu quả điểm trên token của `subagents` thấp hơn khoảng 3 lần. Lưu ý: `tool_calls` của `subagents` (16/7/7) chỉ đếm luồng chính, không gồm việc bên trong subagent.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator, số skill bị xóa và lý do: **3 lần** (lần đầu và 2 lần chạy lại, đúng giới hạn); **6 skill bị xóa**; bộ cuối có 2 skill.
  - Lần 1 sinh `structured-data-delivery`, `repository-fix-compliance`, `log-output-contracts` (11 dòng mỗi skill). Cả 3 bị xóa vì **quá trừu tượng**: chỉ nói "Emit every required top-level field ... with the specified types and schema version", "Update required documentation in the specified format" mà không phát biểu quy tắc nào trong `detail`. Vì đề không chứa quy ước Acme, tác tử đọc skill vẫn không biết "required" là gì, nên skill không thể sửa nhóm lỗi E (9/9 lỗi ở mục 4).
  - Lần 2 (cùng prompt) sinh `tabular-data-deliverables`, `code-fix-completion`, `structured-log-triage` (15-16 dòng), cùng lỗi trừu tượng ("using the specified schema version and exact key names", "Record each fix in the required changelog section and format"). Cả 3 bị xóa. Hai lần giống nhau cho thấy nguyên nhân là **prompt** chứ không phải nhiễu: prompt mẫu yêu cầu "do not mention ... file names ... or numbers" nên mô hình né luôn cả tên và giá trị do quy ước quy định.
  - Trước lần 3, prompt của curator (`CURATOR_PROMPT` trong `src/lab/curator.py`) được sửa: nói rõ quy ước Acme phải được phát biểu cụ thể, và cho phép tên hoặc giá trị do quy ước quy định (tên tệp đầu ra, khóa JSON, header, schema version, tiêu đề), đúng như `05_skill_quality.md`. Các cấm đoán vẫn giữ: id tác vụ, tên tệp dữ liệu đầu vào, cột, hàm, kết quả tính. `test_04` vẫn đạt.
  - Lần 3 sinh 3 skill. `financial-data-output-conventions` bị `validate_skill` **từ chối** ("mentions evaluation material: orders"). Từ `orders` trùng tên một tệp của tác vụ đánh giá, nên cơ chế chống rò rỉ đã chặn nó. Do đã hết số lần chạy lại, bộ skill cuối **không có skill cho họ `data`**. Không sửa tay nội dung skill nào.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
Kết quả Phần 3.4 (`results/skills-auto-dev/`, 3 tác vụ học; `skills_sha256` = `f38a9b7c8c78...` ở cả 3 lần chạy, `skills_modified = false`): code-learn 10/10 (143.812 token), data-learn 5/8 (75.549 token), logs-learn 9/9 (34.380 token). Check quy ước đạt 6/9 so với 0/9 của baseline; 3 check còn trượt đều ở data-learn, họ không có skill. Ở data-learn tác tử đọc cả hai skill không liên quan, chép khóa `schema_version`/`generated_by: log-triage` của skill log vào `answer.json`, và câu trả lời cuối ghi "The report includes the Acme convention metadata", một khẳng định sai lệch (gần nhóm F) vì khối `meta` đúng quy ước không có (`rule_meta_block` trượt).

| `python-package-bugfix-hygiene` | Tổng quát cho mọi tác vụ sửa lỗi gói Python theo quy ước Acme: không nêu tên hàm, tệp nguồn hay lỗi cụ thể của code-learn. Chỉ nêu tên do quy ước quy định (`tests/test_regressions.py`, `CHANGELOG.md`, `## Unreleased`). | Đúng, khớp với `detail` của 3 check `rule_type_hints`, `rule_regression_tests`, `rule_changelog` ("at least 3", định dạng `- fix(<function name>): <short description>`). Không có chỉ dẫn gây hại. | 12 dòng. `description` "Use when fixing bugs in a Python package." đủ rộng để kích hoạt ở tác vụ code mới. `skills_read` = 1 ở code-learn (đọc ngay ở tool call đầu tiên): **10/10** so với baseline 7/10, cả 3 check `rule_*` đạt. Trace cho thấy tác tử tạo `workspace/tests/test_regressions.py` và sửa `workspace/CHANGELOG.md`. |
| `log-triage-output-conventions` | Tổng quát theo họ tác vụ log (ví dụ `payment-service` lấy từ chính `detail`), nhưng chỉ gồm đúng 3 quy ước đã thấy ở logs-learn. Quy ước log **mới** ở tác vụ khác sẽ không được bao phủ. | Đúng, khớp với `detail` của `rule_schema_header`, `rule_service_names`, `rule_sorted_errors`. Lưu ý: skill bảo đổi tên service sang `payment_service`, còn ví dụ trong đề dùng `payment-service`; tác tử phải chọn theo quy ước. | 11 dòng. `description` "Use when transforming application logs into structured JSON error reports." `skills_read` = 1 ở logs-learn: **9/9** so với baseline 6/9, cả 3 check `rule_*` đạt. **Chuyển giao âm** ở data-learn: tác tử (`skills_read` = 2) áp nhầm skill này vào `answer.json` ("schema_version": 2, "generated_by": "log-triage"). |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
(dán bảng ở đây)
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Nhóm đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

> Nêu ít nhất 3 hạn chế và ảnh hưởng của từng hạn chế đến kết luận (ví dụ: chỉ 3 tác vụ mỗi vai trò, mỗi cấu hình chạy một lần, nhiễu của mô hình, tác vụ do giảng viên thiết kế sẵn quy ước, chỉ một mô hình).

1.
2.
3.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự; trong WSL, venv `~/venvs/lab20`):
  1. `pytest` (29 passed, offline), `python scripts/tour.py`.
  2. `python -m lab.runner --condition baseline --tasks data-learn code-learn logs-learn`: lần đầu bị hủy do lỗi CRLF (xem mục 4); lần chạy `subagents` đầu tiên bị dừng giữa chừng vì cùng lý do.
  3. Sửa môi trường: checkout lại `tasks/` với LF (`git -c core.autocrlf=false checkout -- tasks/`), xóa `__pycache__` trong `tasks/`.
  4. `python -m lab.runner --condition baseline --tasks data-learn code-learn logs-learn` rồi `python -m lab.runner --condition subagents --tasks learn` (chạy lại, tuần tự).
  5. Ghi chú: `results/baseline/data-learn/run.json` là của một lần chạy `baseline data-learn` thứ hai bắt đầu lúc 05:15:18 UTC, chạy song song và ghi đè lần chạy lúc 05:12 (log của lần bị ghi đè: 5/8, 51.742 token). Hai lần cùng điểm 5/8 với cùng các check thất bại; báo cáo dùng số liệu trong `run.json`. Chênh lệch token giữa hai lần (25.022 so với 51.742) là một ví dụ về nhiễu ở `temperature=1`.
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
