# CS253 Assignment 1 — Memory-Efficient Versioned File Indexer

A command-line C++ tool that builds a word-frequency index over large text files using a fixed-size buffer, and answers three types of analytical queries across one or two file versions.

---

## Requirements

- C++17 or later
- A C++ compiler: `g++` (GCC 8+) or `clang++` (Clang 7+)
- No external libraries — standard library only

---

## Compilation

```bash
g++ "241054_Sumath.cpp" -o analyzer
```

---

## Usage

```
./analyzer --file <path> --version <name> --buffer <kb> --query <type> [query-specific flags]
```

### All Flags

| Flag | Description |
|------|-------------|
| `--buffer <kb>` | Buffer size in KB. Must be between **256** and **1024**. Required for all queries. |
| `--query <type>` | Query type: `word`, `diff`, or `top`. Required for all queries. |
| `--file <path>` | Input file path. Used by `word` and `top` queries. |
| `--version <name>` | Version label for the file. Used by `word` and `top` queries. |
| `--file1 <path>` | First input file. Used by `diff` query. |
| `--version1 <name>` | Version label for first file. Used by `diff` query. |
| `--file2 <path>` | Second input file. Used by `diff` query. |
| `--version2 <name>` | Version label for second file. Used by `diff` query. |
| `--word <token>` | Word to look up. Used by `word` and `diff` queries. |
| `--top <k>` | Number of top results to return. Used by `top` query. |

---

## Query Types & Examples

### 1. Word Count Query
Returns how many times a word appears in a given file version.

```bash
./analyzer --file test_logs.txt --version v1 --buffer 256 --query word --word error
```

**Output:**
```
Version: v1
Word: error
Count: 605079
Allocated buffer size (KB): 256
Total execution time: 2.18918 seconds
```

---

### 2. Top-K Query
Returns the K most frequent words in a file version, sorted by descending frequency.

```bash
./analyzer --file test_logs.txt --version v1 --buffer 256 --query top --top 10
```

**Output:**
```
Version: v1
Top 10 words:
devops -> 1209558
debug -> 605150
error -> 605079
info -> 604266
warning -> 604149
orderservice -> 484437
paymentservice -> 484078
authservice -> 483842
searchservice -> 483162
userservice -> 483125
Allocated buffer size: 256 KB
Total execution time: 2.15474 seconds
```

---

### 3. Difference Query
Returns the difference in frequency of a word between two file versions (`version1 count - version2 count`).

```bash
./analyzer --file1 test_logs.txt --version1 v1 --file2 verbose_logs.txt --version2 v2 --buffer 256 --query diff --word error
```

**Output:**
```
Version 1: v1
Version 2: v2
Word: error
Difference (v1 - v2): 495377
Allocated buffer size: 256 KB
Total execution time: 4.80576 seconds
```

---

## How It Works

The program processes files incrementally — the entire file is **never** loaded into memory.

1. `BufferedFileReader` allocates a fixed buffer (your specified KB) once and reuses it every read cycle.
2. `Tokenizer` scans the raw buffer byte-by-byte, extracting lowercase alphanumeric tokens. A **carry string** handles words that are split across two buffer boundaries.
3. `VersionedIndexer` stores each unique word exactly once in an `unordered_map`, incrementing its count on each occurrence.
4. After all chunks are read, the appropriate `Query` subclass (`WordQuery`, `DiffQuery`, or `TopKQuery`) runs against the completed index.

**Memory usage grows only with the number of unique words** — not with file size.

---

## Error Handling

The program throws descriptive errors for all invalid inputs and writes them to `stderr`. It exits with code `1` on any error.

Common errors:

| Situation | Error Message |
|-----------|---------------|
| `--buffer` or `--query` not provided | `Usage: --buffer and --query are required.` |
| Buffer outside 256–1024 KB | `Buffer size must be between 256 KB and 1024 KB.` |
| Non-integer passed to `--buffer` | `--buffer must be an integer between 256 and 1024.` |
| Non-integer passed to `--top` | `--top must be a positive integer.` |
| File not found | `Failed to open file: <path>` |
| Version name not found in index | `Version not found: <name>` |
| Missing or empty required flag | `Missing or empty required argument: <flag>` |
| Invalid `--query` type | `Invalid query type. Use: word \| diff \| top` |

---

## Design Overview

| Class | Role |
|-------|------|
| `BufferedFileReader` | Reads file in fixed-size chunks; buffer allocated once |
| `Tokenizer` | Converts raw bytes to lowercase tokens; handles split words via carry |
| `VersionedIndexer` | Stores word-frequency maps per version; answers all query types |
| `Query` | Abstract base class with pure virtual `execute()` |
| `WordQuery` | Derived — prints frequency of one word in one version |
| `DiffQuery` | Derived — prints frequency difference across two versions |
| `TopKQuery` | Derived — prints top-K words using `partial_sort` |
| `CommandLineParser` | Parses and validates `--key value` arguments from `argv` |

---

## C++ Features Demonstrated

- **Inheritance** — `Query` (abstract base), `WordQuery`, `DiffQuery`, `TopKQuery` (derived)
- **Runtime Polymorphism** — `virtual execute() = 0`; dispatched via `std::unique_ptr<Query>`
- **Function Overloading** — `getWordCount(version, word)` vs `getWordCount(word)`
- **Exception Handling** — `try` / `catch` / `throw` throughout, including per-`stoi()` blocks
- **User-defined Template** — `template<typename T> void printTopK(...)`

---

## Notes

- Word matching is **case-insensitive**. `Error`, `ERROR`, and `error` are all counted as `error`.
- Words are defined as contiguous sequences of **alphanumeric characters** only. Punctuation and whitespace are delimiters.
- A positive difference in a `diff` query means the word appears more in `version1`. A negative value means it appears more in `version2`.
- The `--version` flag is a user-defined label — it does not need to match the filename.
