# Emacs fuzzy completion style benchmark

This is a benchmark intended for measuring the relative performance
of a set of fuzzy completions styles for Emacs.

The styles included in this benchmark are

* The builtin `basic` style.
  
  Prefix completion implemented in C.
  Not fuzzy in the slightest and only included as a baseline.
* The builtin `substring` style.

  Matches if the completion contains a contiguous substring of the search string.
* The builtin `flex` style.
  
  "Greedy" fuzzy algorithm in Emacs Lisp.
* [`hotfuzz`][hotfuzz] with its dynamic module loaded.

  Unlike with other dynamic modules
  that are consulted *for each* individual each candidate to compute its score,
  the Hotfuzz Lisp code only calls out to the native code once,
  passing along the entire completions list.
  This reduces overhead and enables multithreaded filtering and scoring.
* [`fussy`][fussy]

  Fussy is generic over different scoring backends,
  the following of which are benchmarked:
  
  - [flx], an Emacs Lisp library for fuzzy matching
  - [flx-rs], a Rust dynamic module reimplementation of `flx`
  - [fzf-native] which is a dynamic module implementing the [fzf] algorithm,
    benchmarked in two modes × two filter strategies (four rows total):
    - *Full scoring* — per-candidate fuzzy-DP via `fzf_get_score`.
    - *Filter-only* (`fussy-filter-only` pseudo-style) — swaps in
      `fzf_has_match` (~5× cheaper boolean check) and skips top-K sorting.
      See [`fzf-native-filter-only-min-pool`][fzf-native-filter-only] for
      the per-session knob; the benchmark forces it to `1` so every run
      takes the filter-only path regardless of pool size.

    Each mode is benchmarked with both `fussy-filter-by-scoring` (no
    upstream filter — scorer runs on every candidate) and
    `fussy-filter-default` (regex pre-filter via `completion-regexp-list`).
  - [fuz], a dynamic module implementing skim's algorithm (or clangd's!)
  
  The [LiquidMetal] and [sublime_fuzzy] backends had to be excluded
  due to them erroring out on the benchmark input.
  
  To make timings comparable to other the styles,
  the following options were set
  
  - `fussy-use-cache` to `nil`,
    since the benchmark tries completing with the same input many times
  - `fussy-filter-fn` to `#'fussy-filter-default` since it is faster than the current default
  - `fussy-compare-same-score-fn` to `nil`, and
  - `fussy-score-threshold-to-filter-alist` to `nil` 
    both since other styles do not implement this
* [`orderless`][orderless]
  
  Not fuzzy by default, but somewhat interesting to include
  just for reference given its popularity.
  Benchmarked with two configurations of `orderless-matching-styles`:
  the default (`orderless-literal` + `orderless-regexp`)
  and `orderless-flex` for fuzzy-style matching.
  
The benchmark consists of completing against a list of candidate completions
of length 95653, with the median length of each string being 37.
About 30% are interned symbols taken from `obarray` of an Emacs session,
and the rest are relative file paths.
The search strings are of length between one and ten.

Case-insensitive matching is used,
i.e. `completion-ignore-case` is set to `t`.

To reproduce,
compile the Hotfuzz dynamic module as instructed in its README
and place the resulting dynamic library in this directory.
The other dynamic module packages ship precompiled binaries.
Then run
```sh
./runbench
```
in a shell from within this directory.

## Results

Running the benchmark on an Apple M5 in Emacs 31.0.50
the resulting times were

| Style                                                          | Time (s) | #GC | GC time (s) |   Rel |
|----------------------------------------------------------------|---------:|----:|------------:|------:|
| `basic`                                                        | 0.054805 |   0 |         0.0 |  0.47 |
| `fussy`/`fzf-native` (filter-only, `fussy-filter-default`)     | 0.090381 |   0 |         0.0 |  0.78 |
| `fussy`/`fzf-native` (filter-only, `fussy-filter-by-scoring`)  | 0.094981 |   0 |         0.0 |  0.82 |
| `fussy`/`fzf-native` (`fussy-filter-default`)                  | 0.096279 |   0 |         0.0 |  0.83 |
| `fussy`/`fzf-native` (`fussy-filter-by-scoring`)               | 0.105952 |   0 |         0.0 |  0.91 |
| `hotfuzz`                                                      | 0.115911 |   0 |         0.0 |     1 |
| `orderless` (default)                                          | 0.306053 |   0 |         0.0 |  2.64 |
| `fussy`/`fuz-bin`                                              | 0.432805 |   0 |         0.0 |  3.73 |
| `substring`                                                    | 0.772593 |   0 |         0.0 |  6.67 |
| `orderless` (flex)                                             | 0.841234 |   0 |         0.0 |  7.26 |
| `fussy`/`flx`                                                  | 0.910818 |   2 |    0.134576 |  7.86 |
| `flex`                                                         | 2.183582 |   0 |         0.0 | 18.84 |
| `fussy`/`flx-rs`                                               | 3.995279 |   0 |         0.0 | 34.47 |

where the "Rel" column indicates the relative slowdown factor
compared to `hotfuzz` (the fastest fuzzy style that performs
score-based sorting).

## Conclusion

Hotfuzz is pretty fast.

`fussy` paired with the `fzf-native` dynamic module is the only
configuration that beats `hotfuzz` on this workload. In the filter-only
mode, both filter strategies fit between `basic` and the full-scoring
variants — at this candidate count (~95k) the absolute win over full
scoring is small (5–11 ms; 6–10%), but it grows linearly with pool
size because filter-only's per-candidate inner loop is ~5× cheaper.
The intended sweet spot is much larger pools (500k+) where the swap
pays off.

`orderless` is fast in its default configuration; switching to
`orderless-flex` for fuzzy matching costs roughly 2.6×.

[hotfuzz]:https://github.com/axelf4/hotfuzz
[fussy]: https://github.com/jojojames/fussy
[flx]: https://github.com/lewang/flx
[flx-rs]: https://github.com/jcs-elpa/flx-rs
[fzf-native]: https://github.com/dangduc/fzf-native
[fzf-native-filter-only]: https://github.com/dangduc/fzf-native#filter-only-mode
[fzf]: https://github.com/junegunn/fzf
[fuz]: https://github.com/rustify-emacs/fuz.el
[LiquidMetal]: https://github.com/rmm5t/liquidmetal
[sublime_fuzzy]: https://github.com/Schlechtwetterfront/fuzzy-rs
[orderless]: https://github.com/oantolin/orderless
