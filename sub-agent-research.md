# Research: Chia Sub-agent Cho Migration RPG Fixed Column Sang Java + DB2 SQL

Tài liệu này ghi lại hướng nghiên cứu cách chia sub-agent khi xây dựng hệ thống chuyển đổi IBM i / AS400 RPG fixed column sang Java và IBM DB2 SQL. Mục tiêu là dùng Claude Code hoặc các coding agent tương tự để phát triển từng phase, nhưng vẫn giữ được tính deterministic, traceable và có thể test được.

## Bối cảnh

Pipeline migration đang được nghiên cứu:

```text
Tokenizer / Parser
    ↓
AST + Symbol Table
    ↓
Semantic Analyzer
    ↓
IR
    ↓
CFG / DFG / File Access Graph
    ↓
Pattern-based Translator
    ↓
Java + DB2 SQL
    ↓
Golden Master Testing
```

Ý tưởng là mỗi phase có thể được một sub-agent phụ trách. Tuy nhiên, không nên để các agent truyền logic cho nhau bằng text tự do. Mỗi phase cần tạo ra artifact có schema rõ ràng để phase sau đọc vào.

## Nguyên tắc chính

### 1. Sub-agent không thay thế compiler pipeline

Sub-agent nên được xem là người xây dựng, reviewer hoặc gap handler cho từng phase. Phần lõi của hệ thống vẫn nên là compiler-style pipeline:

- input rõ ràng.
- output có schema.
- deterministic rule.
- trace được về source line.
- có test riêng cho từng phase.

Nếu để agent sau đọc mô tả text của agent trước rồi "đoán tiếp", logic migration sẽ rất dễ sai ngầm.

### 2. Contract giữa các phase phải là structured artifact

Nên có các artifact trung gian:

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
generated/
tests/
```

Mỗi artifact nên có JSON Schema hoặc format contract riêng:

```text
schemas/
  token.schema.json
  ast.schema.json
  symbol-table.schema.json
  semantic-model.schema.json
  ir.schema.json
  graph.schema.json
  translation-plan.schema.json
```

### 3. Mỗi phase cần giữ source trace

Mỗi node quan trọng trong AST, semantic model, IR và graph nên giữ được thông tin:

```text
source_file
source_line
source_column
raw_text
```

Điều này giúp debug conversion, giải thích SQL/Java sinh ra từ đâu, và tạo report cho những case chưa convert được.

## Đề xuất chia sub-agent

### Agent 1: Source & Parser Agent

Trách nhiệm:

- Đọc RPG fixed column.
- Xử lý comment, continuation line, fixed/free mixed nếu có.
- Đọc `/COPY` nếu cần.
- Parse các spec chính: H/F/D/C/O spec.
- Tạo token stream và AST sơ bộ.

Input:

```text
RPG source files
Copybook files
Parser config
```

Output:

```text
tokens.json
ast.json
parser_diagnostics.json
```

Rủi ro:

- Fixed column bị lệch cột.
- RPG có nhiều dialect/version.
- `/COPY` và external-described files làm context phụ thuộc bên ngoài.

### Agent 2: Symbol & Semantic Agent

Trách nhiệm:

- Tạo symbol table.
- Resolve variable, field, file, record format.
- Lấy thông tin type, length, decimal.
- Hiểu indicator và global state.
- Annotate AST với semantic information.

Input:

```text
ast.json
DDS/schema files
DB2 catalog metadata nếu có
```

Output:

```text
symbols.json
semantic_model.json
semantic_diagnostics.json
```

Rủi ro:

- Thiếu DDS/schema sẽ làm semantic không đầy đủ.
- Packed/zoned decimal, date, char padding có thể ảnh hưởng kết quả.
- Indicator có ý nghĩa nghiệp vụ ẩn trong code.

### Agent 3: IR Agent

Trách nhiệm:

- Chuyển AST đã annotate sang IR trung lập hơn.
- Normalize opcode RPG như `CHAIN`, `READ`, `SETLL`, `READE`, `UPDATE`, `DELETE`, `EVAL`, `IF`, `DO`.
- Biểu diễn subroutine call và side effects.
- Giữ trace về source.

Input:

```text
ast.json
symbols.json
semantic_model.json
```

Output:

```text
ir.json
ir_diagnostics.json
```

Rủi ro:

- IR quá gần RPG thì khó sinh SQL set-based.
- IR quá tổng quát thì mất chi tiết cần thiết của RPG.
- Side effect của subroutine có thể bị bỏ sót.

### Agent 4: Graph Analysis Agent

Trách nhiệm:

- Tạo Control Flow Graph.
- Tạo Data Flow Graph.
- Tạo File Access Graph.
- Tạo dependency graph giữa program, file, copybook, procedure.

Input:

```text
ir.json
semantic_model.json
```

Output:

```text
cfg.json
dfg.json
file_access_graph.json
dependency_graph.json
graph_diagnostics.json
```

Rủi ro:

- Flow phức tạp do subroutine, GOTO, indicator, loop.
- Data flow sai sẽ làm translator sinh SQL sai.
- Cần phân biệt read-only, update, delete và insert access.

### Agent 5: Pattern-based Translator Agent

Trách nhiệm:

- Nhận IR và graph để phát hiện migration pattern.
- Phân loại đoạn nào có thể chuyển sang SQL set-based.
- Phân loại đoạn nào nên giữ procedural trong Java hoặc SQL cursor.
- Tạo translation plan thay vì sinh code trực tiếp.

Input:

```text
ir.json
cfg.json
dfg.json
file_access_graph.json
semantic_model.json
```

Output:

```text
translation_plan.json
unsupported_cases.json
translator_diagnostics.json
```

Pattern nên ưu tiên:

- `CHAIN` -> `SELECT ... WHERE key = ?`
- `READ` loop -> cursor, batch Java, hoặc set-based SQL.
- Accumulation -> `SUM`, `GROUP BY`, window function.
- Level break -> `GROUP BY`, `ROLLUP`, window function.
- Indicator-driven logic -> boolean conditions có tên rõ nghĩa.

Rủi ro:

- Có đoạn nhìn giống SQL set-based nhưng phụ thuộc thứ tự record.
- Update trong loop có side effect theo sequence.
- Indicator làm điều kiện bị ẩn.

### Agent 6: Codegen Agent

Trách nhiệm:

- Sinh Java service/repository/code orchestration.
- Sinh DB2 SQL.
- Sinh stored procedure hoặc cursor nếu cần.
- Sinh migration notes và mapping report.

Input:

```text
translation_plan.json
semantic_model.json
ir.json
```

Output:

```text
generated/java/
generated/sql/
generated/migration-notes.md
generated/source-map.json
```

Rủi ro:

- SQL đúng syntax nhưng sai DB2 dialect.
- Java sinh ra quá gần RPG và khó maintain.
- Thiếu source map làm khó review.

### Agent 7: Golden Master Testing Agent

Trách nhiệm:

- Tạo test harness so sánh output giữa chương trình cũ và code mới.
- Sinh test cases từ branch logic và boundary values.
- Compare result set, output file, table changes, report output.
- Báo mismatch và liên kết mismatch về source/IR/translation plan.

Input:

```text
RPG baseline output
generated/java/
generated/sql/
semantic_model.json
translation_plan.json
```

Output:

```text
tests/golden-master/
test_results.json
mismatch_report.md
```

Rủi ro:

- Khó chạy baseline RPG nếu không có IBM i environment.
- Dữ liệu test không phủ hết branch.
- Sai khác về decimal rounding, date, char padding, null handling.

## Cấu trúc thư mục đề xuất

```text
agents/
  parser-agent.md
  semantic-agent.md
  ir-agent.md
  graph-agent.md
  translator-agent.md
  codegen-agent.md
  testing-agent.md

schemas/
  token.schema.json
  ast.schema.json
  symbol-table.schema.json
  semantic-model.schema.json
  ir.schema.json
  graph.schema.json
  translation-plan.schema.json

artifacts/
  tokens.json
  ast.json
  symbols.json
  semantic_model.json
  ir.json
  cfg.json
  dfg.json
  file_access_graph.json
  dependency_graph.json
  translation_plan.json

generated/
  java/
  sql/
  migration-notes.md
  source-map.json

tests/
  golden-master/
```

## Cách dùng LLM/sub-agent hợp lý

Nên dùng LLM/sub-agent để:

- xây từng module parser, semantic analyzer, IR transformer, graph builder.
- review artifact và diagnostics.
- giải thích logic RPG.
- đề xuất SQL cho pattern chưa có rule.
- tạo test case.
- phát hiện unsupported case.

Không nên dùng LLM/sub-agent để:

- truyền state giữa các phase bằng text tự do.
- quyết định logic migration mà không có rule hoặc artifact kiểm chứng.
- sinh SQL trực tiếp từ source RPG mà bỏ qua parser/semantic/IR.

## Kết luận nghiên cứu

Chia mỗi phase thành một sub-agent là hướng đi hợp lý, nhưng điểm cốt lõi nằm ở contract giữa các phase. Nếu mỗi agent nhận và trả về structured artifact có schema rõ ràng, hệ thống sẽ gần với compiler pipeline và có thể test được. Nếu mỗi agent chỉ đọc và viết mô tả tự do, kết quả sẽ khó lặp lại, khó trace và dễ sai nghiệp vụ.

Hướng ưu tiên nên là:

```text
Sub-agent builds phase
Phase emits structured artifact
Next phase consumes artifact
Tests validate artifact
LLM assists but does not silently decide core migration logic
```
