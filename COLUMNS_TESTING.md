# Testing COLUMNS Expression Support

This document provides comprehensive tests for the newly implemented DuckDB COLUMNS expression support in sqlparser-rs.

## Features Implemented

The following DuckDB COLUMNS syntax is now supported:

- ✅ `COLUMNS(*)` - Basic wildcard
- ✅ `COLUMNS(* EXCLUDE (col1, col2))` - Exclude specific columns
- ✅ `COLUMNS(* EXCLUDE col1)` - Exclude single column (no parentheses)
- ✅ `COLUMNS(* REPLACE (expr AS col))` - Replace column with expression
- ✅ `COLUMNS(['col1', 'col2', 'col3'])` - Array of column names
- ✅ `COLUMNS(columns(c -> c LIKE '%pattern%'))` - Column filtering with columns() function
- ✅ `COLUMNS(columns(c -> c ~ 'regex'))` - Regex pattern matching with columns() function

## Rust Tests

Run the sqlparser-rs tests to verify parser functionality:

```bash
# Run all DuckDB tests
cargo test --test sqlparser_duckdb

# Run specific COLUMNS tests
cargo test --test sqlparser_duckdb -- test_columns

# Run GenericDialect COLUMNS tests
cargo test --test sqlparser_common parse_columns_expression_generic_dialect

# Run all tests with features
cargo test --all-features
```

## Real DuckDB Testing

### Setup

Ensure DuckDB is available in your development environment:

```bash
# Using nix (recommended)
nix develop

# Or install DuckDB manually
# macOS: brew install duckdb
# Ubuntu: apt install duckdb
# Windows: Download from https://duckdb.org/
```

### Test SQL Queries

Create a test dataset and run the following queries in DuckDB:

```sql
-- Create test table
CREATE TABLE employees (
    id INTEGER,
    first_name VARCHAR,
    last_name VARCHAR,
    department_id INTEGER,
    salary DECIMAL(10,2),
    hire_date DATE,
    email VARCHAR,
    phone VARCHAR,
    address VARCHAR
);

-- Insert test data
INSERT INTO employees VALUES 
    (1, 'John', 'Doe', 101, 75000.00, '2020-01-15', 'john.doe@company.com', '555-0101', '123 Main St'),
    (2, 'Jane', 'Smith', 102, 82000.00, '2019-03-22', 'jane.smith@company.com', '555-0102', '456 Oak Ave'),
    (3, 'Bob', 'Johnson', 101, 68000.00, '2021-07-08', 'bob.johnson@company.com', '555-0103', '789 Pine Rd'),
    (4, 'Alice', 'Williams', 103, 91000.00, '2018-11-30', 'alice.williams@company.com', '555-0104', '321 Elm St'),
    (5, 'Charlie', 'Brown', 102, 77000.00, '2020-09-14', 'charlie.brown@company.com', '555-0105', '654 Maple Dr');
```

### Test Cases

#### 1. Basic COLUMNS Functionality

```sql
-- Test 1: Basic wildcard
SELECT COLUMNS(*) FROM employees LIMIT 1;

-- Test 2: Array syntax for specific columns
SELECT COLUMNS(['id', 'first_name', 'last_name']) FROM employees LIMIT 3;
```

#### 2. EXCLUDE Functionality

```sql
-- Test 3: Exclude single column
SELECT COLUMNS(* EXCLUDE address) FROM employees LIMIT 1;

-- Test 4: Exclude multiple columns
SELECT COLUMNS(* EXCLUDE (email, phone, address)) FROM employees LIMIT 1;

-- Test 5: Exclude with WHERE clause
SELECT COLUMNS(* EXCLUDE (id, department_id)) 
FROM employees 
WHERE salary > 75000;
```

#### 3. REPLACE Functionality

```sql
-- Test 6: Replace with calculation
SELECT COLUMNS(* REPLACE (salary * 1.1 AS salary)) 
FROM employees 
WHERE department_id = 101;

-- Test 7: Replace with concatenation
SELECT COLUMNS(* REPLACE (first_name || ' ' || last_name AS full_name) EXCLUDE (first_name, last_name))
FROM employees 
LIMIT 3;
```

#### 4. Column Name Patterns (DuckDB columns() function)

```sql
-- Test 8: Pattern matching with columns() function
SELECT COLUMNS(columns(c -> c LIKE '%_name')) FROM employees LIMIT 1;

-- Test 9: Complex pattern conditions
SELECT COLUMNS(columns(c -> c LIKE '%id' OR c LIKE '%salary')) FROM employees LIMIT 2;

-- Test 10: Exclusion patterns
SELECT COLUMNS(columns(c -> c NOT LIKE '%address%' AND c != 'phone')) FROM employees LIMIT 1;
```

#### 5. Regular Expression Patterns

```sql
-- Test 11: Regex pattern with columns() function
SELECT COLUMNS(columns(c -> c ~ '^.*_name$')) FROM employees LIMIT 1;

-- Test 12: Character class regex
SELECT COLUMNS(columns(c -> c ~ '[a-z]+_[a-z]+')) FROM employees LIMIT 2;

-- Test 13: Quantifier regex
SELECT COLUMNS(columns(c -> length(c) BETWEEN 3 AND 10)) FROM employees LIMIT 1;
```

#### 6. Array Syntax

```sql
-- Test 14: Specific column array
SELECT COLUMNS(['first_name', 'last_name', 'email']) FROM employees LIMIT 1;

-- Test 15: Mixed case and underscores
SELECT COLUMNS(['ID', 'department_id', 'hire_date']) FROM employees LIMIT 1;

-- Test 16: Single column array
SELECT COLUMNS(['salary']) FROM employees LIMIT 1;
```

#### 7. Complex Combinations

```sql
-- Test 17: EXCLUDE with REPLACE
SELECT COLUMNS(* EXCLUDE (phone, address) REPLACE (UPPER(email) AS email))
FROM employees 
WHERE department_id IN (101, 102)
LIMIT 3;

-- Test 18: Pattern matching with EXCLUDE
SELECT COLUMNS(columns(c -> c LIKE '%name') EXCLUDE first_name) 
FROM employees 
LIMIT 2;

-- Test 19: Multiple modifiers (parser test - some combinations may not be valid in DuckDB)
-- This tests our parser's ability to handle multiple modifiers
```

## sqlparser-rs CLI Testing

Test the parser directly using the CLI tool:

```bash
# Build the CLI example
cargo build --example cli

# Test basic COLUMNS parsing
echo "SELECT COLUMNS(*) FROM table1" | cargo run --example cli -- /dev/stdin --dialect=duckdb

# Test array syntax
echo "SELECT COLUMNS(['user_id', 'user_name']) FROM users" | cargo run --example cli -- /dev/stdin --dialect=duckdb

# Test columns() function with pattern
echo "SELECT COLUMNS(columns(c -> c LIKE 'user_%')) FROM users" | cargo run --example cli -- /dev/stdin --dialect=duckdb

# Test columns() function with regex
echo "SELECT COLUMNS(columns(c -> c ~ '^user_.*')) FROM users" | cargo run --example cli -- /dev/stdin --dialect=duckdb

# Test complex combination
echo "SELECT COLUMNS(* EXCLUDE (id, created_at) REPLACE (amount / 100 AS amount)) FROM transactions" | cargo run --example cli -- /dev/stdin --dialect=duckdb
```

## Expected Results

### Parser Output
The sqlparser-rs should successfully parse all the above SQL statements without errors and produce proper AST representations.

### DuckDB Execution
- **COLUMNS(*)**: Returns all columns
- **COLUMNS(['first_name', 'last_name'])**: Returns only the specified columns
- **COLUMNS(columns(c -> c LIKE '%_name'))**: Returns columns matching the pattern condition
- **COLUMNS(columns(c -> c ~ '^.*_name$'))**: Returns columns matching the regex pattern
- **COLUMNS(* EXCLUDE (email, phone, address))**: Returns all columns except the excluded ones
- **COLUMNS(* REPLACE (salary * 1.1 AS salary))**: Returns all columns with salary multiplied by 1.1

### Round-trip Testing
All parsed queries should be convertible back to valid SQL strings that produce the same parse tree:

```bash
# Test round-trip conversion
echo "SELECT COLUMNS(['user_name', 'user_email']) FROM users" | \
  cargo run --example cli -- /dev/stdin --dialect=duckdb | \
  grep -A 1000 "AST:" | \
  cargo run --example cli -- /dev/stdin --dialect=duckdb
```

## Troubleshooting

### Common Issues

1. **Array syntax parsing fails**: Ensure array elements are quoted strings: `['col1', 'col2']`
2. **Lambda expression parsing fails**: Check arrow syntax: `c -> condition`
3. **Regex pattern not recognized**: Ensure regex patterns are single-quoted strings
4. **Multiple modifiers conflict**: Some combinations may not be semantically valid in DuckDB

### Debug Commands

```bash
# Check parser internals
RUST_LOG=debug cargo test --test sqlparser_duckdb -- test_columns_with_lambda

# Verbose test output
cargo test --test sqlparser_duckdb -- --nocapture test_columns

# Check specific dialect support
cargo run --example cli -- --help
```

## Contributing

When adding new COLUMNS features:

1. Add parser tests in `tests/sqlparser_duckdb.rs`
2. Add round-trip tests to verify Display implementation
3. Update this documentation with new test cases
4. Test with real DuckDB to ensure compatibility
5. Consider adding support to GenericDialect for broader compatibility

## References

- [DuckDB COLUMNS Documentation](https://duckdb.org/docs/stable/sql/expressions/star.html#columns-expression)
- [GitHub Issue #1972](https://github.com/apache/datafusion-sqlparser-rs/issues/1972)
- [sqlparser-rs Documentation](https://docs.rs/sqlparser/)