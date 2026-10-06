# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Vũ Đình Đăng
- Mã sinh viên: 2A202602946

- Nhà cung cấp và mô hình: `openai:gpt-4.1-mini`; nhiệt độ `0`; `recursion_limit=60`.
- Deep Agents `0.7.21`; Python 3.12 trên Linux container Docker.
- Số lần chạy tác vụ đã dùng / ngân sách: 27 lượt trong vòng làm lại, nằm trong gợi ý tối đa khoảng 30 lượt; thêm 2 lần gọi curator.
- Commit của tag `freeze`: `d6bffcd`.

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): Dự đoán subagents không cải thiện ổn định điểm đánh giá so với baseline và tốn nhiều token hơn. Trên ba tác vụ học, chỉ data và logs gọi implementer; điểm tương ứng là 3/8 và 1/9, thấp hơn baseline data (5/8) và bằng baseline logs (1/9), trong khi token tăng.
- H2 (skills-auto so với baseline): Dự đoán bộ skill tổng quát có thể hỗ trợ các check kỹ thuật và quy ước lặp lại nhưng sẽ không khắc phục quy ước mới của tác vụ đánh giá; các tổng kết nghiên cứu được nêu trong GUIDE cảnh báo về quá khớp và lợi ích không chuyển giao. Ba skill cuối đều hợp lệ nhưng lượt thử có `skills_read=0`, nên mức cải thiện dự kiến nhỏ và không ổn định.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán điểm đánh giá thấp hơn điểm học do tác vụ đánh giá dùng dữ liệu khác và thêm quy ước mới; kết quả học hiện cho thấy quy ước đầu ra và phân tích log là nguồn lỗi đáng kể.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có các công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; công cụ `execute` cho phép chạy lệnh shell.
2. `task` khởi chạy subagent tạm thời `general-purpose`, có thể nghiên cứu và chạy nhiều bước; mỗi lần gọi thường stateless và chỉ thấy phần mô tả tác vụ được gửi, không tự động kế thừa toàn bộ ngữ cảnh của tác tử chính.
3. Mô tả `task` yêu cầu gửi đầy đủ chi tiết vì subagent chỉ thấy nội dung được truyền; mô tả `execute` yêu cầu trích dẫn đường dẫn có dấu cách và tránh `find`/`grep` shell, dùng công cụ `glob`/`grep` thay thế.

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| code-learn | `rule_type_hints`, `rule_regression_tests`, `rule_changelog` | E | Ba `detail` yêu cầu type hints đầy đủ, tệp kiểm thử hồi quy mới và mục Unreleased trong changelog. |
| code-learn | `tests_not_modified` | G (tuân thủ chỉ dẫn) | `detail`: không được sửa các tệp kiểm thử có sẵn; vết cho thấy tác tử đã dò và đọc thư mục `tests/`. |
| data-learn | `rule_money_in_cents`, `rule_meta_block`, `rule_clean_csv` | E | `detail` yêu cầu tiền dạng integer cents, object `meta`, và CSV đã chuẩn hoá. |
| logs-learn | `entry_count`, `timestamps_utc`, `exception_fields`, `repeat_counts`, `counts_by_service` | D | Check báo lần lượt số entry sai (19), chỉ 7/25 timestamp đúng, 18 exception sai, 18 repeat count sai và tổng theo service sai; vết cho thấy xử lý log nhiều dòng và repeat không chính xác. |
| logs-learn | `rule_service_names`, `rule_sorted_errors`, `rule_schema_header` | E | `detail` nêu chuẩn hoá tên service, thứ tự sắp xếp và các trường schema bắt buộc. |

Nhận xét: lỗi quy ước E xuất hiện ở cả ba họ tác vụ; dữ liệu học có 9 check quy ước thất bại. Skill có thể nhắc quy trình rà soát quy ước, nhưng lượt thử chưa đọc skill nên chưa có bằng chứng phòng ngừa.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa: `explorer` (đọc và báo cáo), `implementer` (thực hiện), `reviewer` (kiểm tra độc lập), mỗi vai trò có mô tả điều kiện gọi riêng.
- `subagent_calls`: code-learn 0, data-learn 1, logs-learn 1. Tác tử chính không giao việc cho các tác vụ code; hai tác vụ còn lại giao cho `implementer`.
- Lời giao việc data/logs nêu dữ liệu, yêu cầu chính và đường dẫn đầu ra; ở data, implementer bỏ sót convention chi tiết. Không có call nào cho thấy reviewer được dùng.
- Token baseline → subagents: code 40,176 → 62,496 (+22,320); data 45,385 → 87,159 (+41,774); logs 21,344 → 45,881 (+24,537). Điểm data giảm 5/8 → 3/8, logs giữ 1/9; code giữ 6/10.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Curator được gọi 2 lần trong vòng làm lại. Lần đầu sinh 3 skill hợp lệ nhưng `description` không kích hoạt việc đọc (`skills_read=0` ở cả ba tác vụ học), nên đã xóa theo hướng dẫn chất lượng. Sau khi prompt curator yêu cầu `description` bắt đầu bằng “Use when” và nêu rõ họ tác vụ, lần chạy lại sinh 3 skill cuối; không sửa tay nội dung skill.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `python-code-repair-best-practices` | Tổng quát cho sửa gói Python | Đúng: yêu cầu đọc docstring, sửa nguyên nhân gốc, type hints, regression tests và changelog | 9 dòng; description nêu Python code repair; lượt thử `skills_read=0`. |
| `tabular-data-cleaning-standardization` | Tổng quát cho dữ liệu bảng bẩn | Đúng: chuẩn hóa ngày/múi giờ, giá trị thiếu, trùng lặp, đơn vị tiền và metadata | 11 dòng; description nêu tabular data; lượt thử `skills_read=0`. |
| `multiline-log-parsing-and-normalization` | Tổng quát cho log nhiều dòng | Đúng: tách entry, traceback/repeat, UTC, schema, chuẩn hóa và sắp xếp | 11 dòng; description nêu multiline log; lượt thử `skills_read=0`. |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

```text
| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 6/10 | 6/10 | 6/10 |
| data-learn | 5/8 | 3/8 | 5/8 |
| logs-learn | 1/9 | 1/9 | 6/9 |
| code-eval | 6/11 | 6/11 | 6/11 |
| data-eval | 5/9 | 5/9 | 5/9 |
| logs-eval | 1/10 | 0/10 | 6/10 |
| Mean score - learning tasks | 0.45 | 0.36 | 0.63 |
| Mean score - evaluation tasks | 0.40 | 0.37 | 0.57 |
| Mean tokens per run | 37,040 | 58,036 | 49,990 |
| Runs that read a skill | 0/6 | 0/6 | 0/6 |

check_breakdown.py:
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval     12/18         0/12          38,445      0/3
baseline      learn    12/18         0/9           35,635      0/3
subagents     eval     11/18         0/12          50,894      0/3
subagents     learn    10/18         0/9           65,178      0/3
skills-auto   eval     17/18         0/12          43,907      0/3
skills-auto   learn    17/18         0/9           56,073      0/3

verify_freeze.py: checked 6 runs of skill conditions: OK.
Không lần chạy chính thức nào có `error` hoặc `skills_modified=true`.
```

## 8. Phân tích

1. Trên tác vụ học, subagents thấp hơn baseline (0.36 so với 0.45), còn skills-auto cao hơn (0.63). Trên eval, subagents thấp hơn baseline (0.37 so với 0.40), còn skills-auto đạt 0.57. Tuy nhiên `skills_read=0/6`, nên mức tăng của skills-auto không thể quy cho skill; đây là dấu hiệu nhiễu giữa các lần gọi model.
2. Skills-auto đạt 17/18 check kỹ thuật trên cả learn và eval, cao hơn baseline 12/18, nhưng cả ba điều kiện đều đạt 0 check quy ước (0/9 learn, 0/12 eval). Do không skill nào được đọc, không có bằng chứng skill giúp nhóm check nào; quy ước mới của eval cũng không được skill giúp.
3. Không thể chỉ ra một check được skill giúp theo quan hệ nhân quả vì `skills_read=0`. Ví dụ, logs-eval tăng từ baseline 1/10 lên skills-auto 6/10, nhưng vết không có lệnh đọc `multiline-log-parsing-and-normalization`; ngược lại các check `rule_*` đều trượt, phù hợp với việc các quy tắc trong skill chưa được nạp.
4. Token trung bình toàn bộ sáu tác vụ là 37,040 baseline, 58,036 subagents và 49,990 skills-auto; riêng eval là 38,445, 50,894 và 43,907. Dùng mean eval score chia mean eval token, baseline đạt khoảng 10.4, subagents 7.3, skills-auto 13.0 điểm chuẩn hóa trên một triệu token. Subagents không đáng chi phí trong mẫu này vì tốn nhiều token hơn nhưng điểm eval thấp hơn baseline.
5. Ba skill không chứa id, tệp hay giá trị riêng của eval và đều qua `validate_skill`; curator chỉ đọc run có `role=learn`. `verify_freeze.py` xác nhận hash skill không đổi và các lần chạy diễn ra sau freeze. Không thấy rò rỉ trực tiếp, nhưng vì skill không được đọc nên chưa thể đánh giá overfitting qua tác động thực tế.
6. Với cùng bộ skill cuối, lượt dev đạt code 6/10, data 5/8, logs 1/9; sau freeze đạt 6/10, 5/8, 6/9. Chênh lệch số check đạt là 0, 0 và +5 dù `skills_read` đều bằng 0. Dao động lớn ở logs cho thấy chênh lệch một lần chạy chưa đáng tin cậy và cần lặp nhiều seed/lần chạy.

## 9. Hạn chế và tính hợp lệ

1. Chỉ có ba tác vụ cho mỗi vai trò; một tác vụ logs thay đổi mạnh có thể chi phối trung bình, nên kết quả khó khái quát sang miền khác.
2. Mỗi cấu hình/tác vụ chính thức chỉ chạy một lần; logs-learn dao động từ 1/9 lên 6/9 dù cùng skill và không đọc skill, làm suy yếu mọi kết luận nhân quả.
3. `skills_read=0` ở toàn bộ lần chạy chính thức; thí nghiệm đo được pipeline tạo, kiểm định và đóng băng skill nhưng chưa đo được hiệu quả khi model thực sự nạp nội dung skill.
4. Chỉ dùng `gpt-4.1-mini`, nhiệt độ 0 và một thiết kế prompt; model hoặc prompt khác có thể kích hoạt skill và subagent khác.

## 10. Kết luận

Subagents tăng chi phí nhưng không tăng điểm trong sáu tác vụ này. Skills-auto có điểm cao hơn baseline, đặc biệt ở logs-eval, nhưng không skill nào được đọc nên không thể coi đó là hiệu quả của self-evolving. Curator đã tạo ba skill hợp lệ, ngắn và không rò rỉ eval; quy trình freeze được verifier xác nhận. Bước tiếp theo là cải thiện cơ chế kích hoạt skill và chạy lặp từng cấu hình để tách tác động skill khỏi nhiễu model.

## Phụ lục

- Lệnh đã chạy (theo thứ tự): pytest; tour; baseline learn; subagents learn; curator; skills-auto learn; xóa skill không kích hoạt; curator lần 2; skills-auto learn; commit hypotheses/skills; tag freeze; baseline eval; subagents eval; skills-auto all; compare; verify_freeze; check_breakdown; pytest.
- Thử thách mở rộng: không thực hiện; các lượt dev bổ sung chỉ dùng để đánh giá description và nhiễu trước freeze.
- Ghi chú khác: `.env` bị Git bỏ qua. Khóa API đã xuất hiện trong cuộc trò chuyện nên cần thu hồi sau khi hoàn tất.
