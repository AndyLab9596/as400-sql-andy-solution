# IBM i / AS400 RPG Fixed Column to Java + DB2 SQL Migration

Tài liệu này ghi lại hướng tiếp cận đã bàn cho dự án chuyển đổi RPG fixed column trên IBM i / AS400 sang Java và IBM DB2 SQL. Trong dự án này, logic nghiệp vụ được ưu tiên chuyển sang SQL khi có thể, nhưng vẫn cần Java cho các phần procedural, orchestration, validation, hoặc những logic không phù hợp để viết bằng set-based SQL.

## Workflow ban đầu

Workflow được đề xuất ban đầu:

```text
RPG Fixed Column
    ↓
Tokenizer
    ↓
AST
    ↓
Semantic Analyzer
    ↓
IR
    ↓
Graph Builder
    ↓
LLM
    ↓
SQL
```

Đánh giá chung: workflow này đúng hướng vì nó tiếp cận bài toán như một compiler. Tuy nhiên, nếu dùng cho migration nghiêm túc thì cần bổ sung một số lớp quan trọng và không nên để LLM là backend chính sinh SQL.

## Workflow đề xuất

Workflow thực tế hơn:

```text
RPG + DDS + Copybooks
    ↓
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
    ↓
LLM-assisted Explanation / Refactor / Gap Handling
```

## Vai trò của từng thành phần

### Tokenizer / Parser

RPG fixed column phụ thuộc rất nhiều vào vị trí cột, nên parser phải hiểu layout của source code. Cần xử lý các thành phần như:

- H spec, F spec, D spec, C spec, O spec.
- opcode, factor 1, factor 2, result field.
- indicator columns.
- comment line và continuation line.
- RPG fixed/free mixed nếu source có cả hai dạng.

Không nên chỉ dùng regex đơn giản vì fixed column rất dễ sai khi line bị lệch cột hoặc có nhiều biến thể cũ.

### AST

AST không chỉ nên là expression tree. Nó cần biểu diễn được:

- khai báo file.
- khai báo biến và data structure.
- calculation specs.
- subroutine/procedure.
- file I/O operations.
- indicators.
- record formats.
- copybooks và `/COPY`.

AST nên giữ metadata nguồn như file, line, column và raw text để debug conversion dễ hơn.

### Semantic Analyzer

Đây là lớp rất quan trọng. Semantic analyzer cần hiểu:

- field type, length, decimal.
- packed decimal, zoned decimal, character, date semantics.
- schema từ DDS hoặc DB2 catalog.
- ý nghĩa của indicators.
- key access path.
- hành vi của RPG opcode.
- implicit conversions.
- global state và side effects của subroutine.

Nếu semantic model yếu, SQL sinh ra có thể nhìn đúng nhưng sai nghiệp vụ.

### IR

IR là dạng trung gian giúp chuẩn hóa RPG thành logic độc lập hơn với cú pháp cũ.

Ví dụ, RPG:

```text
C     CUSTNO        CHAIN     CUSTMAST
C                   IF        %FOUND
C                   EVAL      BAL = BAL + AMT
C                   UPDATE    CUSTREC
C                   ENDIF
```

Có thể được chuẩn hóa thành IR:

```text
lookup CUSTMAST by CUSTNO
if found:
  BAL = BAL + AMT
  update CUSTMAST
```

Từ IR mới quyết định nên sinh Java, SQL, stored procedure, cursor, hay set-based SQL.

### Graph Builder

Nên tách graph thành nhiều loại:

- Control Flow Graph: luồng `IF`, `DO`, `LEAVE`, `ITER`, subroutine call.
- Data Flow Graph: field nào đọc/ghi field nào.
- File Access Graph: program nào read/update/delete file nào.
- Dependency Graph: liên kết program, copybook, file, procedure.

Graph giúp phát hiện đoạn nào có thể chuyển sang SQL set-based và đoạn nào nên giữ procedural trong Java.

## Vai trò của LLM

Không nên để LLM là compiler backend chính theo kiểu:

```text
Graph Builder
    ↓
LLM
    ↓
SQL
```

Lý do: migration legacy cần deterministic behavior. Cùng một input phải cho ra cùng một output, có trace, có rule rõ ràng, và có thể kiểm chứng bằng test.

LLM nên được dùng như assistant:

- giải thích logic RPG thành ngôn ngữ tự nhiên.
- gợi ý SQL tương đương.
- phát hiện pattern nghiệp vụ.
- refactor SQL dài.
- tạo documentation.
- hỗ trợ case mà rule-based translator chưa cover.
- sinh test case.
- so sánh output cũ/mới và gợi ý nguyên nhân sai khác.

Phần mapping opcode, type conversion, file I/O, indicator, update/delete/chain nên được làm rule-based càng nhiều càng tốt.

## Các pattern nên ưu tiên

### File lookup

RPG operations như `CHAIN`, `SETLL`, `READE`, `READP` có thể mapping sang:

- `SELECT ... WHERE key = ?`
- cursor.
- join.
- existence check.

### Loop over records

RPG `READ` loop có thể mapping sang:

- SQL cursor nếu cần procedural.
- `INSERT INTO ... SELECT`.
- `UPDATE ... WHERE`.
- batch Java processing.

### Accumulation / total

Logic cộng dồn có thể mapping sang:

- `SUM`.
- `GROUP BY`.
- window function.

### Level break

Report-style level break có thể mapping sang:

- `GROUP BY`.
- `ROLLUP`.
- window function.
- Java grouping nếu formatting phức tạp.

### Indicator-driven logic

Indicator là phần nguy hiểm vì ý nghĩa thường ẩn trong code cũ. Nên normalize indicator thành biến boolean có tên rõ nghĩa khi có thể.

Ví dụ:

```text
*IN03 -> isExitRequested
*IN90 -> customerFound
```

Nếu không đặt tên tốt, Java và SQL sinh ra sẽ khó maintain.

## Kết luận

Kiến trúc nên lấy rule-based translator làm lõi chính, kết hợp parser, semantic model, IR và graph analysis. LLM nên đóng vai trò hỗ trợ giải thích, refactor, review và xử lý khoảng trống, không nên là thành phần quyết định logic migration duy nhất.

Trong bài toán này, việc quan trọng không chỉ là sinh SQL, mà là phát hiện đúng đoạn RPG nào có thể chuyển sang SQL set-based và đoạn nào cần giữ lại ở Java hoặc procedural SQL.
