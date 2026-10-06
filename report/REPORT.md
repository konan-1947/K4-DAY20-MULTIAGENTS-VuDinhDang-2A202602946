# Báo cáo Lab: Self evolving Agentic

> Sao chép tệp này thành `report/REPORT.md` (đã làm ở Phần 0) và điền dần qua các Phần của lab. Xóa các dòng hướng dẫn dạng trích dẫn (bắt đầu bằng `>`). Văn phong kỹ thuật, ngắn gọn, mọi nhận định đi kèm số liệu hoặc bằng chứng. Trong buổi học: điền mục 1 đến 7 (bản nháp). Sau buổi học: hoàn thiện mục 8 đến 10.

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Vũ Đình Đăng
- Mã sinh viên: 2A202602946

- Nhà cung cấp và mô hình: `openai:gpt-4.1-mini`; nhiệt độ `0`; `recursion_limit=60`.
- Deep Agents `0.7.21`; Python 3.12 trên Linux container Docker.
- Số lần chạy tác vụ đã dùng / ngân sách: sẽ tổng kết sau các lần chạy chính thức; GUIDE gợi ý tối đa khoảng 30 lượt.
- Commit của tag `freeze`: sẽ điền sau khi đóng băng bộ skill cải tiến.

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

> Dự đoán điều kiện nào đạt điểm cao nhất trên **tác vụ đánh giá** và vì sao. Nêu căn cứ từ phân loại lỗi (mục 4) và từ tài liệu tham khảo. Điền cả ba dòng; `verify_freeze.py` kiểm tra điều này.

- H1 (subagents so với baseline): Dự đoán subagents không cải thiện ổn định điểm đánh giá so với baseline và tốn nhiều token hơn. Trên ba tác vụ học, chỉ data và logs gọi implementer; điểm tương ứng là 3/8 và 1/9, thấp hơn baseline data (5/8) và bằng baseline logs (1/9), trong khi token tăng.
- H2 (skills-auto so với baseline): Dự đoán bộ skill tổng quát có thể hỗ trợ các check kỹ thuật và quy ước lặp lại nhưng sẽ không khắc phục quy ước mới của tác vụ đánh giá; các tổng kết nghiên cứu được nêu trong GUIDE cảnh báo về quá khớp và lợi ích không chuyển giao. Ba skill cuối đều hợp lệ nhưng lượt thử có `skills_read=0`, nên mức cải thiện dự kiến nhỏ và không ổn định.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán điểm đánh giá thấp hơn điểm học do tác vụ đánh giá dùng dữ liệu khác và thêm quy ước mới; kết quả học hiện cho thấy quy ước đầu ra và phân tích log là nguồn lỗi đáng kể.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có các công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; công cụ `execute` cho phép chạy lệnh shell.
2. `task` khởi chạy subagent tạm thời `general-purpose`, có thể nghiên cứu và chạy nhiều bước; mỗi lần gọi thường stateless và chỉ thấy phần mô tả tác vụ được gửi, không tự động kế thừa toàn bộ ngữ cảnh của tác tử chính.
3. Mô tả `task` yêu cầu gửi đầy đủ chi tiết vì subagent chỉ thấy nội dung được truyền; mô tả `execute` yêu cầu trích dẫn đường dẫn có dấu cách và tránh `find`/`grep` shell, dùng công cụ `glob`/`grep` thay thế.

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

> Chỉ dùng tác vụ học. Mỗi dòng là một check thất bại.

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

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
Chưa có kết quả đánh giá chính thức. Bảng sẽ được cập nhật sau khi đóng băng và chạy các tác vụ eval.
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Bạn đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

> Nêu ít nhất 3 hạn chế và ảnh hưởng của từng hạn chế đến kết luận (ví dụ: chỉ 3 tác vụ mỗi vai trò, mỗi cấu hình chạy một lần, nhiễu của mô hình, tác vụ do giảng viên thiết kế sẵn quy ước, chỉ một mô hình).

1.
2.
3.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
