# Implementation Guide: Dùng Claude Code/Codex Để Tạo Sub-agent Theo Flow Migration

Tài liệu này ghi lại cách triển khai từng bước khi dùng Claude Code hoặc Codex để tạo các sub-agent cho pipeline chuyển đổi IBM i / AS400 RPG fixed column sang Java và IBM DB2 SQL.

Mục tiêu không phải là yêu cầu LLM dịch thẳng RPG sang SQL, mà là dùng LLM/coding agent để xây từng phase của một compiler-style pipeline có schema, artifact, diagnostics và test rõ ràng.

## Flow tổng thể

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

## Nguyên tắc implement

- Không tạo agent để dịch bằng text tự do.
- Mỗi phase phải có input/output artifact rõ ràng.
- Mỗi artifact nên có schema để validate.
- Mỗi node quan trọng phải có source trace.
- Nếu gặp case chưa support, ghi diagnostics thay vì đoán.
- Không để phase sau làm thay việc của phase trước.
- Không sinh SQL/Java trước khi có IR và translation plan.

Source trace tối thiểu:

```text
source_file
source_line
source_column
raw_text
```

## Bước 1: Tạo cấu trúc thư mục nền

Tạo cấu trúc đề xuất:

```text
agents/
  common-rules.md
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
  samples/

src/
  parser/
  semantic/
  ir/
  graph/
  translator/
  codegen/
  testing/

tests/
  parser/
  semantic/
  ir/
  graph/
  translator/
  codegen/
  golden-master/
```

Prompt mẫu:

```text
Hãy tạo cấu trúc thư mục nền cho dự án migration RPG fixed column sang Java + DB2 SQL.

Yêu cầu:
- Tạo agents/, schemas/, artifacts/, src/, tests/.
- Không implement logic ngay.
- Tạo README ngắn trong từng thư mục nếu cần để giải thích vai trò.
```

## Bước 2: Tạo common rules cho tất cả agent

File cần tạo:

```text
agents/common-rules.md
```

Nội dung nên bao gồm:

```text
- Mỗi phase phải nhận input artifact rõ ràng và sinh output artifact rõ ràng.
- Output không được chỉ là mô tả tự do.
- Mỗi artifact phải validate được bằng schema tương ứng.
- Mỗi node quan trọng phải giữ source trace.
- Nếu thiếu thông tin, ghi diagnostics thay vì đoán.
- Không sinh Java/SQL nếu phase hiện tại không phải Codegen Agent.
- Cùng một input phải tạo ra cùng một output.
```

Prompt mẫu:

```text
Tạo agents/common-rules.md cho tất cả sub-agent.

Rules phải nhấn mạnh:
- structured artifact
- schema validation
- source trace
- diagnostics
- deterministic output
- không làm việc của phase sau
```

## Bước 3: Implement Parser Agent

File agent:

```text
agents/parser-agent.md
```

Trách nhiệm:

- Đọc RPG fixed column.
- Parse H/F/D/C/O spec ở mức cơ bản.
- Xử lý comment và continuation line.
- Tạo `tokens.json`, `ast.json`, `parser_diagnostics.json`.
- Không đoán semantic.

Prompt tạo agent:

```text
Tạo agents/parser-agent.md.

Agent này phụ trách Tokenizer / Parser cho RPG fixed column.

Input:
- RPG source files
- Copybook files nếu có
- Parser config

Output:
- tokens.json
- ast.json
- parser_diagnostics.json

Yêu cầu:
- Parse H/F/D/C/O spec.
- Giữ source trace trên token và AST node.
- Nếu line chưa support, ghi diagnostics.
- Không implement semantic analyzer.
- Không sinh SQL/Java.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/parser-agent.md.

Implement src/parser.
Tạo schemas/token.schema.json và schemas/ast.schema.json tối thiểu.
Tạo tests/parser với sample RPG fixed column.

Yêu cầu:
- Parser sinh tokens.json, ast.json, parser_diagnostics.json.
- Mỗi token và AST node có source trace.
- Test phải cover CHAIN, IF, EVAL, UPDATE cơ bản.
```

Sample RPG nên dùng:

```text
C     CUSTNO        CHAIN     CUSTMAST
C                   IF        %FOUND
C                   EVAL      BAL = BAL + AMT
C                   UPDATE    CUSTREC
C                   ENDIF
```

Checkpoint:

```text
- tokens.json validate được chưa?
- ast.json validate được chưa?
- parser_diagnostics.json có ghi unsupported case chưa?
- AST có source trace chưa?
- Parser không làm semantic chưa?
```

## Bước 4: Implement Semantic Agent

File agent:

```text
agents/semantic-agent.md
```

Trách nhiệm:

- Tạo symbol table.
- Resolve variable, field, file, record format.
- Annotate type, length, decimal.
- Hiểu indicator ở mức structural.
- Tạo `symbols.json`, `semantic_model.json`, `semantic_diagnostics.json`.

Prompt tạo agent:

```text
Tạo agents/semantic-agent.md.

Agent này phụ trách Symbol Table và Semantic Analyzer.

Input:
- ast.json
- DDS/schema files nếu có
- DB2 catalog metadata nếu có

Output:
- symbols.json
- semantic_model.json
- semantic_diagnostics.json

Yêu cầu:
- Resolve variable, field, file, record format.
- Annotate type, length, decimal.
- Ghi diagnostics nếu thiếu DDS/schema.
- Không sinh IR.
- Không sinh SQL/Java.
```

Prompt implement:

```text
Đọc agents/common-rules.md, agents/semantic-agent.md và schemas/ast.schema.json.

Implement src/semantic.
Tạo schemas/symbol-table.schema.json và schemas/semantic-model.schema.json.
Viết tests/semantic dùng ast.json từ parser.

Yêu cầu:
- Sinh symbols.json và semantic_model.json.
- Nếu thiếu schema metadata, ghi semantic_diagnostics.json.
- Không tự đoán type khi không đủ dữ liệu.
```

Checkpoint:

```text
- symbols.json có file/field/variable chưa?
- semantic_model.json có annotation chưa?
- Thiếu DDS/schema có diagnostics chưa?
- Semantic Agent không sinh IR chưa?
```

## Bước 5: Implement IR Agent

File agent:

```text
agents/ir-agent.md
```

Trách nhiệm:

- Chuyển AST đã annotate sang IR.
- Normalize opcode RPG.
- Giữ source trace.
- Tạo `ir.json`, `ir_diagnostics.json`.

Mapping ban đầu:

```text
CHAIN  -> lookup operation
READ   -> record iteration operation
EVAL   -> assignment operation
IF     -> conditional operation
UPDATE -> update operation
DELETE -> delete operation
```

Prompt tạo agent:

```text
Tạo agents/ir-agent.md.

Agent này phụ trách chuyển AST + semantic_model sang IR.

Input:
- ast.json
- symbols.json
- semantic_model.json

Output:
- ir.json
- ir_diagnostics.json

Yêu cầu:
- Normalize opcode RPG thành operation trung gian.
- Giữ source trace.
- Không sinh graph.
- Không sinh SQL/Java.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/ir-agent.md.

Implement src/ir.
Tạo schemas/ir.schema.json.
Viết tests/ir cho CHAIN, IF, EVAL, UPDATE, READ loop.

Yêu cầu:
- IR không quá phụ thuộc fixed column syntax.
- IR vẫn giữ đủ chi tiết để sinh graph và translation plan.
- Unsupported opcode phải vào ir_diagnostics.json.
```

Checkpoint:

```text
- ir.json có operation chuẩn hóa chưa?
- Mỗi operation có source trace chưa?
- Unsupported opcode có diagnostics chưa?
- IR Agent không sinh graph/code chưa?
```

## Bước 6: Implement Graph Analysis Agent

File agent:

```text
agents/graph-agent.md
```

Trách nhiệm:

- Tạo Control Flow Graph.
- Tạo Data Flow Graph.
- Tạo File Access Graph.
- Tạo dependency graph.

Prompt tạo agent:

```text
Tạo agents/graph-agent.md.

Agent này phụ trách graph analysis.

Input:
- ir.json
- semantic_model.json

Output:
- cfg.json
- dfg.json
- file_access_graph.json
- dependency_graph.json
- graph_diagnostics.json

Yêu cầu:
- CFG biểu diễn control flow.
- DFG biểu diễn field dependency.
- File Access Graph biểu diễn read/update/delete/insert.
- Không tạo translation plan.
- Không sinh SQL/Java.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/graph-agent.md.

Implement src/graph.
Tạo schemas/graph.schema.json.
Viết tests/graph dùng ir.json.

Yêu cầu:
- Sinh cfg.json, dfg.json, file_access_graph.json.
- Phân biệt read-only, update, delete, insert access.
- Nếu flow chưa support, ghi graph_diagnostics.json.
```

Checkpoint:

```text
- CFG có node/edge chưa?
- DFG có dependency giữa field chưa?
- File Access Graph có read/update/delete/insert chưa?
- Graph Agent không tạo translation plan chưa?
```

## Bước 7: Implement Pattern-based Translator Agent

File agent:

```text
agents/translator-agent.md
```

Trách nhiệm:

- Nhận IR và graph.
- Phát hiện pattern migration.
- Phân loại logic thành `set_based_sql`, `sql_cursor`, `java_procedural`, `unsupported`.
- Tạo `translation_plan.json`.

Prompt tạo agent:

```text
Tạo agents/translator-agent.md.

Agent này phụ trách Pattern-based Translator.

Input:
- ir.json
- cfg.json
- dfg.json
- file_access_graph.json
- semantic_model.json

Output:
- translation_plan.json
- unsupported_cases.json
- translator_diagnostics.json

Yêu cầu:
- Chỉ tạo translation plan.
- Không sinh Java/SQL trực tiếp.
- Nếu thiếu semantic information, đánh dấu unsupported hoặc diagnostics.
- Phân loại logic thành set_based_sql, sql_cursor, java_procedural, unsupported.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/translator-agent.md.

Implement src/translator.
Tạo schemas/translation-plan.schema.json.
Viết tests/translator cho CHAIN, READ loop, accumulation, UPDATE.

Yêu cầu:
- CHAIN đơn giản có thể thành lookup/select plan.
- READ loop có thể được phân loại cursor hoặc set_based_sql tùy pattern.
- Không sinh code trong phase này.
```

Checkpoint:

```text
- translation_plan.json validate được chưa?
- Plan có source mapping chưa?
- Unsupported cases có lý do chưa?
- Translator không sinh SQL/Java chưa?
```

## Bước 8: Implement Codegen Agent

File agent:

```text
agents/codegen-agent.md
```

Trách nhiệm:

- Sinh Java theo translation plan.
- Sinh DB2 SQL theo translation plan.
- Sinh migration notes.
- Sinh source map.

Prompt tạo agent:

```text
Tạo agents/codegen-agent.md.

Agent này phụ trách code generation.

Input:
- translation_plan.json
- semantic_model.json
- ir.json

Output:
- generated/java/
- generated/sql/
- generated/migration-notes.md
- generated/source-map.json

Yêu cầu:
- Không tự thay đổi translation decision.
- Sinh DB2 SQL theo translation_plan.json.
- Sinh Java orchestration nếu plan yêu cầu.
- Mọi generated code phải trace được về source line.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/codegen-agent.md.

Implement src/codegen.
Viết tests/codegen dạng snapshot cho generated SQL/Java.

Yêu cầu:
- Input là translation_plan.json.
- Output là generated/java, generated/sql, migration-notes.md, source-map.json.
- Không tự phân loại lại pattern.
```

Checkpoint:

```text
- SQL có đúng DB2 dialect ở mức cơ bản chưa?
- Java có source map chưa?
- Codegen có tuân thủ translation_plan chưa?
- Có migration notes chưa?
```

## Bước 9: Implement Golden Master Testing Agent

File agent:

```text
agents/testing-agent.md
```

Trách nhiệm:

- So sánh output RPG cũ và Java/SQL mới.
- Tạo test cases.
- Báo mismatch.
- Liên kết mismatch về source map và translation plan.

Prompt tạo agent:

```text
Tạo agents/testing-agent.md.

Agent này phụ trách Golden Master Testing.

Input:
- RPG baseline output
- generated Java/SQL output
- translation_plan.json
- generated/source-map.json

Output:
- tests/golden-master/
- test_results.json
- mismatch_report.md

Yêu cầu:
- Compare expected/actual.
- Nếu mismatch, ghi rõ loại mismatch.
- Link mismatch về source line nếu có source map.
```

Prompt implement:

```text
Đọc agents/common-rules.md và agents/testing-agent.md.

Implement src/testing.
Tạo golden master harness tối thiểu.
Viết sample expected/actual và sinh mismatch_report.md.

Yêu cầu:
- Phân loại mismatch: data, decimal, date, padding, null, ordering, business_logic.
- Không sửa generated code trong phase testing.
```

Checkpoint:

```text
- test_results.json có expected/actual/mismatch chưa?
- mismatch_report.md có source mapping chưa?
- Có phân loại mismatch chưa?
- Testing Agent không sửa codegen output chưa?
```

## Thứ tự triển khai khuyến nghị

Không nên tạo cả 7 agent và toàn bộ implementation cùng lúc. Nên đi theo thứ tự:

```text
1. agents/common-rules.md
2. agents/parser-agent.md
3. schemas/token.schema.json + schemas/ast.schema.json
4. src/parser + tests/parser

5. agents/semantic-agent.md
6. schemas/symbol-table.schema.json + schemas/semantic-model.schema.json
7. src/semantic + tests/semantic

8. agents/ir-agent.md
9. schemas/ir.schema.json
10. src/ir + tests/ir

11. agents/graph-agent.md
12. schemas/graph.schema.json
13. src/graph + tests/graph

14. agents/translator-agent.md
15. schemas/translation-plan.schema.json
16. src/translator + tests/translator

17. agents/codegen-agent.md
18. src/codegen + tests/codegen

19. agents/testing-agent.md
20. src/testing + tests/golden-master
```

## Prompt chuẩn dùng cho từng phase

Khi dùng Claude Code/Codex, nên prompt theo format:

```text
Bạn đang làm phase: <PHASE_NAME>.

Hãy đọc:
- agents/common-rules.md
- agents/<phase-agent>.md
- schemas/<required-schema>.json nếu có

Yêu cầu:
- Implement module src/<phase>.
- Chỉ làm trách nhiệm của phase này.
- Output phải là artifact đã định nghĩa.
- Mỗi artifact phải validate được bằng schema.
- Mỗi node quan trọng phải có source trace.
- Nếu gặp case chưa support, ghi diagnostics thay vì đoán.
- Viết test cho sample nhỏ.
```

Ví dụ cho Parser:

```text
Bạn đang làm phase Parser Agent.

Hãy đọc:
- agents/common-rules.md
- agents/parser-agent.md
- schemas/token.schema.json
- schemas/ast.schema.json

Yêu cầu:
- Implement module src/parser.
- Không implement semantic analyzer.
- Output phải là tokens.json, ast.json, parser_diagnostics.json.
- Mỗi AST node phải có source trace.
- Nếu gặp syntax chưa support, ghi diagnostics thay vì đoán.
- Viết unit test cho samples/simple-chain.rpg.
```

## Checklist sau mỗi phase

Sau mỗi phase, kiểm tra:

```text
- Schema đã có chưa?
- Artifact output có validate được không?
- Có source trace không?
- Có diagnostics không?
- Có test sample không?
- Có unsupported case rõ ràng không?
- Phase sau có thể consume artifact mà không cần đọc explanation text không?
- Agent hiện tại có lỡ làm việc của phase sau không?
```

Nếu checklist này chưa đạt, không nên chuyển sang phase tiếp theo.

## Kết luận

Cách implement an toàn là dùng Claude Code/Codex để xây từng phase nhỏ, có artifact rõ ràng, có schema và có test. LLM nên giúp viết code, review logic, tạo diagnostics và test case; không nên để LLM truyền state giữa các phase bằng mô tả tự do hoặc tự quyết định migration logic mà không có rule.

Flow implement nên giữ cố định:

```text
Agent instruction
    ↓
Schema
    ↓
Implementation
    ↓
Sample input
    ↓
Artifact output
    ↓
Validation test
    ↓
Next agent
```
