# strings

String utilities for SSPUR in the spirit of Apache Commons Lang `StringUtils`, `WordUtils` and Commons Text: blank checks, safe substrings that never trap, case conversion with a real word splitter, padding by display width, grapheme-aware reversing and abbreviation, edit distances and similarity, word wrapping, accent stripping, URL slugs, natural sorting, and escaping for HTML, XML, JSON, shells, regexes, CSV and URLs.

It builds on the builtin `Str` methods (`trim`, `split`, `split_once`, `pad_left`, `fold_case`, `compare_ci`, `words`, `reverse`, regexes) and adds what they leave out. Every function is pure and total: indexes are clamped, a missing separator has a documented result, and no input traps. Results are built with linear pipelines rather than appends in loops, so they stay fast in the interpreter too; the tests run them on 100,000-character inputs in both tiers (the edit distances are O(n m)).

## Example

```sspur
use strings.{to_snake, to_camel, abbreviate, slugify, unique_slug, levenshtein, closest, wrap, pad_left_width, escape_html, reverse_graphemes, natural_sort}

fn main() -> Unit ! log
= do
  log(to_snake("parseHTTPResponse2xx"))
  log(to_camel("max-retry_count"))
  log(abbreviate("The quick brown fox", 12))
  log(slugify("Crème Brûlée & Co."))
  log(unique_slug("hello", ["hello", "hello-2"]))
  log(levenshtein("kitten", "sitting").str)
  log(closest("colour", ["color", "column", "collar"]).str)
  log(wrap("The quick brown fox jumps over the lazy dog", 16))
  log("[" + pad_left_width("日本", 6, ".") + "]")
  log(escape_html("<b>Tom & Jerry</b>"))
  log(reverse_graphemes("ok 👍🏽"))
  log(natural_sort(["img12.png", "img10.png", "img2.png"]).str)
```

Output (the same with `sspur run` and `sspur run --interp`):

```
parse_http_response2xx
maxRetryCount
The quick...
creme-brulee-and-co
hello-3
3
some("color")
The quick brown
fox jumps over
the lazy dog
[..日本]
&lt;b&gt;Tom &amp; Jerry&lt;/b&gt;
👍🏽 ko
["img2.png", "img10.png", "img12.png"]
```

## Conventions

- Indexes and lengths count code points, like the builtin `len`, `take` and `index_of`, unless a function says grapheme clusters (`abbreviate`, `take_graphemes`) or display columns (`pad_left_width`, `center`, `wrap`).
- Whitespace is the Unicode White_Space set. Letters are what the builtin `is_alpha` says; decimal digits are ASCII and the other Unicode Nd blocks of the BMP; a letter is uppercase when it has a lowercase mapping and is its own uppercase.
- "Character set" parameters (`chars`, `set`, `seps`) are strings whose characters form the set.
- Functions that take a refined argument (`width: Int where _ >= 0`) state it; everything else accepts any value, with negative counts treated as 0.

## API
### Empty and blank

- `is_blank(s: Str) -> Bool`: Whether `s` is empty or only whitespace.
- `is_whitespace(s: Str) -> Bool`: Whether `s` is non-empty and only whitespace (see `is_blank` for the set); `""` is not.
- `contains_whitespace(s: Str) -> Bool`: Whether `s` contains at least one whitespace character.
- `is_any_blank(xs: List[Str]) -> Bool`: Whether any string in `xs` is blank; false for an empty list.
- `is_all_blank(xs: List[Str]) -> Bool`: Whether every string in `xs` is blank; true for an empty list.
- `is_any_empty(xs: List[Str]) -> Bool`: Whether any string in `xs` is empty; false for an empty list.
- `default_if_empty(s: Str, d: Str) -> Str`: `s`, or `d` when `s` is empty.
- `default_if_blank(s: Str, d: Str) -> Str`: `s`, or `d` when `s` is blank.
- `first_non_blank(xs: List[Str]) -> Opt[Str]`: The first string in `xs` that is not blank.
- `first_non_empty(xs: List[Str]) -> Opt[Str]`: The first string in `xs` that is not empty.
- `trim_to_none(s: Str) -> Opt[Str]`: `s` trimmed (builtin `trim`, which strips Unicode whitespace), or `none` when nothing is left.
- `none_if_empty(s: Str) -> Opt[Str]`: `some(s)`, or `none` when `s` is empty.

### Predicates

- `is_numeric(s: Str) -> Bool`: Whether `s` is non-empty and every character is a Unicode decimal digit (ASCII `0-9`, Arabic-Indic, Devanagari and the other Nd blocks of the BMP, and fullwidth digits).
- `is_ascii_digits(s: Str) -> Bool`: Whether `s` is non-empty and only ASCII digits `0-9`.
- `is_alphanumeric(s: Str) -> Bool`: Whether `s` is non-empty and every character is a letter (builtin `is_alpha`) or a decimal digit (as `is_numeric`).
- `is_alpha_space(s: Str) -> Bool`: Whether every character of `s` is a letter or the space `" "`; true for `""`, as in Commons.
- `is_alphanumeric_space(s: Str) -> Bool`: Whether every character of `s` is a letter, a decimal digit or the space `" "`; true for `""`.
- `is_ascii(s: Str) -> Bool`: Whether every character of `s` is ASCII (below U+0080); true for `""`.
- `is_ascii_printable(s: Str) -> Bool`: Whether every character of `s` is printable ASCII, U+0020 to U+007E; true for `""`.
- `is_all_upper(s: Str) -> Bool`: Whether `s` is non-empty and every character is an uppercase letter (a character that has a lowercase mapping and is its own uppercase); digits, spaces and uncased letters make it false.
- `is_all_lower(s: Str) -> Bool`: Whether `s` is non-empty and every character is a lowercase letter; the mirror of `is_all_upper`.
- `is_mixed_case(s: Str) -> Bool`: Whether `s` contains both an uppercase and a lowercase letter.

### Graphemes and display width

- `graphemes(s: Str) -> List[Str]`: The grapheme clusters of `s`, an approximation of UAX 29 (see Unicode notes below).
- `grapheme_count(s: Str) -> Int`: The number of grapheme clusters in `s` (see `graphemes`).
- `reverse_graphemes(s: Str) -> Str`: `s` with its grapheme clusters in reverse order, so accents stay on their letters and flags and emoji sequences stay whole (builtin `reverse` reverses code points).
- `take_graphemes(s: Str, n: Int) -> Str`: The first `n` grapheme clusters of `s`; all of `s` when it has fewer, `""` when `n <= 0`.
- `drop_graphemes(s: Str, n: Int) -> Str`: `s` without its first `n` grapheme clusters; `s` when `n <= 0`.
- `display_width(s: Str) -> Int`: The width of `s` in terminal columns: 0 for control and zero-width characters, 2 for East Asian wide characters, emoji and flags, else 1.

### Padding, abbreviation and repetition

- `pad_left_width(s: Str, width: Int, fill: Str) -> Str`: `s` padded on the left with `fill` to `width` display columns (see `display_width`), so CJK and emoji line up in a terminal.
- `pad_right_width(s: Str, width: Int, fill: Str) -> Str`: `s` padded on the right with `fill` to `width` display columns; see `pad_left_width`.
- `center(s: Str, width: Int, fill: Str) -> Str`: `s` centred in `width` display columns with `fill` on both sides; an odd remainder goes on the right, as in Commons `center`.
- `abbreviate(s: Str, width: Int where _ >= 0) -> Str`: `s` cut to at most `width` grapheme clusters, ending in `"..."` when it was cut: `abbreviate( "abcdefg", 6)` is `"abc..."`.
- `abbreviate_with(s: Str, width: Int where _ >= 0, marker: Str) -> Str`: `s` cut to at most `width` grapheme clusters, the last ones being `marker` when it was cut; when `width` is not longer than `marker`, the first `width` clusters of `s` without a marker.
- `abbreviate_middle(s: Str, width: Int where _ >= 0, marker: Str) -> Str`: `s` cut to at most `width` grapheme clusters by replacing its middle with `marker`, keeping one more cluster at the front than at the back when the split is uneven: `abbreviate_middle( "abcdef", 4, ".")` is `"ab.f"`.
- `abbreviate_words(s: Str, width: Int where _ >= 0, marker: Str) -> Str`: `s` cut at a word boundary to at most `width` characters including `marker`, which is added when it was cut: the cut falls at the last whitespace that fits, or inside the first word when no whitespace does (as WordUtils `abbreviate`).
- `truncate_bytes(s: Str, limit: Int where _ >= 0) -> Str`: The longest prefix of `s` whose UTF-8 encoding fits in `limit` bytes, never splitting a character (a grapheme cluster may be split).
- `repeat_sep(s: Str, sep: Str, n: Int where _ >= 0) -> Str`: `s` repeated `n` times with `sep` between the copies; `""` when `n` is 0.
- `mask(s: Str, keep_start: Int where _ >= 0, keep_end: Int where _ >= 0, mask_char: Str) -> Str`: `s` with all but the first `keep_start` and last `keep_end` characters replaced by `mask_char`; masks everything when the kept parts would cover it.
- `rotate_chars(s: Str, k: Int) -> Str`: `s` rotated right by `k` characters (left when `k` is negative), as Commons `rotate`: `rotate_chars("abcdefg", 2)` is `"fgabcde"`.
- `reverse_delimited(s: Str, sep: Str) -> Str`: The parts of `s` between `sep` in reverse order: `reverse_delimited("a.b.c", ".")` is `"c.b.a"`.

### Case

- `case_words(s: Str) -> List[Str]`: The words of `s` for case conversion.
- `to_camel(s: Str) -> Str`: `s` in camelCase: `to_camel("Hello world_HTTP")` is `"helloWorldHttp"`.
- `to_pascal(s: Str) -> Str`: `s` in PascalCase: `"HttpServer"` for `"HTTP server"`.
- `to_snake(s: Str) -> Str`: `s` in snake_case: `"http_server"` for `"HTTPServer"`.
- `to_screaming_snake(s: Str) -> Str`: `s` in SCREAMING_SNAKE_CASE, the constant style: `"HTTP_SERVER"`.
- `to_kebab(s: Str) -> Str`: `s` in kebab-case: `"http-server"`.
- `to_train(s: Str) -> Str`: `s` in Train-Case: `"Http-Server"`.
- `to_dot(s: Str) -> Str`: `s` in dot.case: `"http.server"`.
- `to_title(s: Str) -> Str`: The words of `s` as a title, each capitalized and the rest lowercased, joined by spaces: `to_title("the_quick-brownFox")` is `"The Quick Brown Fox"`.
- `to_sentence(s: Str) -> Str`: The words of `s` as a sentence: lowercase, joined by spaces, the first letter capitalized: `to_sentence("helloWorld")` is `"Hello world"`.
- `capitalize(s: Str) -> Str`: `s` with its first character uppercased (by its full mapping, so `ß` becomes `SS`).
- `uncapitalize(s: Str) -> Str`: `s` with its first character lowercased.
- `swap_case(s: Str) -> Str`: `s` with uppercase letters lowercased and lowercase letters uppercased; other characters stay.
- `capitalize_words(s: Str) -> Str`: `s` with the first letter of every whitespace-separated word capitalized and the rest left as they are, keeping the whitespace (WordUtils `capitalize`).
- `capitalize_fully(s: Str) -> Str`: `s` with every whitespace-separated word capitalized and the rest of it lowercased (WordUtils `capitalizeFully`): `"hELLO wORLD"` is `"Hello World"`.
- `initials(s: Str) -> Str`: The first character of every whitespace-separated word of `s`: `initials("Ada  King lovelace")` is `"AKl"`.

### Splitting and joining

- `split_any(s: Str, seps: Str) -> List[Str]`: The tokens of `s` between any of the characters in `seps`, with empty tokens dropped (Commons `split(str, chars)`): `split_any("a,b;;c", ",;")` is `["a", "b", "c"]`.
- `split_any_keep(s: Str, seps: Str) -> List[Str]`: The parts of `s` between any of the characters in `seps`, keeping empty parts, so the result has one more part than there are separators: `split_any_keep("a,,b", ",")` is `["a", "", "b"]` and `""` gives `[""]`, like the builtin `split`.
- `split_trim(s: Str, sep: Str) -> List[Str]`: The parts of `s` split at `sep`, each trimmed, with empty parts dropped: `split_trim(" a , b,,", ",")` is `["a", "b"]`.
- `split_n(s: Str, sep: Str, n: Int where _ >= 1) -> List[Str]`: `s` split at the first `n - 1` occurrences of `sep`, so there are at most `n` parts and the last one holds the rest: `split_n("a=b=c", "=", 2)` is `["a", "b=c"]`.
- `split_last(s: Str, sep: Str) -> Opt[(Str, Str)]`: `s` split at the last occurrence of `sep` into what comes before and after it; `none` when `sep` does not occur or is empty.
- `split_by_char_type(s: Str) -> List[Str]`: `s` split into runs of one character type: uppercase, lowercase, digits, whitespace, uncased letters and everything else (Commons `splitByCharacterType`): `"fooBar 42"` is `["foo", "B", "ar", " ", "42"]`.
- `split_by_char_type_camel(s: Str) -> List[Str]`: Like `split_by_char_type`, but an uppercase run followed by lowercase letters gives its last capital to them: `"ASFRules"` is `["ASF", "Rules"]` and `"fooBar"` is `["foo", "Bar"]`.
- `lines(s: Str) -> List[Str]`: The lines of `s`, split at `\n`, `\r\n` and `\r`, without the line endings.
- `join_non_empty(xs: List[Str], sep: Str) -> Str`: The strings of `xs` that are not empty, joined with `sep`.
- `chunks_of(s: Str, n: Int where _ >= 1) -> List[Str]`: `s` cut into pieces of `n` characters; the last piece may be shorter, and `""` gives `[]`.

### Prefixes, suffixes and differences

- `common_prefix(xs: List[Str]) -> Str`: The longest prefix every string in `xs` starts with; `""` for an empty list.
- `common_suffix(xs: List[Str]) -> Str`: The longest suffix every string in `xs` ends with; `""` for an empty list.
- `index_of_difference(a: Str, b: Str) -> Opt[Int]`: The index of the first character where `a` and `b` differ (the length of the shorter one when it is a prefix of the other); `none` when they are equal.
- `difference(a: Str, b: Str) -> Str`: The rest of `b` from the first character where it differs from `a` (Commons `difference`): `difference("abcde", "abxyz")` is `"xyz"`; `""` when they are equal.
- `remove_prefix(s: Str, prefix: Str) -> Str`: `s` without `prefix` at the start; `s` when it doesn't start with it.
- `remove_suffix(s: Str, suffix: Str) -> Str`: `s` without `suffix` at the end; `s` when it doesn't end with it.
- `ensure_prefix(s: Str, prefix: Str) -> Str`: `s` starting with `prefix`: unchanged when it already does, else `prefix + s`.
- `ensure_suffix(s: Str, suffix: Str) -> Str`: `s` ending with `suffix`: unchanged when it already does, else `s + suffix`.
- `starts_with_any(s: Str, prefixes: List[Str]) -> Bool`: Whether `s` starts with any of `prefixes`.
- `ends_with_any(s: Str, suffixes: List[Str]) -> Bool`: Whether `s` ends with any of `suffixes`.
- `starts_with_ci(s: Str, prefix: Str) -> Bool`: Whether `s` starts with `prefix`, ignoring case by full case folding (`fold_case`, so `"STRASSE"` starts with `"straß"`).
- `ends_with_ci(s: Str, suffix: Str) -> Bool`: Whether `s` ends with `suffix`, ignoring case by full case folding.
- `contains_ci(s: Str, sub: Str) -> Bool`: Whether `s` contains `sub`, ignoring case by full case folding.

### Safe substrings

- `left(s: Str, n: Int) -> Str`: The first `n` characters of `s`; `""` when `n <= 0`, all of `s` when `n` is larger.
- `right(s: Str, n: Int) -> Str`: The last `n` characters of `s`; `""` when `n <= 0`, all of `s` when `n` is larger.
- `mid(s: Str, pos: Int, n: Int) -> Str`: The `n` characters of `s` from `pos` (a negative `pos` counts as 0); `""` when `n <= 0` or `pos` is past the end.
- `substring(s: Str, start: Int, end: Int) -> Str`: The characters of `s` from `start` up to `end` (exclusive).
- `substring_from(s: Str, start: Int) -> Str`: The characters of `s` from `start` to the end; a negative `start` counts from the end.
- `substring_before(s: Str, sep: Str) -> Str`: What comes before the first `sep` in `s`; all of `s` when `sep` doesn't occur, `""` when `sep` is empty (Commons `substringBefore`).
- `substring_after(s: Str, sep: Str) -> Str`: What comes after the first `sep` in `s`; `""` when `sep` doesn't occur, all of `s` when `sep` is empty.
- `substring_before_last(s: Str, sep: Str) -> Str`: What comes before the last `sep` in `s`; all of `s` when `sep` doesn't occur or is empty.
- `substring_after_last(s: Str, sep: Str) -> Str`: What comes after the last `sep` in `s`; `""` when `sep` doesn't occur or is empty.
- `between(s: Str, open: Str, close: Str) -> Opt[Str]`: The text between the first `open` and the first `close` after it (Commons `substringBetween`): `between("a[b]c", "[", "]")` is `some("b")`.
- `between_all(s: Str, open: Str, close: Str) -> List[Str]`: Every text between an `open` and the next `close`, left to right without overlaps: `between_all("[a][b]", "[", "]")` is `["a", "b"]`; `[]` when `open` or `close` is empty.
- `chomp(s: Str) -> Str`: `s` without one line ending at the end (`\r\n`, `\n` or `\r`); `s` when it has none.
- `chop(s: Str) -> Str`: `s` without its last character, or without a final `\r\n` as a whole; `""` stays `""`.

### Character sets

- `strip_chars(s: Str, set: Str) -> Str`: `s` without the characters of `set` at both ends; an empty `set` strips whitespace (the builtin `trim`).
- `strip_start_chars(s: Str, set: Str) -> Str`: `s` without the characters of `set` at the start; an empty `set` strips whitespace.
- `strip_end_chars(s: Str, set: Str) -> Str`: `s` without the characters of `set` at the end; an empty `set` strips whitespace.
- `remove_chars(s: Str, set: Str) -> Str`: `s` without any of the characters in `set` (Commons `CharSetUtils.delete`).
- `keep_chars(s: Str, set: Str) -> Str`: Only the characters of `s` that are in `set` (Commons `CharSetUtils.keep`).
- `squeeze(s: Str, set: Str) -> Str`: `s` with every run of one repeated character from `set` squeezed to one: `squeeze("aabbcc", "ab")` is `"abcc"`.
- `count_chars(s: Str, set: Str) -> Int`: How many characters of `s` are in `set`.
- `digits_only(s: Str) -> Str`: Only the decimal digits of `s` (see `is_numeric`): `digits_only("(555) 123-4567")` is `"5551234567"`.
- `delete_whitespace(s: Str) -> Str`: `s` without any whitespace characters.
- `normalize_space(s: Str) -> Str`: `s` trimmed, with every run of whitespace inside it replaced by one space.
- `remove_all(s: Str, sub: Str) -> Str`: `s` without every occurrence of `sub`; `s` when `sub` is empty.

### Searching

- `indexes_of(s: Str, sub: Str) -> List[Int]`: The start index of every non-overlapping occurrence of `sub` in `s`, left to right; `[]` when `sub` is empty.
- `count_matches(s: Str, sub: Str) -> Int`: How many non-overlapping times `sub` occurs in `s` (Commons `countMatches`); 0 when `sub` is empty: `count_matches("aaaa", "aa")` is 2.
- `index_of_from(s: Str, sub: Str, from: Int) -> Opt[Int]`: The index of the first `sub` in `s` at or after `from`; a negative `from` counts as 0, and an empty `sub` is found at `from` while `from` is within the string.
- `last_index_of(s: Str, sub: Str) -> Opt[Int]`: The index of the last `sub` in `s`; an empty `sub` is found at `s.len`.
- `ordinal_index_of(s: Str, sub: Str, n: Int where _ >= 1) -> Opt[Int]`: The index of the `n`th occurrence of `sub` in `s`, counting overlapping occurrences as Commons `ordinalIndexOf` does (`ordinal_index_of("aaa", "aa", 2)` is `some(1)`); an empty `sub` is found at 0.
- `index_of_any(s: Str, chars: Str) -> Opt[Int]`: The index of the first character of `s` that is in `chars`.
- `index_of_any_but(s: Str, chars: Str) -> Opt[Int]`: The index of the first character of `s` that is not in `chars`.
- `last_index_of_any(s: Str, chars: Str) -> Opt[Int]`: The index of the last character of `s` that is in `chars`.
- `index_of_first(s: Str, subs: List[Str]) -> Opt[Int]`: The earliest index at which any of the non-empty strings in `subs` occurs in `s`.
- `contains_any(s: Str, chars: Str) -> Bool`: Whether `s` contains any character of `chars`.
- `contains_none(s: Str, chars: Str) -> Bool`: Whether `s` contains none of the characters of `chars`; true for an empty `s` or `chars`.
- `contains_only(s: Str, chars: Str) -> Bool`: Whether every character of `s` is one of `chars`; true for an empty `s`.
- `contains_any_of(s: Str, subs: List[Str]) -> Bool`: Whether `s` contains at least one of `subs`.
- `contains_all_of(s: Str, subs: List[Str]) -> Bool`: Whether `s` contains every one of `subs`; true for an empty list.

### Replacing

- `replace_once(s: Str, a: Str, b: Str) -> Str`: `s` with the first `a` replaced by `b`; `s` when `a` is empty or missing.
- `replace_last(s: Str, a: Str, b: Str) -> Str`: `s` with the last `a` replaced by `b`; `s` when `a` is empty or missing.
- `replace_n(s: Str, a: Str, b: Str, n: Int where _ >= 0) -> Str`: `s` with the first `n` non-overlapping occurrences of `a` replaced by `b`, left to right; `s` when `a` is empty or `n` is 0.
- `replace_each(s: Str, pairs: List[(Str, Str)]) -> Str`: `s` with every key of `pairs` replaced by its value in one pass (Commons `replaceEach`), so a replacement is never replaced again: `replace_each("ab", [("a", "b"), ("b", "a")])` is `"ba"`.
- `replace_chars(s: Str, from: Str, to: Str) -> Str`: `s` with each character of `from` replaced by the character at the same index of `to`, and characters of `from` beyond the end of `to` deleted (Commons `replaceChars`, like `tr`).
- `overlay(s: Str, x: Str, start: Int, end: Int) -> Str`: `s` with the characters from `start` to `end` replaced by `x` (Commons `overlay`): indexes are clamped to the string and swapped when `start > end`, so `overlay("abcdef", "zz", 2, 4)` is `"abzzef"` and an index past the end appends.
- `insert_at(s: Str, pos: Int, x: Str) -> Str`: `s` with `x` inserted before the character at `pos`, clamped to the string (0 prepends, `s.len` or more appends, negative counts from the end).

### Distance and similarity

- `levenshtein(a: Str, b: Str) -> Int`: The Levenshtein edit distance between `a` and `b`: the fewest single-character insertions, deletions and substitutions that turn one into the other, over code points.
- `levenshtein_within(a: Str, b: Str, limit: Int where _ >= 0) -> Opt[Int]`: The Levenshtein distance between `a` and `b` when it is at most `limit`, else `none`; it stops as soon as every path costs more, so a small limit is fast on long strings.
- `damerau_levenshtein(a: Str, b: Str) -> Int`: The Damerau-Levenshtein distance in its optimal string alignment form: like `levenshtein`, and swapping two adjacent characters costs 1, but no substring is edited twice (so `"ca"` to `"abc"` is 3).
- `hamming(a: Str, b: Str) -> Opt[Int]`: The Hamming distance, the number of positions where `a` and `b` differ; `none` unless they have the same length in code points.
- `jaro(a: Str, b: Str) -> F64`: The Jaro similarity of `a` and `b`, from 0.0 (nothing in common) to 1.0 (equal); two empty strings are 1.0 and one empty string is 0.0.
- `jaro_winkler(a: Str, b: Str) -> F64`: The Jaro-Winkler similarity: `jaro` raised for a common prefix of up to 4 characters with the usual scaling factor 0.1, from 0.0 to 1.0.
- `lcs_length(a: Str, b: Str) -> Int`: The length of the longest common subsequence of `a` and `b` (characters in order, not necessarily adjacent): `lcs_length("ABCBDAB", "BDCABA")` is 4.
- `longest_common_substring(a: Str, b: Str) -> Str`: The longest string that occurs in both `a` and `b`, the leftmost in `a` on a tie; `""` when they share no character.
- `similarity(a: Str, b: Str) -> F64`: How similar `a` and `b` are by edit distance, `1 - levenshtein / longer length`, from 0.0 to 1.0; two empty strings are 1.0.
- `closest(s: Str, candidates: List[Str]) -> Opt[Str]`: The candidate closest to `s` by Levenshtein distance, the first one on a tie; `none` for an empty list.

### Natural order

- `natural_compare(a: Str, b: Str) -> Int`: Compares `a` and `b` in natural order, with runs of ASCII digits compared as numbers, so `"file2" < "file10"`: negative, 0 or positive.
- `natural_sort(xs: List[Str]) -> List[Str]`: `xs` sorted in natural order (see `natural_compare`); the sort is stable.

### Wrapping and indentation

- `wrap_lines(s: Str, width: Int where _ >= 1, break_long: Bool) -> List[Str]`: `s` word-wrapped into lines of at most `width` display columns (see `display_width`).
- `wrap(s: Str, width: Int where _ >= 1) -> Str`: `s` wrapped to `width` columns with `\n` between lines, not breaking long words (WordUtils `wrap`); see `wrap_lines`.
- `wrap_with(s: Str, width: Int where _ >= 1, newline: Str, break_long: Bool) -> Str`: `s` wrapped to `width` columns with `newline` between lines; `break_long` cuts words wider than `width`.
- `unwrap(s: Str) -> Str`: `s` with the lines of each paragraph joined by single spaces; paragraphs are separated by blank lines and come back separated by one blank line (`"\n\n"`).
- `indent(s: Str, prefix: Str) -> Str`: `s` with `prefix` added at the start of every line that is not blank (Python `textwrap.indent`); line breaks are kept as they are.
- `dedent(s: Str) -> Str`: `s` with the whitespace prefix common to all of its non-blank lines removed (Python `textwrap.dedent`); blank lines become empty.
- `surround(s: Str, q: Str) -> Str`: `s` between two copies of `q`: `surround("a", "'")` is `"'a'"`.
- `surround_if_missing(s: Str, q: Str) -> Str`: `s` with `q` added at the start and at the end where it is missing.
- `unsurround(s: Str, q: Str) -> Str`: `s` without one `q` at each end when it starts and ends with `q` (and is long enough to hold both); otherwise `s`.

### Accents, transliteration and slugs

- `strip_accents(s: Str) -> Str`: `s` without diacritics: precomposed Latin letters with accents, and letters with a stroke (`ø`, `ł`, `đ`, `ħ`), become their base letters, from the Latin-1 Supplement, Latin Extended-A and B and Latin Extended Additional (so Vietnamese works), and combining marks U+0300 to U+036F are removed, which handles decomposed text.
- `to_ascii(s: Str, replacement: Str) -> Str`: `s` spelled in ASCII where it can be: accents stripped (`strip_accents`), ligatures and special letters expanded (`æ` to `ae`, `ß` to `ss`, `þ` to `th`), typographic quotes, dashes, spaces and the ellipsis made plain, and a few symbols spelled out (`©` to `(c)`, `€` to `EUR`).
- `slug_options() -> SlugOptions`: The default slug options: `-` between words, no length limit, lowercase.
- `slugify(s: Str) -> Str`: A URL slug of `s` with the default options: `slugify("Crème Brûlée & Co.")` is `"creme-brulee-and-co"`.
- `slugify_with(s: Str, o: SlugOptions) -> Str`: A URL slug of `s`: transliterated to ASCII (`to_ascii`), `&` spelled `and`, lowercased when `o.lowercase`, and every run of characters other than ASCII letters and digits replaced by `o.sep`, with none at the ends.
- `unique_slug(base: Str, taken: List[Str]) -> Str`: `base` if it is not in `taken`, else the first of `base-2`, `base-3`, ...

### Escaping and quoting

- `escape_html(s: Str) -> Str`: `s` with `&`, `<`, `>`, `"` and `'` replaced by character references (`&amp;`, `&lt;`, `&gt;`, `&quot;`, `&#39;`), safe in HTML text and in quoted attribute values.
- `escape_xml(s: Str) -> Str`: `s` with the five XML special characters escaped as `&amp;`, `&lt;`, `&gt;`, `&quot;` and `&apos;`.
- `unescape_html(s: Str) -> Str`: `s` with character references decoded: the HTML 4 named references (`&amp;`, `&eacute;`, `&nbsp;`, `&euro;` and the rest of the 252) plus `&apos;`, and decimal and hexadecimal ones (`&#233;`, `&#xE9;`).
- `unescape_xml(s: Str) -> Str`: `s` with the five XML entities and numeric character references decoded; anything else is left as it is.
- `escape_json(s: Str) -> Str`: `s` escaped for the inside of a JSON string: `"` and `\` get a backslash, `\b \f \n \r \t` use their short forms, other control characters and U+2028 and U+2029 (which break JavaScript) become `\u00XX` forms; everything else stays as UTF-8.
- `quote_json(s: Str) -> Str`: `s` as a JSON string literal: `escape_json(s)` between double quotes.
- `unescape_json(s: Str) -> Opt[Str]`: The text of a JSON string's inside with its escapes decoded (`\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t` and `\uHHHH` (four hex digits), with surrogate pairs combined); `none` for an unknown escape, a truncated one or an unpaired surrogate.
- `quote_shell(s: Str) -> Str`: `s` quoted as one POSIX shell word: unchanged when it only has safe characters, else in single quotes (Python `shlex.quote`).
- `quote_regex(s: Str) -> Str`: `s` with every regex metacharacter escaped, so `regex(quote_regex(s))` matches `s` literally.
- `quote_csv(s: Str) -> Str`: `s` as one CSV field (RFC 4180): in double quotes, with each `"` doubled, when it contains a comma, a double quote, a carriage return or a line feed; otherwise unchanged.
- `percent_encode(s: Str) -> Str`: `s` percent-encoded for a URL component (RFC 3986): every byte but the unreserved characters becomes `%XX`.
- `percent_decode(s: Str) -> Opt[Str]`: `s` with `%XX` escapes decoded; `none` when a `%` is not followed by two hex digits or the bytes are not valid UTF-8.

## Unicode notes

- Grapheme clusters approximate UAX 29: base characters with combining marks (U+0300 to U+036F and the other common mark blocks, plus Hebrew, Arabic, Devanagari, Bengali and Thai marks and kana voicing marks), variation selectors, emoji modifiers, tag sequences and ZWJ emoji sequences stay together, regional indicators pair into flags, CR LF is one cluster and Hangul jamo join into syllables. Prepend characters and the Indic conjunct rule are not modelled (Devanagari `स्ते` is two clusters here, one in Unicode 15.1).
- Display width follows the East Asian Width wide and fullwidth ranges and counts emoji and flags as 2; ambiguous-width characters count 1.
- `strip_accents` covers Latin (Latin-1 Supplement, Latin Extended-A and B, Latin Extended Additional) and combining marks; `to_ascii` adds ligatures, special letters, typographic punctuation and a few symbols. Other scripts are not transliterated.

## Out of scope

- Unicode normalization (NFC, NFD, NFKC); `strip_accents` handles precomposed Latin and decomposed text, which covers the common cases.
- Locale-specific casing (Turkish dotted i, Lithuanian) and collation: use the builtin `Locale` (`compare`, `sort`).
- Full UAX 29 grapheme, word and sentence segmentation, and the UAX 14 line breaking algorithm; `wrap` breaks at whitespace.
- Transliteration of non-Latin scripts (Cyrillic, Greek, CJK); slugs drop what `to_ascii` can't spell.
- Phonetic encodings (Soundex, Metaphone) and the more exotic Commons Text distances (cosine, Jaccard over n-grams); `levenshtein`, `damerau_levenshtein`, `jaro_winkler` and `lcs_length` cover the usual needs.
- Text templating and string substitution (`StringSubstitutor`): SSPUR has interpolation.
- Number and date formatting: the builtin `format` and `Locale` do that.

## Install

```
sspur add strings
```

MIT licensed. Copyright (c) 2026 Utkarsha Vardhana.
