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
    benchmarked with both `fussy-filter-by-scoring` and `fussy-filter-default`
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

| Style                                            | Time (s) | #GC | GC time (s) |   Rel |
|--------------------------------------------------|---------:|----:|------------:|------:|
| `basic`                                          | 0.057035 |   0 |         0.0 |  0.50 |
| `fussy`/`fzf-native` (`fussy-filter-default`)    | 0.089633 |   0 |         0.0 |  0.78 |
| `fussy`/`fzf-native` (`fussy-filter-by-scoring`) | 0.101029 |   0 |         0.0 |  0.88 |
| `hotfuzz`                                        | 0.115202 |   0 |         0.0 |     1 |
| `orderless` (default)                            | 0.310777 |   0 |         0.0 |  2.70 |
| `fussy`/`fuz-bin`                                | 0.486535 |   0 |         0.0 |  4.22 |
| `substring`                                      | 0.792698 |   0 |         0.0 |  6.88 |
| `orderless` (flex)                               | 0.846879 |   0 |         0.0 |  7.35 |
| `fussy`/`flx`                                    | 1.004024 |   2 |    0.139745 |  8.72 |
| `flex`                                           | 2.247146 |   0 |         0.0 | 19.51 |
| `fussy`/`flx-rs`                                 | 4.026389 |   0 |         0.0 | 34.95 |

where the "Rel" column indicates the relative slowdown factor
compared to the fastest sorting fuzzy style.

## Conclusion

Hotfuzz is pretty fast.

`fussy` paired with the `fzf-native` dynamic module is the only configuration
that beats `hotfuzz` on this workload.

`orderless` is fast in its default configuration; switching to
`orderless-flex` for fuzzy matching costs roughly 2.7×.

[hotfuzz]:https://github.com/axelf4/hotfuzz
[fussy]: https://github.com/jojojames/fussy
[flx]: https://github.com/lewang/flx
[flx-rs]: https://github.com/jcs-elpa/flx-rs
[fzf-native]: https://github.com/dangduc/fzf-native
[fzf]: https://github.com/junegunn/fzf
[fuz]: https://github.com/rustify-emacs/fuz.el
[LiquidMetal]: https://github.com/rmm5t/liquidmetal
[sublime_fuzzy]: https://github.com/Schlechtwetterfront/fuzzy-rs
[orderless]: https://github.com/oantolin/orderless
