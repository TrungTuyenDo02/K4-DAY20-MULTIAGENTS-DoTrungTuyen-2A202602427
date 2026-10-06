# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Đỗ Trung Tuyến | 2A202602427 | Toàn bộ (cài đặt harness, chạy thí nghiệm, báo cáo) |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `LAB_MODEL=openai:gpt-6-luna`; `LAB_TEMPERATURE=1` (mô hình này từ chối `temperature=0` với lỗi 400 "Only the default (1) value is supported", nên mọi lần chạy đều có tính ngẫu nhiên); `recursion_limit=60` (mặc định của runner).
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`; Windows 11, chạy trực tiếp trong WSL2 Ubuntu (Python 3.12.3, venv riêng), không dùng Docker.
- Số lần chạy tác vụ đã dùng / ngân sách: 21 lần chạy được lưu trong `results/` (baseline 6, subagents 6, skills-auto 6, skills-auto-dev 3), tổng 1.700.368 token; cộng thêm 3 lần chạy baseline bị bỏ do lỗi CRLF, 1 lần chạy subagents bị dừng giữa chừng, 1 lần baseline data-learn song song bị ghi đè, và 3 lần gọi curator. Không có ngân sách cố định.
- Commit của tag `freeze`: `110baa9a4beab6395255b6f3efeb3669d2be23f4` (2026-10-06T12:33:24+07:00); commit `hypotheses`: `ed16cbd`.

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

Kết quả Phần 3.4 (`results/skills-auto-dev/`, 3 tác vụ học; `skills_sha256` = `f38a9b7c8c78...` ở cả 3 lần chạy, `skills_modified = false`): code-learn 10/10 (143.812 token), data-learn 5/8 (75.549 token), logs-learn 9/9 (34.380 token). Check quy ước đạt 6/9 so với 0/9 của baseline; 3 check còn trượt đều ở data-learn, họ không có skill. Ở data-learn tác tử đọc cả hai skill không liên quan, chép khóa `schema_version`/`generated_by: log-triage` của skill log vào `answer.json`, và câu trả lời cuối ghi "The report includes the Acme convention metadata", một khẳng định sai lệch (gần nhóm F) vì khối `meta` đúng quy ước không có (`rule_meta_block` trượt).

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `python-package-bugfix-hygiene` | Tổng quát cho mọi tác vụ sửa lỗi gói Python theo quy ước Acme: không nêu tên hàm, tệp nguồn hay lỗi cụ thể của code-learn. Chỉ nêu tên do quy ước quy định (`tests/test_regressions.py`, `CHANGELOG.md`, `## Unreleased`). | Đúng, khớp với `detail` của 3 check `rule_type_hints`, `rule_regression_tests`, `rule_changelog` ("at least 3", định dạng `- fix(<function name>): <short description>`). Không có chỉ dẫn gây hại. | 12 dòng. `description` "Use when fixing bugs in a Python package." đủ rộng để kích hoạt ở tác vụ code mới. `skills_read` = 1 ở code-learn (đọc ngay ở tool call đầu tiên): **10/10** so với baseline 7/10, cả 3 check `rule_*` đạt. Trace cho thấy tác tử tạo `workspace/tests/test_regressions.py` và sửa `workspace/CHANGELOG.md`. |
| `log-triage-output-conventions` | Tổng quát theo họ tác vụ log (ví dụ `payment-service` lấy từ chính `detail`), nhưng chỉ gồm đúng 3 quy ước đã thấy ở logs-learn. Quy ước log **mới** ở tác vụ khác sẽ không được bao phủ. | Đúng, khớp với `detail` của `rule_schema_header`, `rule_service_names`, `rule_sorted_errors`. Lưu ý: skill bảo đổi tên service sang `payment_service`, còn ví dụ trong đề dùng `payment-service`; tác tử phải chọn theo quy ước. | 11 dòng. `description` "Use when transforming application logs into structured JSON error reports." `skills_read` = 1 ở logs-learn: **9/9** so với baseline 6/9, cả 3 check `rule_*` đạt. **Chuyển giao âm** ở data-learn: tác tử (`skills_read` = 2) áp nhầm skill này vào `answer.json` ("schema_version": 2, "generated_by": "log-triage"). |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

`python -m lab.compare > report/table.md`:

```text
| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 7/10 | 7/10 | 10/10 |
| data-learn | 5/8 | 5/8 | 5/8 |
| logs-learn | 6/9 | 6/9 | 9/9 |
| code-eval | 7/11 | 7/11 | 10/11 |
| data-eval | 5/9 | 5/9 | 5/9 |
| logs-eval | 6/10 | 6/10 | 9/10 |
| **Mean score - learning tasks** | 0.66 | 0.66 | 0.88 |
| **Mean score - evaluation tasks** | 0.60 | 0.60 | 0.79 |
| **Mean tokens per run** | 45,552 | 143,086 | 52,466 |
| **Runs that read a skill** | 0/6 | 0/6 | 6/6 |
```

`python scripts/check_breakdown.py`:

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval     18/18         0/12          51,070      0/3
baseline      learn    18/18         0/9           40,034      0/3
subagents     eval     18/18         0/12         156,553      0/3
subagents     learn    18/18         0/9          129,619      0/3
skills-auto   eval     18/18         6/12          54,917      3/3
skills-auto   learn    18/18         6/9           50,015      3/3
```

Check thất bại trên tác vụ đánh giá (`detail` của tác vụ đánh giá để trống; chỉ có tên check):

| Tác vụ | baseline và subagents (giống nhau) | skills-auto |
|---|---|---|
| code-eval | `rule_type_hints`, `rule_regression_tests`, `rule_changelog`, `rule_version_bump` | `rule_version_bump` |
| data-eval | `rule_money_in_cents`, `rule_meta_block`, `rule_clean_csv`, `rule_sorted_keys_format` | giống baseline |
| logs-eval | `rule_service_names`, `rule_sorted_errors`, `rule_schema_header`, `rule_source_line` | `rule_source_line` |

- Không lần chạy nào có `error`; `skills_modified = false` ở cả 21 lần chạy được lưu.
- `python scripts/verify_freeze.py` (trong WSL): `checked 6 runs of skill conditions: OK`. Hai lần kiểm tra đầu báo FAIL do môi trường, không phải do skill:
  1. Khi chạy bằng Python Windows, `hash_skills` băm đường dẫn với `\` thay vì `/`, nên không khớp hash do WSL ghi.
  2. `skills/auto/README.md` bị Windows checkout với CRLF, nên git trong WSL thấy khác tag. README được checkout lại đúng bytes của commit (không đổi nội dung skill), sau đó kiểm tra đạt `OK`.

## 8. Phân tích

1. **Học và đánh giá.**
   - `skills-auto` cải thiện cả hai: tác vụ học 0,664 → 0,875 (+0,21; 18/27 → 24/27 check), tác vụ đánh giá 0,597 → 0,788 (+0,19; 18/30 → 24/30 check).
   - `subagents` không cải thiện ở đâu: điểm giống `baseline` đến từng check trên cả 6 tác vụ (H1 được ủng hộ).
   - Không có điều kiện nào cải thiện tác vụ học mà không cải thiện tác vụ đánh giá. Mức tăng tuyệt đối bằng nhau (+6 check), nhưng tỉ lệ check quy ước đạt giảm từ 6/9 (67%) xuống 6/12 (50%). Phần chênh nằm hoàn toàn ở 3 quy ước **mới** của tác vụ đánh giá. Vì vậy H3 chỉ được ủng hộ một phần: có giới hạn chuyển giao, nhưng không có quá khớp theo nghĩa "quy ước đã học không dùng được ở tác vụ mới". H2 được ủng hộ: `skills-auto` cao nhất, lợi ích chỉ ở quy ước dùng chung của họ code và logs, không có ở họ data.
2. **Kỹ thuật và quy ước.**
   - Check kỹ thuật đạt 18/18 ở mọi điều kiện và vai trò, nên skill không có chỗ giúp. Toàn bộ lợi ích nằm ở check `rule_`.
   - Trên tác vụ đánh giá, các quy ước **dùng chung** với tác vụ học cùng họ đạt 6/6 ở code và logs (`rule_type_hints`, `rule_regression_tests`, `rule_changelog`, `rule_service_names`, `rule_sorted_errors`, `rule_schema_header`).
   - Quy ước **mới** (`rule_version_bump`, `rule_sorted_keys_format`, `rule_source_line`) đạt 0/3 ở mọi điều kiện. Lý do: curator chỉ thấy phản hồi của tác vụ học, nên skill không thể chứa một quy tắc chưa từng xuất hiện, và đề không nêu chúng.
   - Họ data đạt 0/4 vì bộ skill không có skill data (skill đó bị `validate_skill` chặn, mục 6).
3. **Một check được giúp, một check không.**
   - *Được giúp:* `rule_changelog` và `rule_regression_tests` ở code-eval. Tác tử đọc `skills/python-package-bugfix-hygiene/SKILL.md` (`skills_read` = 1), rồi vết có `write_file`/`edit_file` vào `workspace/tests/test_regressions.py` và `workspace/CHANGELOG.md`. Vết `baseline` code-eval chỉ sửa ba tệp nguồn trong gói và không động tới hai tệp này.
   - *Không được giúp, vì skill thiếu:* `rule_source_line` ở logs-eval. Tác tử đọc `log-triage-output-conventions` và làm đủ 3 quy tắc trong đó (9/10), nhưng skill không có quy tắc này.
   - *Không được giúp, vì skill sai họ:* ở data-eval, tác tử chỉ đọc skill log (`skills_read` = 1) và ghi `"generated_by": "log-triage"` vào `answer.json`, một chuyển giao âm. Hiện tượng này cũng đã thấy ở data-learn Phần 3.4. Điểm không giảm vì khóa thừa không bị chấm, nhưng đây là bằng chứng skill có thể được áp nhầm khi `description` của skill khác họ vẫn "nghe hợp lý".
4. **Chi phí.**
   - Token trung bình mỗi lần chạy: `baseline` 45.552, `skills-auto` 52.466 (+15%), `subagents` 143.086 (**3,1×**).
   - Điểm trung bình (học và đánh giá) trên 100k token: `skills-auto` 0,832/52.466 ≈ **1,58**; `baseline` 0,631/45.552 ≈ 1,38; `subagents` 0,631/143.086 ≈ 0,44.
   - Đa tác tử **không đáng** chi phí trong thí nghiệm này: gấp ~3 lần token và 2-11 lần thời gian (ví dụ data-eval 204,2 s so với 18,8 s) mà không thêm check nào. Lỗi duy nhất là thiếu thông tin (quy ước không có trong đề), nên chia việc cho nhiều tác tử không giải quyết được.
5. **Rò rỉ và quá khớp.**
   - Không có rò rỉ: hai skill giữ lại đều qua `validate_skill` (không chứa marker nào của tác vụ đánh giá). Curator chỉ đọc run có `role == "learn"` (`test_04` kiểm tra điều này). Một skill data đã bị chặn vì chứa `orders`, tên tệp của tác vụ đánh giá.
   - Có dấu hiệu quá khớp hẹp: `log-triage-output-conventions` chỉ gồm đúng 3 quy ước của logs-learn, nên bỏ sót quy ước log mới (`rule_source_line`).
   - Biện pháp: chỉ dùng phản hồi tác vụ học; prompt cấm id tác vụ, tên tệp đầu vào, cột, hàm và kết quả; xóa hai bộ skill trừu tượng; không sửa tay skill; đóng băng bằng tag `freeze` và kiểm tra `verify_freeze`.
6. **Nhiễu.**
   - Cùng bộ skill, tác vụ học, Phần 3.4 (`results/skills-auto-dev`) so với sau đóng băng: điểm **giống hệt** (10/10, 5/8, 9/9; chênh 0 check). Token thì dao động mạnh: code-learn 143.812 → 71.097, data-learn 75.549 → 41.463, logs-learn 34.380 → 37.485.
   - Hai lần `baseline data-learn` (phụ lục) cũng cùng 5/8 nhưng 51.742 so với 25.022 token.
   - Vậy ở `temperature=1`, **điểm** ổn định (nhiễu quan sát được bằng 0 check), còn **token** có thể chênh khoảng 2 lần. Chênh lệch +6 check của `skills-auto` lớn hơn nhiều so với nhiễu điểm quan sát được, và có cơ chế giải thích bằng vết. Ngược lại, so sánh token giữa các điều kiện chỉ đáng tin khi chênh lệch lớn (như 3× của `subagents`); +15% của `skills-auto` nằm trong mức nhiễu.

## 9. Hạn chế và tính hợp lệ

1. **Ít tác vụ, mỗi cấu hình chạy một lần.** Chỉ 3 tác vụ mỗi vai trò, mỗi tổ hợp điều kiện × tác vụ chạy 1 lần (riêng `skills-auto` trên tác vụ học có 2 lần). Không thể tính khoảng tin cậy; kết luận "+6 check" dựa vào việc nhiễu điểm quan sát được bằng 0 trên 4 cặp lặp lại, một mẫu rất nhỏ.
2. **Một mô hình, buộc dùng `temperature=1`.** Chỉ dùng `gpt-6-luna`, vốn không cho đặt `temperature=0`. Mô hình mạnh nên check kỹ thuật bão hòa (18/18), khiến không đo được tác dụng của skill hay subagent lên nhóm lỗi A-D. Với mô hình yếu hơn kết quả có thể khác, ví dụ reviewer có thể bắt được lỗi kỹ thuật.
3. **Tác vụ thiết kế sẵn quy ước ẩn.** Mọi lỗi đều là nhóm E, tức là thiếu thông tin chứ không phải thiếu năng lực. Điều này ưu ái `skills-auto` (skill mang thông tin mới) và bất lợi cho `subagents` (chỉ tổ chức lại thông tin sẵn có). Kết luận không khái quát cho tác vụ mà lỗi chủ yếu là kỹ thuật.
4. **Can thiệp vào curator.** Prompt curator được sửa một lần sau khi thấy hai bộ skill trừu tượng (mục 6), vẫn trong giai đoạn học và trước khi xem tác vụ đánh giá. Kết quả `skills-auto` vì vậy phản ánh "curator + một vòng chỉnh prompt của người", không phải curator mặc định. Việc thiếu skill data cũng phụ thuộc vào một lần sinh ngẫu nhiên (từ `orders`).
5. **Môi trường.** Lỗi CRLF trên Windows buộc phải bỏ lần chạy baseline đầu và chạy lại; một lần chạy baseline data-learn song song đã ghi đè kết quả (phụ lục). Số đếm `tool_calls`, `skills_read` chỉ tính luồng chính, không thấy việc bên trong subagent.

## 10. Kết luận

Trên 6 tác vụ với `gpt-6-luna`, mọi check kỹ thuật đều đạt (18/18) ở cả ba điều kiện, và toàn bộ điểm mất là do quy ước Acme không có trong đề. Skill do curator sinh tăng điểm đánh giá từ 0,60 lên 0,79 (+6 check) với token chỉ tăng ~15%, nhưng chỉ ở các quy ước đã gặp trong tác vụ học (6/6); quy ước mới đạt 0/3, và có chuyển giao âm khi skill log bị áp vào tác vụ data. Đa tác tử không cải thiện check nào mà tốn ~3,1 lần token, vì không tạo ra thông tin còn thiếu. Đề xuất tiếp theo: cho curator ghi skill theo từng họ tác vụ với `description` nêu rõ loại đầu ra (ví dụ "JSON error report from logs"), thêm một skill quy trình "khi đề nhắc tới review bot hoặc quy ước, tìm và liệt kê quy ước trước khi làm", rồi chạy lặp nhiều lần để đo phương sai.

## Phụ lục

- Lệnh đã chạy (theo thứ tự; trong WSL, venv `~/venvs/lab20`):
  1. `pytest` (29 passed, offline), `python scripts/tour.py`.
  2. `python -m lab.runner --condition baseline --tasks data-learn code-learn logs-learn`: lần đầu bị hủy do lỗi CRLF (xem mục 4); lần chạy `subagents` đầu tiên bị dừng giữa chừng vì cùng lý do.
  3. Sửa môi trường: checkout lại `tasks/` với LF (`git -c core.autocrlf=false checkout -- tasks/`), xóa `__pycache__` trong `tasks/`.
  4. `python -m lab.runner --condition baseline --tasks data-learn code-learn logs-learn` rồi `python -m lab.runner --condition subagents --tasks learn` (chạy lại, tuần tự).
  5. Ghi chú: `results/baseline/data-learn/run.json` là của một lần chạy `baseline data-learn` thứ hai bắt đầu lúc 05:15:18 UTC, chạy song song và ghi đè lần chạy lúc 05:12 (log của lần bị ghi đè: 5/8, 51.742 token). Hai lần cùng điểm 5/8 với cùng các check thất bại; báo cáo dùng số liệu trong `run.json`. Chênh lệch token giữa hai lần (25.022 so với 51.742) là một ví dụ về nhiễu ở `temperature=1`.
  6. `python -m lab.curator` ×3 (xóa bộ skill sau lần 1 và lần 2; sửa `CURATOR_PROMPT` trước lần 3; xem mục 6).
  7. `python -m lab.runner --condition skills-auto --tasks learn` (Phần 3.4).
  8. `git commit -m "hypotheses"` (`ed16cbd`); `git commit --allow-empty -m "freeze skills" && git tag freeze` (`110baa9`).
  9. `mv results/skills-auto results/skills-auto-dev`; `python -m lab.runner --condition baseline --tasks eval`; `--condition subagents --tasks eval`; `--condition skills-auto --tasks all` (tuần tự).
  10. `python scripts/verify_freeze.py` → `OK`; `python -m lab.compare > report/table.md`; `python scripts/check_breakdown.py`.
- Thử thách mở rộng (nếu có): không thực hiện.
- Ghi chú khác: `final_message` trong `run.json` lấy `AIMessage.text` (chỉ phần văn bản), vì `gpt-6-luna` qua Responses API trả về các khối reasoning đã mã hóa. `trace.md` (do `render_trace` có sẵn sinh ra) vẫn chứa các khối này.
