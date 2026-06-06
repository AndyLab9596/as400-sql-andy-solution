# Post-implementation Roadmap: Các Bước Sau Khi Hoàn Thành Pipeline Migration

Tài liệu này ghi lại các bước nên làm sau khi đã implement xong pipeline/sub-agent cho việc chuyển đổi IBM i / AS400 RPG fixed column sang Java và IBM DB2 SQL.

Mục tiêu của giai đoạn này không phải là chạy migration hàng loạt ngay, mà là kiểm chứng pipeline, tìm lỗi theo từng phase, tạo baseline, chạy pilot, rồi mới mở rộng theo batch.

## Tổng quan

Sau khi implementation hoàn tất, thứ tự khuyến nghị:

```text
End-to-end sample
    ↓
Artifact validation
    ↓
Migration assessment report
    ↓
Golden master baseline
    ↓
Pilot program
    ↓
Review translation plan
    ↓
Codegen + SQL review
    ↓
Golden master comparison
    ↓
Feedback loop
    ↓
Batch migration
```

Generated Java/SQL không nên được xem là kết quả cuối cùng. Kết quả cuối cùng phải là generated code pass golden master test và có trace rõ ràng từ RPG source sang IR, translation plan, rồi Java/SQL.

## Bước 1: Chạy end-to-end với sample nhỏ

Chọn 3-5 RPG program thật nhỏ hoặc sample tự tạo:

```text
simple CHAIN
READ loop
IF/ELSE + UPDATE
INSERT/DELETE đơn giản
CALL/subroutine đơn giản
```

Mục tiêu là kiểm tra toàn bộ flow:

```text
RPG
  -> AST
  -> Semantic Model
  -> IR
  -> Graph
  -> Translation Plan
  -> Java/SQL
  -> Test
```

Checkpoint:

```text
- Sample có chạy hết pipeline không?
- Mỗi phase có artifact output không?
- Artifact phase sau có consume được output phase trước không?
- Có source trace từ generated code về RPG source không?
- Có diagnostics cho case chưa support không?
```

Nếu sample nhỏ chưa chạy end-to-end được, chưa nên mở rộng sang program thật lớn.

## Bước 2: Validate artifact giữa các phase

Kiểm tra tất cả artifact trung gian:

```text
tokens.json
ast.json
symbols.json
semantic_model.json
ir.json
cfg.json
dfg.json
file_access_graph.json
translation_plan.json
```

Mỗi artifact cần trả lời được:

```text
- Có validate được bằng schema không?
- Có source trace không?
- Có diagnostics không?
- Có unsupported case rõ ràng không?
- Phase sau có thể consume artifact mà không cần đọc explanation text không?
```

Artifact validation là checkpoint quan trọng để tránh việc các phase phụ thuộc vào diễn giải tự do của LLM.

## Bước 3: Tạo migration assessment report

Trước khi convert thật, nên scan toàn bộ source RPG để tạo report đánh giá migration.

Report nên có:

```text
program_name
complexity_level
detected_opcodes
if_else_count
loop_count
nested_loop_count
call_count
subroutine_count
file_access_summary
unsupported_opcodes
risk_flags
recommended_strategy
```

Các strategy gợi ý:

```text
set_based_sql
sql_cursor
java_procedural
manual_review
unsupported
```

Ví dụ risk flags:

```text
indicator_driven_logic
update_in_loop
record_order_dependency
nested_loop
external_call
subroutine_side_effect
missing_schema
unsupported_opcode
```

Mục tiêu của report:

- Biết project lớn và phức tạp đến đâu.
- Chọn pilot program hợp lý.
- Tách nhóm program theo độ khó.
- Ước lượng phần có thể auto-convert và phần cần manual review.

## Bước 4: Làm golden master baseline

Nếu có IBM i environment, chạy chương trình RPG cũ với bộ input cố định và lưu lại output.

Baseline nên bao gồm:

```text
input tables/files
output tables/files
reports/spool output
return status
job log nếu cần
```

Golden master baseline là sự thật gốc để so sánh với Java/SQL mới. Nếu không có baseline, migration dễ bị đánh giá bằng cảm giác "code nhìn có vẻ đúng".

Checkpoint:

```text
- Có bộ input cố định chưa?
- Có expected output chưa?
- Có lưu trạng thái table trước/sau chưa?
- Có capture report/spool output nếu chương trình sinh report không?
- Có ghi job log hoặc error status nếu cần không?
```

## Bước 5: Chọn pilot program

Chọn 1-3 program thật để chạy pilot.

Pilot tốt nên có:

```text
đủ đại diện nghiệp vụ
có READ/CHAIN/UPDATE
có test data
ít CALL external
ít indicator khó hiểu
ít report formatting phức tạp
```

Không nên chọn program quá đơn giản vì không kiểm chứng được pipeline. Cũng không nên chọn program phức tạp nhất ngay từ đầu vì dễ bị nghẽn ở quá nhiều điểm cùng lúc.

Output mong muốn sau pilot:

```text
pilot_translation_plan.json
generated/java/
generated/sql/
sql_review_report.md
golden_master_result.json
mismatch_report.md nếu có
```

## Bước 6: Review translation plan trước Codegen

Trước khi sinh Java/SQL, review `translation_plan.json`.

Các điểm cần kiểm tra:

```text
- Đoạn nào chuyển sang set-based SQL?
- Đoạn nào dùng SQL cursor?
- Đoạn nào giữ Java procedural?
- Đoạn nào unsupported hoặc manual review?
- Risk flags có hợp lý không?
- Source mapping có đủ không?
- Translation decision có dựa trên semantic/IR/graph không?
```

Nếu translation plan sai, codegen phía sau chắc chắn sẽ sai. Vì vậy translation plan nên là checkpoint bắt buộc trước khi sinh code.

## Bước 7: Sinh code và chạy SQL review

Sau khi Codegen tạo Java/SQL, chạy SQL review.

Các nhóm review nên có:

```text
DB2 compatibility
join correctness
null handling
decimal precision
date handling
transaction behavior
performance/index usage
naming convention
readability
```

Với SQL dài, nên split theo 2 chiều:

```text
SQL chunk
convention rule
```

Ví dụ chunk:

```text
CTE section
JOIN section
WHERE section
GROUP BY / HAVING
Window functions
INSERT / UPDATE / DELETE / MERGE
Stored procedure variables/cursors
Exception handling
```

Output nên có:

```text
sql_review_report.md
sql_review_findings.json
recommended_fixes.md
```

SQL review không thay thế golden master test. Nó chỉ là quality gate cho generated SQL.

## Bước 8: Chạy golden master comparison

So sánh output của chương trình cũ với Java/SQL mới.

Những thứ cần compare:

```text
expected vs actual result set
table changes
report output
decimal value
date value
char padding
null handling
ordering
business result
```

Nếu có mismatch, report nên link ngược về:

```text
source RPG line
IR operation
translation_plan item
generated SQL/Java line
```

Các loại mismatch nên phân loại:

```text
data
decimal
date
padding
null
ordering
business_logic
db2_dialect
test_data
unknown
```

## Bước 9: Tạo feedback loop

Mỗi lỗi từ pilot hoặc golden master comparison phải được phân loại về đúng phase.

Các nhóm lỗi:

```text
parser_bug
semantic_bug
ir_bug
graph_bug
translator_rule_missing
codegen_bug
db2_dialect_issue
sql_review_gap
test_data_issue
manual_business_ambiguity
unsupported_legacy_pattern
```

Nguyên tắc sửa:

- Nếu parser sai, sửa parser và test parser.
- Nếu semantic sai, sửa semantic analyzer và test semantic.
- Nếu IR sai, sửa IR transformer và test IR.
- Nếu translation decision sai, sửa translator rule.
- Nếu SQL syntax sai, sửa codegen hoặc DB2 dialect layer.
- Không vá random trong generated code nếu lỗi thuộc pipeline.

Feedback loop giúp pipeline tốt dần và tránh lặp lại cùng một lỗi ở nhiều program.

## Bước 10: Mở rộng theo batch

Khi pilot ổn, mới chạy migration theo batch nhỏ.

Thứ tự batch gợi ý:

```text
Batch 1: Level 1-2 programs
Batch 2: Level 3 programs
Batch 3: Level 4 with manual review
Batch 4: unsupported/special cases
```

Mỗi batch nên có report:

```text
converted_count
passed_golden_master_count
failed_golden_master_count
manual_review_count
unsupported_count
common_failure_reasons
rules_added
open_risks
```

Không nên tăng batch size nếu:

```text
- Golden master pass rate thấp.
- Có nhiều lỗi cùng loại chưa được sửa ở pipeline.
- Translation plan thường xuyên phải sửa tay.
- Source trace không đủ để debug mismatch.
- SQL review phát hiện nhiều lỗi DB2 dialect lặp lại.
```

## Quality gates

Trước khi coi một program là migrated, cần pass các gate:

```text
1. Artifact schema validation passed.
2. Translation plan reviewed.
3. Generated Java/SQL has source map.
4. SQL review passed or findings resolved.
5. Golden master comparison passed.
6. Unsupported/manual review items resolved.
7. Migration notes generated.
```

## Kết luận

Sau khi implementation xong, việc quan trọng nhất là kiểm chứng pipeline bằng sample nhỏ, rồi pilot thật, rồi feedback loop, sau đó mới batch migration. Generated code chỉ là một phần của kết quả. Kết quả đáng tin là code đã qua artifact validation, SQL review, golden master comparison và có trace rõ ràng về source RPG.
