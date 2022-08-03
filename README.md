# Amsterdam notes

This is the research notebook for my FWF project in Amsterdam / Innsbruck.


2022-08-03
----------

To benchmark the time that Rust needs to read files:

~~~
fn main() -> std::io::Result<()> {
    for arg in std::env::args().skip(1) {
        let _s = std::fs::read_to_string(&arg)?;
    }
    Ok(())
}
~~~

And when we're already at it:
Here is a little program that is a light version of `ts` ---
it timestamps each line from stdin with the time elapsed since the
start of the program.
I wrote it because `ts` performs unconditional line buffering,
which hurts performance when writing to files, for example.

~~~
use std::{io, time};

fn main() -> io::Result<()> {
    let stdout = io::stdout().lock();
    if atty::is(atty::Stream::Stdout) {
        real_main(stdout)
    } else {
        real_main(io::BufWriter::new(stdout))
    }
}

fn real_main(mut stdout: impl io::Write) -> io::Result<()> {
    let now = time::Instant::now();
    for line in io::stdin().lines() {
        let line = line?;
        let elapsed = now.elapsed().as_secs_f64();
        writeln!(stdout, "{elapsed} {line}")?;
    }
    Ok(())
}
~~~

Only dependency: `atty` 0.2.14



2022-06-15
----------

The equality relation on terms in Dedukti does
neither consider the names of variables nor the domain of abstractions.
I implemented the same behaviour in Kontroli, yielding the following code:

~~~ rust
impl<V, Tm: PartialEq> PartialEq for Comb<V, Tm> {
    fn eq(&self, other: &Self) -> bool {
        match (self, other) {
            (Self::Appl(head1, args1), Self::Appl(head2, args2)) => {
                head1 == head2 && args1 == args2
            }
            (Self::Abst(_, _, tm1), Self::Abst(_, _, tm2)) => tm1 == tm2,
            (Self::Prod(_, ty1, tm1), Self::Prod(_, ty2, tm2)) => ty1 == ty2 && tm1 == tm2,
            _ => false,
        }
    }
}
impl<V, Tm: Eq> Eq for Comb<V, Tm> {}
~~~

I compiled this to a binary `kocheck-impleq` and called the version that instead
automatically derives `Eq` `kocheck-deriveeq`.
The evaluation results on my ThinkPad X230 are as follows:

~~~
$ hyperfine -w1 -L ver impl,derive "target/release/kocheck-{ver}eq koeval-itp/isabelle_hol/isaexport.dk --eta"
Benchmark #1: target/release/kocheck-impleq koeval-itp/isabelle_hol/isaexport.dk --eta
  Time (mean ± σ):     227.213 s ±  6.534 s    [User: 220.402 s, System: 3.935 s]
  Range (min … max):   220.843 s … 240.934 s    10 runs

Benchmark #2: target/release/kocheck-deriveeq koeval-itp/isabelle_hol/isaexport.dk --eta
  Time (mean ± σ):     221.569 s ±  0.806 s    [User: 219.135 s, System: 2.417 s]
  Range (min … max):   220.220 s … 222.593 s    10 runs
~~~

Interestingly, doing *less* work for the comparison seems to *increase* the time.
Perhaps this is because the manual `Eq` makes more terms equal,
which could change the behaviour of some algorithms.

Furthermore, the variance of the version using the manual `Eq` implementation is much higher.
No idea why. (I made one warm-up run with `-w1`.)

So for now, I stay with the automatically derived `Eq`.


2021-12-07
----------

To generate the sizes of Dedukti commands:

    for i in hol_stdlib_u/*.dk; do kofmt < $i | awk '{print length($0)}'; done > hol.lines

Integrating the data:

    sort -n hol.lines | awk '{print NR " " $1}' > hol.integrated

Compressing the data ---
the first `awk` command quantises the data,
the second removes redundant data points:

    cat hol.integrated \
    | awk '{print $1 " " exp(int(log($2) * 100) / 100)}' \
    | awk '{a[$2] = $1} END {for (i in a) print a[i] " " i}' \
    | sort -n

Did I mention that I love `awk`? :)


2021-11-18
----------

To partially answer the first question from my last post:
nonaCoP is as fast as meanCoP on CNF problems.
To measure this, I evaluated meanCoP and nonaCoP on TPTP 6.3.0 problems.
Within a timeout of 1 second, meanCoP solves 881 and nonaCoP 880 CNF problems,
both using conjecture-directed search and no cuts.
To find out how many CNF problems are solved by both together:

    comm \
      <(cat solved/TPTP-v6.3.0/1s/meancop--conj-n | grep "-") \
      <(cat solved/TPTP-v6.3.0/1s/meancop--conj   | grep "-") | wc -l

This yields 883 CNF problems, meaning that
nonaCoP and meanCoP solve the same problems.

On FOF problems, the image looks different:
meanCoP solves 1576 and nonaCoP only 1554 problems.
Together, they solve 1650 problems (+4.7% compared to meanCoP).
That means that not only nonaCoP solves fewer problems in total,
but it also solves different problems than meanCoP.

My intuition was that the nonclausal proof search should always
take equally many or fewer inferences than clausal proof search.
But that does not seem to be the case:
I found some problems where nonclausal search
takes *more* inferences than clausal search.
That might be due to different order of processing clauses.

My goal is now the following:
Find a way to make nonaCoP always find a proof
after at most as many inferences as meanCoP.
Once this is achieved, this should give us
better means to compare clausal and nonclausal proof search.


2021-11-16
----------

Last Friday, I have finished the implementation of
my first nonclausal automated theorem prover in Rust! 🎉
Get it [here](https://github.com/01mf02/cop-rs/).

This nonclausal automated theorem prover is integrated in meanCoP.
It can be used by passing the `-n` flag to meanCoP.
For brevity, I will refer to it as *nonaCoP* from now on,
and by *meanCoP*, I will describe only the clausal version.

nonaCoP is based in spirit on Jens Otten's [nanoCoP](http://leancop.de/nanocop/),
another nonclausal automated theorem prover.
However, there are at the moment a few important differences:

* nonaCoP currently only considers one extension clause per contrapositive, whereas
  nanoCoP backtracks over different extension clauses for the same contrapositive,
  starting with the smallest.
  My approach is a kind of MVP of nonclausal proof search,
  yet I am not aware of any literature describing my approach.
  My approach resembles clausal proof search with Tseitin's transformation,
  but I am not completely sure yet about the exact relation between the two.
* nonaCoP implements a new type of cut, namely cut on decomposition steps.
  This is a change that can be implemented in nanoCoP easily.
  I show in the table below that it can improve the number of solved problems by 4.4%.

I evaluated the preliminary result of nonaCoP and compare them with meanCoP,
which performs roughly the same proof steps as leanCoP.
I chose as problem dataset the MPTP2078 bushy problems.
I performed the evaluation on the colossus cluster of the University of Innsbruck.

Table: Solved bushy problems by meanCoP with 10 seconds timeout.

| Conj | Nonclausal | Cuts  | Solved |
| ---- | ---------- | ----- | ------ |
|      | ✔          |       |    544 |
| ✔    | ✔          |       |    549 |
| ✔    |            |       |    552 |
| ✔    |            | REX   |    850 |
| ✔    | ✔          | REX   |    758 |
| ✔    | ✔          | REXDX |    791 |

There is hardly a difference in number of solved problems between meanCoP and nonaCoP
when using cut-free proof search (3 problems more solved by meanCoP).
However, nonaCoP proves 30 problems that meanCoP could not solve, and
meanCoP proves 33 problems that nonaCoP could not solve.
Together, they thus prove 6.0% more problems than meanCoP alone.

On the other hand, when using cuts,
the current nonaCoP version is not yet en par with meanCoP.
Using the same configuration (conjecture-directed search, REX cuts),
meanCoP proves 92 problems more than nonaCoP.
However, when using exclusive cut on decomposition steps (the "DX" of "REXDX"),
the advantage of meanCoP shrinks to 59 problems.
Furthermore, nonaCoP with REXDX cuts proves
46 problems that meanCoP with REX cannot prove.

Now there are several things to do:

* Find out why meanCoP performs better than nonaCoP.
  Is it due to raw performance or is there a theoretical reason?
* Implement backtracking over different extension clauses for the same contrapositive.
  This also opens the door towards a new extension:
  nanoCoP, as far as I understand it, restricts
  extensions into extension clauses on the same path to
  *maximally one* for each contrapositive, namely the most recently put onto the path.
  However, studying the reconstruction of connection proofs has convinced me that
  it makes sense to lift this restriction, allowing extensions into
  arbitrarily many extension clauses on the same path.
* Implement different extension clause orders, as already done
  [during my PhD](https://github.com/01mf02/notes#nanocop-extension-clause-order).


2021-11-05
----------

The way I currently generate nonclausal beta-clauses is relatively ugly,
because it involves deep-cloning clauses.
That is why I studied generating beta-clauses lazily using iterators.
As test vehicle, I used the clausal version of meanCoP.

I compared six versions that use as iterators:

* `iter.take(1).chain(iter.skip(1))` (cts),
* `iter.enumerate().filter_map(|x| Some(x.1) })` (efm),
* `iter.enumerate().filter_map(|x| if x.0 == 10000000000 { None } else { Some(x.1) })` (efm2),
* `SkipSingle::new(10000000000, iter)` (ss),
* `iter.skip(0)` (skip),
* `iter` (bare).

The code for `SkipSingle` is:

~~~ rust
#[derive(Clone)]
struct SkipSingle<I> {
    iter: I,
    pos: usize,
    skip: usize,
}

impl<I> SkipSingle<I> {
    fn new(skip: usize, iter: I) -> Self {
        Self {
            pos: 0, iter, skip,
        }
    }
}

impl<I: Iterator> Iterator for SkipSingle<I> {
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        if self.pos == self.skip {
            self.pos += 1;
            self.iter.next();
            self.next()
        } else {
            self.pos += 1;
            self.iter.next()
        }
    }
}
~~~

Previously, I always used `Skip<I>`, but only this experience made me realise
that it was actually superfluous, and I should have been using `I` from the start.

The evaluation results are:

~~~
$ hyperfine -w 1 -L version cts,efm,efm2,ss,skip,bare -m 5 "target/release/meancop-{version} eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex"
Benchmark #1: target/release/meancop-cts eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.820 s ±  0.136 s    [User: 7.819 s, System: 0.001 s]
  Range (min … max):    7.708 s …  8.053 s    5 runs
 
Benchmark #2: target/release/meancop-efm eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.377 s ±  0.027 s    [User: 7.373 s, System: 0.002 s]
  Range (min … max):    7.343 s …  7.409 s    5 runs
 
Benchmark #3: target/release/meancop-efm2 eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.397 s ±  0.021 s    [User: 7.391 s, System: 0.003 s]
  Range (min … max):    7.379 s …  7.428 s    5 runs
 
Benchmark #4: target/release/meancop-ss eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.353 s ±  0.026 s    [User: 7.348 s, System: 0.003 s]
  Range (min … max):    7.324 s …  7.380 s    5 runs
 
Benchmark #5: target/release/meancop-skip eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.349 s ±  0.053 s    [User: 7.345 s, System: 0.001 s]
  Range (min … max):    7.288 s …  7.430 s    5 runs
 
Benchmark #6: target/release/meancop-bare eval/i/bushy/relat_1__t203_relat_1.p --conj --cuts rex
  Time (mean ± σ):      7.180 s ±  0.075 s    [User: 7.179 s, System: 0.000 s]
  Range (min … max):    7.097 s …  7.297 s    5 runs
~~~

The used version of the iterators has a non-negligible impact on performance.
Especially the cts version is significantly slower than the other versions.
Using no iterator adaptors (bare) is the fastest,
but it requires also the most code and the most memory.
The rank between the variants efm, efm2, ss, varies greatly; in a second evaluation,
efm2 < efm < ss, whereas in this evaluation,
ss < efm < efm2.
It seems that efm2 is a nice compromise between code size and performance.


2021-09-21
----------

To obtain `kocheck.string`, I replaced `parse::<&str>` in `kocheck/src/seq.rs`
by `parse::<String>`.

On the HOL Light dataset on the cluster:

~~~
Benchmark #1: make -s -C hol_stdlib_u KOCHECK=kocheck KOFLAGS='--omit share' kontroli
  Time (mean ± σ):     21.299 s ±  0.128 s    [User: 20.009 s, System: 1.235 s]
  Range (min … max):   21.152 s … 21.385 s    3 runs
 
Benchmark #2: make -s -C hol_stdlib_u KOCHECK=../../kontroli-rs/target/release/kocheck.string KOFLAGS='--omit share' kontroli
  Time (mean ± σ):     28.352 s ±  0.114 s    [User: 27.195 s, System: 1.112 s]
  Range (min … max):   28.268 s … 28.481 s    3 runs

Benchmark #3: make -s -C hol_stdlib_u KOCHECK=kocheck KOFLAGS='--omit share -c' kontroli
  Time (mean ± σ):     96.423 s ±  4.697 s    [User: 113.138 s, System: 35.861 s]
  Range (min … max):   92.789 s … 101.726 s    3 runs
~~~


2021-09-06
----------

I compared the performance of Kontroli compiled with Rust 1.53 and 1.54
on my local computer (Intel NUC).

Rust 1.53:

~~~
Benchmark #1: make KOFLAGS="" lists.koo
  Time (mean ± σ):     31.296 s ±  0.140 s    [User: 30.909 s, System: 0.387 s]
  Range (min … max):   31.149 s … 31.429 s    3 runs

Benchmark #2: make KOFLAGS="-j2" lists.koo
  Time (mean ± σ):     22.667 s ±  0.043 s    [User: 41.691 s, System: 1.560 s]
  Range (min … max):   22.629 s … 22.714 s    3 runs
~~~

Rust 1.54:

~~~
Benchmark #1: make KOFLAGS="" lists.koo
  Time (mean ± σ):     30.654 s ±  0.199 s    [User: 30.227 s, System: 0.428 s]
  Range (min … max):   30.488 s … 30.875 s    3 runs

Benchmark #2: make KOFLAGS="-j2" lists.koo
  Time (mean ± σ):     22.569 s ±  0.137 s    [User: 41.521 s, System: 1.551 s]
  Range (min … max):   22.438 s … 22.711 s    3 runs
~~~

We see improvements for both single- and multi-threaded versions
(with larger improvement for the single-threaded version).
Therefore, I use Rust 1.54 from now on.

I benchmarked Kontroli on a cluster provided by the University of Innsbruck.
The initial results are quite astounding, because they suggest that
multi-threaded checking on the cluster has a much lower impact than
on my local computer.

On the cluster:

~~~
Benchmark #1: make KOFLAGS="" lists.koo
  Time (mean ± σ):     18.014 s ±  0.221 s    [User: 17.591 s, System: 0.382 s]
  Range (min … max):   17.806 s … 18.515 s    10 runs

Benchmark #2: make KOFLAGS="-j2" lists.koo
  Time (mean ± σ):     15.886 s ±  0.650 s    [User: 27.199 s, System: 1.347 s]
  Range (min … max):   15.150 s … 17.343 s    10 runs
~~~

First, we see much larger fluctuations on the cluster.
Second, while the runtime on my local computer decreases by 26.4%,
on the cluster it decreases by only 11.8%.
Why?

I suspected that thread creation overhead might be higher on the cluster.
That does not seem to be the case: I benchmarked the test code at
<https://stackoverflow.com/a/27764581>,
which executes on both cluster and local computer in around 8 seconds.

Anyway, on larger theories, the cluster still seems to yield nice results.
For example, for HOL Light, using four threads takes about 69% of the single-threaded time,
while in the Kontroli version evaluated for last year's CPP,
it took 91% of the time.
Interestingly, using parallel parsing (option `-c`) makes the checker slower.

~~~
Benchmark #1: make KOFLAGS="" cart.koo
  Time (mean ± σ):     226.201 s ±  0.186 s    [User: 221.066 s, System: 4.994 s]
  Range (min … max):   226.053 s … 226.411 s    3 runs

Benchmark #2: make KOFLAGS="-j2" cart.koo
  Time (mean ± σ):     195.693 s ±  1.318 s    [User: 335.184 s, System: 15.886 s]
  Range (min … max):   194.477 s … 197.094 s    3 runs

Benchmark #3: make KOFLAGS="-j3" cart.koo
  Time (mean ± σ):     165.410 s ±  0.730 s    [User: 377.517 s, System: 45.115 s]
  Range (min … max):   164.680 s … 166.139 s    3 runs

Benchmark #4: make KOFLAGS="-j4" cart.koo
  Time (mean ± σ):     157.298 s ±  4.222 s    [User: 440.417 s, System: 85.434 s]
  Range (min … max):   152.849 s … 161.249 s    3 runs

Benchmark #5: make KOFLAGS="-j5" cart.koo
  Time (mean ± σ):     152.622 s ±  1.266 s    [User: 501.496 s, System: 129.638 s]
  Range (min … max):   151.740 s … 154.073 s    3 runs

Benchmark #6: make KOFLAGS="-c -j5" cart.koo
  Time (mean ± σ):     167.110 s ±  0.540 s    [User: 581.040 s, System: 184.637 s]
  Range (min … max):   166.501 s … 167.534 s    3 runs

~~~


2021-07-30
----------

This Monday, upon waking up, I suddenly had an idea
how to improve the term data structure in Kontroli.
So far, I had closely reproduced the term structure of Dedukti,
which looks like this:

~~~ ocaml
type term =
  | Kind                                  (* Kind *)
  | Type                                  (* Type *)
  | DB    of int                          (* deBruijn *)
  | Const of name                         (* Global variable *)
  | App   of term * term list             (* f [ a1 ; ... an ] , f not an App *)
  | Lam   of ident * term option * term   (* Lambda abstraction *)
  | Pi    of ident * term * term          (* Pi abstraction *)
~~~

My corresponding take on it in Rust was originally:

~~~ rust
pub enum Term<C, V, Tm> {
    Kind,
    Type,
    DB(DeBruijn),
    Const(C),
    App(Tm, Vec<Tm>),
    Lam(Arg<V, Option<Tm>>, Tm),
    Pi(Arg<V, Tm>, Tm),
}
~~~

Using the `Tm` parameter, we can create terms that are shared or unshared:

~~~ rust
/// unshared term
pub struct BTerm<C, V>(Box<Term<C, V, BTerm>>);

/// shared term
pub struct RTerm<C, V>(Rc <Term<C, V, RTerm>>);
~~~

The shown term structure requires us to wrap
terms such as `Type`, `DB`, and `Const` (in a `Box` or `Rc`)
even though this does not benefit us, because
duplicating such values takes constant time
(as opposed to duplicating `App`, `Lam`, and `Pi`).
(At least if `C` is instantiated to an appropriate type.)
The overhead of superfluous wrapping shows
particularly with more expensive pointer types,
such as `Arc`, which becomes important when using multiple threads.

For that reason, I refactored the term structure to the following:

~~~ rust
pub enum Term<C, Tm> {
    Kind,
    Type,
    DB(DeBruijn),
    Const(C),
    Comb(Tm),
}

/// Combinator term.
pub enum TermC<V, Tm> {
    App(Tm, Vec<Tm>),
    Lam(Arg<V, Tm>, Tm),
    Pi(Arg<V, Option<Tm>>, Tm),
}
~~~

This allows us to instantiate terms as follows:

~~~ rust
pub type BTerm<C, V> = Term<C, BTermC<C, V>>;
pub type RTerm<C, V> = Term<C, RTermC<C, V>>;

pub struct BTermC<C, V>(Box<TermC<V, BTerm<C, V>>>);
pub struct RTermC<C, V>(Rc <TermC<V, RTerm<C, V>>>);
~~~

This makes the representation of atomic terms more compact;
what was `Rc::new(Term::Type)` in the old representation,
becomes just `Term::Type`.
However, it increases the representation of combinator terms;
what was `Rc::new(Term::App(tm, args))` before,
is now `Term::Comb(Rc::new(TermC::App(tm, args)))`.

I have benchmarked the effects of this change on datasets exported from
Isabelle/HOL, Matita, and HOL Light.
The new term structure is faster or equally fast in single-threaded mode,
and in multi-threaded mode, it is significantly faster.
We gain more performance in multi-threaded mode because
`Arc` is more costly than `Rc`, so
omitting instances of `Arc` pays off more than
omitting instances of `Rc`.

The evaluation results follow.
`kocheck` denotes the published version 0.2.0, and
`target/release/kocheck` is the development version using the new terms.

For Matita (STTfa):

~~~
$ hyperfine -w 1 -L ko kocheck,../../target/release/kocheck -L flags ,-j2 "make KOCHECK={ko} KOFLAGS={flags} kontroli"
Benchmark #1: make KOCHECK=kocheck KOFLAGS= kontroli
  Time (mean ± σ):     456.0 ms ±   4.8 ms    [User: 442.5 ms, System: 15.8 ms]
  Range (min … max):   449.9 ms … 464.3 ms    10 runs
 
Benchmark #2: make KOCHECK=../../target/release/kocheck KOFLAGS= kontroli
  Time (mean ± σ):     458.5 ms ±   5.9 ms    [User: 447.4 ms, System: 13.4 ms]
  Range (min … max):   450.9 ms … 466.5 ms    10 runs
 
Benchmark #3: make KOCHECK=kocheck KOFLAGS=-j2 kontroli
  Time (mean ± σ):     400.5 ms ±  12.0 ms    [User: 622.0 ms, System: 26.9 ms]
  Range (min … max):   387.2 ms … 421.3 ms    10 runs
 
Benchmark #4: make KOCHECK=../../target/release/kocheck KOFLAGS=-j2 kontroli
  Time (mean ± σ):     344.5 ms ±   8.5 ms    [User: 537.5 ms, System: 21.8 ms]
  Range (min … max):   339.4 ms … 368.0 ms    10 runs
~~~

For HOL Light (Theory U):

~~~
$ hyperfine -w 1 -m 5 -L ko kocheck,../../target/release/kocheck -L flags ,-j2 "make KOCHECK={ko} KOFLAGS={flags} pair.koo"
Benchmark #1: make KOCHECK=kocheck KOFLAGS= pair.koo
  Time (mean ± σ):      2.753 s ±  0.009 s    [User: 2.702 s, System: 0.052 s]
  Range (min … max):    2.744 s …  2.765 s    5 runs
 
Benchmark #2: make KOCHECK=../../target/release/kocheck KOFLAGS= pair.koo
  Time (mean ± σ):      2.649 s ±  0.016 s    [User: 2.598 s, System: 0.052 s]
  Range (min … max):    2.627 s …  2.669 s    5 runs
 
Benchmark #3: make KOCHECK=kocheck KOFLAGS=-j2 pair.koo
  Time (mean ± σ):      2.297 s ±  0.024 s    [User: 4.099 s, System: 0.221 s]
  Range (min … max):    2.259 s …  2.317 s    5 runs
 
Benchmark #4: make KOCHECK=../../target/release/kocheck KOFLAGS=-j2 pair.koo
  Time (mean ± σ):      1.986 s ±  0.008 s    [User: 3.544 s, System: 0.197 s]
  Range (min … max):    1.973 s …  1.992 s    5 runs
~~~

Isabelle/HOL (the first 20,000 lines):

~~~
$ hyperfine -w 1 -m 3 -L ko kocheck,../target/release/kocheck -L flags ,-j2 "{ko} {flags} --eta isaexport.20000.dk"
Benchmark #1: kocheck  --eta isaexport.20000.dk
  Time (mean ± σ):     25.573 s ±  0.181 s    [User: 25.449 s, System: 0.120 s]
  Range (min … max):   25.442 s … 25.780 s    3 runs
 
Benchmark #2: ../target/release/kocheck  --eta isaexport.20000.dk
  Time (mean ± σ):     20.424 s ±  0.069 s    [User: 20.265 s, System: 0.155 s]
  Range (min … max):   20.356 s … 20.494 s    3 runs
 
Benchmark #3: kocheck -j2 --eta isaexport.20000.dk
  Time (mean ± σ):     19.589 s ±  0.034 s    [User: 37.627 s, System: 0.811 s]
  Range (min … max):   19.551 s … 19.615 s    3 runs
 
Benchmark #4: ../target/release/kocheck -j2 --eta isaexport.20000.dk
  Time (mean ± σ):     13.892 s ±  0.085 s    [User: 26.533 s, System: 0.825 s]
  Range (min … max):   13.796 s … 13.958 s    3 runs
~~~

The Isabelle/HOL results are particularly astounding.
Compare the results with Dedukti:

~~~
Benchmark #1: dkcheck --eta isaexport.20000.dk
  Time (mean ± σ):     26.789 s ±  0.110 s    [User: 26.644 s, System: 0.139 s]
  Range (min … max):   26.692 s … 26.908 s    3 runs
~~~

Only with this change in term structure,
we finally get significantly below the runtime of Dedukti.
Furthermore, the parallel performance has improved enormously.


2021-07-12
----------

After reading [Temporarily opt-in to shared mutation](https://ryhl.io/blog/temporary-shared-mutation/),
I was curious to see whether I could replace
occurrences of `RefCell` by `Cell` in Kontroli.
I hoped for performance benefits, apart from the non-panicking guarantee.

Although I succeeded in replacing `RefCell`, my hope was disappointed.
`Cell` is slightly slower than `RefCell`, at least on
HOL Light's `lists.dk` as well as on the Sudoku example:

~~~
Benchmark #1: make lists.koo (RefCell)
  Time (mean ± σ):     21.336 s ±  0.123 s    [User: 21.054 s, System: 0.281 s]
  Range (min … max):   21.203 s … 21.445 s    3 runs

Benchmark #1: make lists.koo (Cell)
  Time (mean ± σ):     22.492 s ±  0.338 s    [User: 22.022 s, System: 0.343 s]
  Range (min … max):   22.278 s … 22.882 s    3 runs
~~~

`Cell` might be slower because to access its contents, we need to
replace its contents with a default value, then put the original value back.

Conclusion: I'll stay with `RefCell` for the time being.


2021-07-02
----------

The name for this notebook is now outdated.
I moved to Austria on the 30th of June.
Greetings from the sunny mountains! :)

Since [my last post](#2021-06-15), I worked on
integrating my new Dedukti parser into Kontroli.
While doing this, I realised that what I previously called "scoping"
could be split into a scoping and a sharing operation, where
scoping separates symbols into constants and variables, and
sharing maps equal constants to equal pointers.
This allows us to compare the parsing speed of Dedukti and Kontroli,
because parsing in Dedukti always includes scoping (but not sharing).
More importantly, because this makes scoping a pure, fail-safe operation,
we can execute it in parallel after parsing!

In the new scheme of scoping and sharing, it makes sense for the parser to
always output data structures containing `&str` (as opposed to `String`),
because this allows us to duplicate strings in constant time during scoping,
for example when we add a variable name to the stack of bound variables.

I implemented a single-threaded version of `kocheck` that uses the new parser.
I call this program `kocheck2` and evaluate it below on the
HOL Light export of `lists.ml` using Theory U (weighing 98MB).
I performed all evaluations in this post with my ThinkPad X230.

~~~
Benchmark #1: make clean && make lists.dko
  Time (mean ± σ):     28.508 s ±  0.107 s    [User: 27.402 s, System: 1.102 s]
  Range (min … max):   28.442 s … 28.631 s    3 runs

Benchmark #2: make clean && make KOCHECK=../target/release/kocheck lists.koo
  Time (mean ± σ):     28.129 s ±  0.241 s    [User: 27.478 s, System: 0.623 s]
  Range (min … max):   27.950 s … 28.403 s    3 runs

Benchmark #3: make clean && make KOCHECK=../target/release/kocheck2 lists.koo
  Time (mean ± σ):     22.200 s ±  0.090 s    [User: 21.887 s, System: 0.313 s]
  Range (min … max):   22.096 s … 22.255 s    3 runs
~~~

Benchmark #1 is Dedukti, #2 the old `kocheck`, and #3 the new `kocheck2`.
Everything is run using a single thread.
This shows that while Dedukti and the old `kocheck` are about equally fast,
the new `kocheck2` takes only 77.9% of the time that Dedukti takes.
This is a very nice result.

I now measure the performance of several stages of the new parser; first
only lexing,
then lexing & parsing, and
then lexing & parsing & scoping.
Finally, I evaluate Dedukti's lexing & parsing & scoping,
using [the `--beautify` trick](#2021-06-01).

~~~
Benchmark #1: make KOCHECK=../target/release/kocheck2 KOFLAGS=--no-parse lists.koo
  Time (mean ± σ):     606.7 ms ±   8.3 ms    [User: 558.8 ms, System: 49.9 ms]
  Range (min … max):   598.6 ms … 617.6 ms    4 runs

Benchmark #1: make KOCHECK=../target/release/kocheck2 KOFLAGS=--no-scope lists.koo
  Time (mean ± σ):      1.728 s ±  0.015 s    [User: 1.667 s, System: 0.063 s]
  Range (min … max):    1.712 s …  1.742 s    3 runs

Benchmark #1: make KOCHECK=../target/release/kocheck2 KOFLAGS=--no-share lists.koo
  Time (mean ± σ):      2.921 s ±  0.024 s    [User: 2.865 s, System: 0.057 s]
  Range (min … max):    2.895 s …  2.943 s    3 runs

Benchmark #1: for i in `make --silent -f kontroli.mk -f deps.mk lists.koo`; do dkcheck --beautify $i; done
  Time (mean ± σ):      6.233 s ±  0.047 s    [User: 6.169 s, System: 0.065 s]
  Range (min … max):    6.194 s …  6.289 s    5 runs
~~~

This shows that lexing is the fastest operation of the whole parser,
which is good news, because it is the only part that cannot be parallelised.
Furthermore, this shows that Dedukti's parser is
more than twice as slow than the new parser.

Next, I want to create a `kocheck2` which uses $m + n$ threads, where
$m$ is the number of parse/scope threads and $n$ is the number of check threads.
(Previously, this was limited to $m = 1$.)
This will give rise to a "doubly-parallel" proof checker.


2021-06-15
----------

I now implemented a parser for Dedukti commands,
which took me only about two hours!
This parser supports parsing to commands containing either `&str` or `String`,
in case the second should become necessary at some point
(concurrency, I'm looking at you).
Interestingly, parsing all of Isabelle/HOL with that parser
is about 1.5 seconds *faster* than my [previous experiment](#2021-06-13),
where I parsed only proof terms:

~~~
Benchmark #1: Single-threaded (full commands)
  Time (mean ± σ):     57.049 s ±  1.295 s    [User: 48.685 s, System: 7.652 s]
  Range (min … max):   55.905 s … 59.543 s    10 runs
~~~

The next task will be to integrate this parser into Kontroli, and to see
whether multi-threaded parsing can be used without conversion to `String`.


2021-06-13
----------

Several reviewers for my CPP paper criticised that
the performance gains by my proof checker Kontroli were
decent, but not exceptional.
I may just have found a way to change that.

My CPP evaluation showed that surprisingly,
the largest bottleneck during proof checking of large developments
(Isabelle/HOL, HOL Light) is parsing, not the actual checking itself.
For example, on my home computer, Kontroli takes 188 seconds
only to parse the Isabelle/HOL dataset, which is about 2.4GB large:

~~~
Benchmark #1: kocheck --buffer 512MB isaexport.dk --no-scope
  Time (mean ± σ):     187.562 s ±  2.194 s    [User: 181.851 s, System: 4.831 s]
  Range (min … max):   186.011 s … 189.113 s    2 runs
~~~

I created a new parser for the Dedukti format.
It is divided into a parser and a lexer,
unlike Kontroli's parser, whose parser lexes at the same time.
I realised that having a separate lexer allows me to parallelise parsing,
using Rust's Rayon crate.

The lexer is implemented using the [Logos](https://docs.rs/logos/) crate,
which made it possible to write the lexer (about 50 lines of code) in a single day.
The Logos crate lives up to its claim to create "ridiculously fast" lexers,
because lexing the Isabelle/HOL dataset with it takes only about 12 seconds
(including loading the file)!
One technique significantly reduced the lexing time:
In the tokens, save only references to the input string (`&str`)
instead of owned strings (`String`).

The parser currently only parses terms.
Still, we can already estimate its performance on the Isabelle/HOL dataset,
by just parsing the parts of commands after definitions (":=").
This parses all proof terms, which make up for the largest amount of the data.
The time (including file loading, lexing and parsing) is:

~~~
Benchmark #1: Single-threaded
  Time (mean ± σ):     58.668 s ±  1.327 s    [User: 49.299 s, System: 8.082 s]
  Range (min … max):   57.730 s … 59.606 s    2 runs

Benchmark #2: Multi-threaded, 2 threads
  Time (mean ± σ):     38.729 s ±  0.477 s    [User: 56.543 s, System: 12.088 s]
  Range (min … max):   38.391 s … 39.066 s    2 runs

Benchmark #3: Multi-threaded, 3 threads
  Time (mean ± σ):     37.611 s ±  0.534 s    [User: 78.632 s, System: 19.701 s]
  Range (min … max):   37.233 s … 37.989 s    2 runs
~~~

The results are amazing.
Going down from 188 seconds for the current parser to only 59 seconds
would be already cool enough (more than three times as fast!),
but it also looks like parallel parsing scales quite well,
reducing time further to 39 seconds at two threads.
For three threads, the performance gains are much smaller,
but this might be due to my computer having only two cores.
(And I have still not optimised the parser.
This is the very first version that is able to parse all terms.)

The next step will be two extend the parser to Dedukti *commands*
and to integrate it into Kontroli.


2021-06-01
----------

Given that my work on meanCoP is now mostly completed,
I remembered that I have still a paper sitting around that was rejected at CPP last year.
So I decided that I will finish this before attacking nanoCoP.
I finally mustered all my courage and looked at the reviews (nearly six months later!).
I agree with them, the submitted version is quite technical,
and it would be better to concentrate on "the meat", as one reviewer called it.
After having re-read the paper, I wondered about
the relative parsing performance between Dedukti and Kontroli.

Parsing is heavily intertwined with scoping in Dedukti.
However, I found out that scoping in Dedukti
means something different than in Kontroli:
In both cases, scoping distinguishes
bound variables (such as `x` in a lambda abstraction `x => x`) from
unbound symbols (such as `c` that is not in the scope of some `c => t`).
The difference is that scoping in `dkcheck` does *not* check
whether a symbol `c` has actually been defined before.
To see this, it suffices to run `dkcheck` with the `--beautify` flag,
which parses and scopes, but does not check:

    echo "a: c." | dkcheck --beautify --stdin mod

Here, `dkcheck` does not complain, whereas the following invocation of
`kocheck`, which parses and scopes, does:

    echo "a: c." | kocheck --no-check -

This makes it difficult to compare the parsing performance of Kontroli and Dedukti.

Despite this, it seems that `kocheck --no-check` is faster than `dkcheck --beautify`,
even if Kontroli's scoping performs more work.
For this experiment, I replaced in `api/processor.ml` the `handle_entry` function
in `MakeEntryPrinter` by:

~~~ ocaml
  let handle_entry env e =
    let (module Pp:Pp.Printer) = (module Pp.Make(struct let get_name () = Env.get_name env end)) in
    (* Pp.print_entry Format.std_formatter e *)
    ()
~~~

This disables the pretty printing of the theory, leaving only parsing and scoping.
I used the most recent Kontroli and Dedukti versions from GitHub at the time of writing.
`hyperfine` yields the following timings on `colo12`:

~~~
Benchmark #1: kontroli-rs/target/release/kocheck isabelle_hol/isaexport.dk --buffer 512MB --no-check
  Time (mean ± σ):     449.624 s ±  2.975 s    [User: 444.926 s, System: 4.496 s]
  Range (min … max):   447.520 s … 451.728 s    2 runs

Benchmark #1: ./dkcheck.native ~/isabelle_hol/isaexport.dk --beautify
  Time (mean ± σ):     533.667 s ±  5.084 s    [User: 529.605 s, System: 3.916 s]
  Range (min … max):   530.071 s … 537.262 s    2 runs
~~~

This shows that `kocheck` takes about 16% less time than `dkcheck` to
parse the Dedukti export of Isabelle's `HOL.List` theory.

The Dedukti results for `matita_sttfa`:

~~~
Benchmark #1: dkcheck --beautify *.dk
  Time (mean ± σ):     176.0 ms ±   2.5 ms    [User: 170.2 ms, System: 5.8 ms]
  Range (min … max):   173.2 ms … 183.9 ms    16 runs

Benchmark #1: make dedukti
  Time (mean ± σ):     772.9 ms ±  12.6 ms    [User: 692.0 ms, System: 79.7 ms]
  Range (min … max):   747.6 ms … 788.4 ms    10 runs

~~~

And for Kontroli:

~~~
Benchmark #1: KOFLAGS=--no-scope make kontroli
  Time (mean ± σ):     145.8 ms ±   3.3 ms    [User: 138.0 ms, System: 10.1 ms]
  Range (min … max):   140.9 ms … 153.8 ms    20 runs

Benchmark #1: KOFLAGS=--no-check make kontroli
  Time (mean ± σ):     237.7 ms ±   6.7 ms    [User: 224.9 ms, System: 15.1 ms]
  Range (min … max):   221.8 ms … 245.6 ms    13 runs

Benchmark #1: make kontroli
  Time (mean ± σ):     585.1 ms ±   7.5 ms    [User: 574.2 ms, System: 13.1 ms]
  Range (min … max):   573.4 ms … 597.7 ms    10 runs
~~~

We can see that Kontroli is overall faster than Dedukti, and that
 Dedukti's performance for parsing+scoping is between
Kontroli's performance for parsing and parsing+scoping.


2021-05-29
----------

I have worked on making meanCoP more modular,
by moving preprocessing and parsing code to its support library.
Furthermore, I have added CNF support.
Both changes should benefit the future nanoCoP implementation
as well as the web interface that is currently in development.


2021-05-22
----------

I have worked on releasing a first release of jaq, my jq clone.
This release is now 100% `todo!()`-free. :)
This implied mainly implementing optional path operations, such as
`.a?`, `.[]?`, and `.[i:j]?`.


2021-05-15
----------

I created scripts in the `eval` directory of meanCoP that semi-automate the
analysis of evaluation results (in particular generating tables and plots).
This should make it much easier for somebody else than me (or my future me ^^)
to reproduce the tables/plots featured in my TABLEAUX paper.

Using these scripts, I updated the data in the TABLEAUX paper.
The number of solved problems changed mostly for TPTP, where
REX solves 2102 problems before and 2126 after the `Vec` change (+1.1%).


2021-05-12
----------

I reran the evaluation with the new meanCoP, featuring `Vec`-based formulas.
After this, I noted that the new version was not able to prove
several problems within the timeout of ten seconds.
I obtained these problems via (ran inside `eval/solved`):

    for i in */10s/*; do echo $i; comm -13 $i ../solved.old/$i; done

As a result, I reran the evaluation for all those problems that were lost.
For this, I deleted the output of the lost files and
combined from the previous solved files a new set of solved files:

    for i in */10s/*; do for p in `comm -13 $i ../solved.old/$i`; do rm ../o/$i/$p; done; done
    for i in */10s/*; do cat $i ../solved.old/$i | sort > ../solved.merge/$i ; done

I then reran the evaluation without timeout [as described before](#2021-04-08).
This resulted in all previously lost problems being solved.
Only 15 problems (out of several hundreds of thousands evaluated)
surpassed a time limit of 15 seconds, as shown by:

    for i in */10s/*; do for p in `comm -13 $i ../solved.old/$i`; do
    jaq '.user | select(. > 15)' < ../o/$i/$p.time; done; done | wc -l

However, all previously lost problems were solved in less than 20 seconds.


2021-05-10
----------

I compared the number of inferences between leanCoP and meanCoP on several datasets:

    SETS="bushy chainy flyspeck-top miz40-deps.a15"
    for s in $SETS; do
      echo $s
      for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj`; do
        echo $i $(grep "Inf:" cop/eval/out/$s/jar/plleancop-171116-nodef-cut-conj/$i | cut -d " " -f 2);
      done > $s-plleancop.infs
      for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj`; do
        echo $i $(cat cop-rs/eval/o/$s/10s/meancop--conj--cutsrei/$i.stats | jaq '.infs | add');
      done > $s-meancop.infs
      diff -s $s-plleancop.infs $s-meancop.infs
    done

I noted a few problems in the flyspeck-top dataset that
were solved with differing numbers of inferences.
I found out that this was due to leanCoP's special handling of
symbols that are called `all` or `ex`.
I recalled that I had already
[found that behaviour during my PhD](https://github.com/01mf02/notes#leancop--nanocop-bugs),
but so far, the fix that Jens sent me at the time has not made it into the official leanCoP.
So here is the fix. It replaces the `univar` predicate in `def_mm.pl` with:

~~~ prolog
univar(X,_,X)  :- (atomic(X);var(X);X==[[]]), !.
univar(F,Q,F1) :-
     F=..[A,B|T], ( (A=ex;A=all),B=(X:C) -> delete2(Q,X,Q1),
     copy_term((X,C,Q1),(Y,D,Q1)), univar(D,[Y|Q],D1), F1=..[A,Y:D1] ;
     univar(B,Q,B1), univar(T,Q,T1), F1=..[A,B1|T1] ).
~~~

Apart from this, there are no diverging problems any more! :)


2021-05-07
----------

I found out that meanCoP cannot solve a few problems that leanCoP can solve.
These are all part of TPTP's `CSR` category (example: `CSR085+1`)
and consist of a large number of axioms.
During preprocessing, there is a stack overflow due to the structure of the
formula data type, which stores the conjunctions of these axioms
à la `Cj(a1, Cj(..., an)...)`.
As remedy, I introduced an array-based conjunction/disjunction in meanCoP
à la `Cj([a1, ... ,an])`.
This solved the stack overflow, but it required significant effort
to exactly reproduce the preprocessing of leanCoP again.
The formula ordering by paths and the conversion to CNF
were particularly nasty, taking me nearly one week.
While doing this work, I also discovered and eliminated
a small bug in my implementation of the regularity check,
which did not impact soundness, but sometimes caused unnecessary inferences.
Now the number of inferences of meanCoP on TPTP problems
perfectly matches those of leanCoP.
This is important for the soundness of the evaluation and
prepares the ground for me attacking a first prototype of nanoCoP in Rust.


2021-04-28
----------

I wanted to combine several lines of a table and divide the ratios of the cells.
For this, I first converted the tables from Markdown to JSON via <https://tableconvert.com>.
Then, I used commands such as the following to obtain a Markdown table of ratios:

    jq '[[.[3][]], [.[4][]]] | transpose | .[2:][] | .[1] / .[0] | . * 1000 | round | . / 10' | paste -sd '|'

The `transpose` filter is a clever generalisation of a `zip` function
that zips an arbitrary number of arrays (given themselves as an array).

Speaking of filters, I implemented user-defined filters in jaq.
This allows for the definition of filters such as:

    def transpose: [{ i: range([.[] | length] | max), a: . } | [.a[][.i]]];

This functionality allowed me to define 38 jq filters in very short time,
some of these replacing previously built-in filters.

The `round` filter is not yet implemented in jaq (which is why I use jq here),
but it is a natural candidate for being implemented soon.


2021-04-27
----------

For several datasets and provers, I wanted to obtain the cumulative time that
the prover takes to solve the problems solved by leanCoP.

Time for leanCoP:

    for s in bushy chainy flyspeck-top miz40-deps.a15 tptp630; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj`; do
    cat cop/eval/out/$s/jar/plleancop-171116-nodef-cut-conj/$i.time | grep "user" | cut -d "u" -f 1;
    done | jaq -s 'add';
    done
    for s in tptp630; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj | sed 's|\(...\)|\1/\1|'`; do
    cat cop/eval/out/$s/jar/plleancop-171116-nodef-cut-conj/$i.time | grep "user" | cut -d "u" -f 1;
    done | jaq -s 'add';
    done
    461.899999999999
    319.0799999999998
    2451.5799999999967
    9308.689999999799
    1299.6700000000035

Time for fleanCoP:

    for s in bushy chainy flyspeck-top miz40-deps.a15 tptp630; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj`; do
    cat cop/eval/out/$s/jar/streamcop-171126-nodef-cut-conj/$i.time | grep "user" | cut -d "u" -f 1;
    done | jaq -s 'add';
    done
    for s in tptp630; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj | sed 's|\(...\)|\1/\1|'`; do
    cat cop/eval/out/$s/jar/streamcop-171126-nodef-cut-conj/$i.time | grep "user" | cut -d "u" -f 1;
    done | jaq -s 'add';
    done
    190.86000000000016
    69.84000000000002
    657.1599999999828
    3845.620000000096
    488.10999999999746

Time for meanCoP:

    for s in bushy chainy flyspeck-top miz40-deps.a15; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj`; do
    cat cop-rs/eval/o/$s/10s/meancop--conj--cutsrei/$i.time;
    done | jaq -s '[.[].user] | add';
    done
    for s in tptp630; do
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj | sed 's|\(...\)|Problems/\1/\1|'`; do
    cat cop-rs/eval/o/TPTP-v6.3.0/10s/meancop--conj--cutsrei/$i.time;
    done | jaq -s '[.[].user] | add';
    done
    18.570000000000004
    19.73999999999997
    111.69000000000116
    393.4499999999931
    337.8099999999979

To round the numbers and to show them in a table-like format:

    for i in $(jq '. * 10 | round | . / 10'); do echo -n "| "$i; done

The final result, after some hand-formatting:

| Dataset      |   TPTP | bushy | chainy |  Miz40 | FS-top |
| -----------: | -----: | ----: | -----: | -----: | -----: |
|  leanCoP-REI | 1299.7 | 461.9 |  319.1 | 9308.7 | 2451.6 |
| fleanCoP-REI |  488.1 | 190.9 |   69.8 | 3845.6 |  657.2 |
|  meanCoP-REI |  337.8 |  18.6 |   19.7 |  393.4 |  111.7 |

This shows that meanCoP is vastly faster than both fleanCoP and leanCoP.


2021-04-24
----------

I wanted to highlight a number in a LaTeX table with bold letters.
However, the boldfaced numbers were larger than the other numbers,
rendering the table ugly.
I found out that the culprit was the combination of
the `lmodern` package with the `llncs` document class:
The `llncs` class makes the content of tables `\small`.
The `lmodern` package, however, does not seem to define an adequate font for
characters that are small *and* boldfaced at the same time,
resulting in the size mismatch mentioned.
My solution was to not load the `lmodern` package,
which I only used for providing a less "grainy" font than the default font.
Instead, I discovered that by installing the `cm-super` package,
the default font is replaced with a less "grainy" version.
This does not require loading any packages in LaTeX documents
and is thus clearly preferable to using `lmodern`.


2021-04-17
----------

I spent quite some time recently on generating statistics for each proof
in which steps that are REI/REX compatible are marked.
The idea is that if a proof contains only REX compatible proof steps,
then REX should be able to find that proof.
However, to this date, I have not been able to make it work correctly. :(
In particular, identifying REX compatible steps is terribly tricky ...
The simplest TPTP problem that my statistics identify as REX compatible,
but which REX actually cannot solve, is `PUZ031+1.p`
(as well as its `+2` and `+3` counterparts).
REI is simpler, but even there, there are a few problems that are not
correctly classified, such as the Miz40 problem `t30_trees_3.p`.

So I have opted (for now) for a simpler solution:
Whenever REX/REI find a proof that is completely identical to
a proof found by the complete strategy,
we know that the complete strategy used only REX/REI compatible proof steps.
This solution unfortunately does not allow an analysis of individual proof steps,
but it still provides interesting insights.
The results are:

| Dataset                         |    bushy |   chainy | flyspeck-top | miz40-deps.a15 | TPTP-v6.3.0 |
| ------------------------------- | -------: | -------: | -----------: | -------------: | ----------: |
| COMP proofs                     |      546 |      208 |         4038 |           9236 |        1711 |
| REX proofs in COMP              |      363 |      186 |         3272 |           7183 |        1443 |
| REI proofs in COMP              |      255 |      120 |         2723 |           5003 |        1175 |
| REX proofs                      |      850 |      294 |         4994 |          16134 |        2102 |
| REI proofs in REX               |      347 |      174 |         3326 |           8083 |        1339 |
| COMP infs for COMP & REX proofs | 45399093 | 21064243 |    503063533 |     1352758570 |   135693802 |
| REX infs for COMP & REX proofs  |  1210051 |  2108849 |     25430077 |       35639209 |    30412235 |
| COMP infs for COMP & REI proofs |  7148403 |  8754173 |    352551327 |      692280513 |    60448738 |
| REI infs for COMP & REI proofs  |   128641 |   272841 |     12247254 |       12781751 |    14193952 |
| REX infs for REX & REI proofs   | 36022528 | 14060149 |    594990240 |     2635362677 |   141348317 |
| REI infs for REX & REI proofs   |  8670913 |  1681341 |    267973213 |     1092512767 |    42204117 |


2021-04-08
----------

To speed up the reevaluation of solved problems,
I added a mode to the evaluation which only evaluates problems solved.
I use it like this:

~~~
SETS={bushy,chainy,TPTP-v6.3.0,miz40-deps.a15,flyspeck-top}
CFGS=meancop--conj{,--cuts{ei,ex},--cutsr{,ei,ex}}
eval make USE_SOLVED=1 o/$SETS/{1,10}s/$CFGS -j40
eval make -f solved.mk solved/$SETS/{1,10}s/$CFGS
~~~


2021-04-07
----------

Long time, no write. How have you been?

I wrote a rebuttal for my CADE article and started to realise my proposed changes.
The most complex change involves collecting proof step statistics in meanCoP.
For this, I now collect statistics for every proof step about
the backtracking that was involved to find it.
I took this as an opportunity to also allow for
easier analysis of the number of inferences.

To demonstrate the new statistics, I analyse the bushy problems
solved by the complete meanCoP strategy in 10 seconds.

Total number of solved problems (same as `cat *.p | wc -l`):

    cat *.p | jaq -s 'length'
    546

Total number of inferences for all solved problems:

    cat *.p | jaq -s '[.[].infs | add] | add'
    122387404

Total number of inferences spent in the highest depth:

    cat *.p | jaq -s '[.[].infs[-1]] | add'
    99389423

Average depth at which a proof is found:

    cat *.p | jaq -s '[.[].infs | length] | add / length'
    3.835

Total number of proof steps:

    cat *.p | jaq -s '[.[].branches.closed] | add'
    9697

Total number of proof steps that require
a change of the root / a change below the root
(**NOTE: all numbers from here on have to be considered wrong**):

    cat *.p | jaq -s '[.[].branches.root_changed] | add'
    745
    cat *.p | jaq -s '[.[].branches.descendant_changed] | add'
    228

This is quite cool; it shows that about 3/4 of all backtracking
involves only root changes, but no backtracking below.
Read the other way around, one can catch 3/4 of the beneficial backtracking
by just allowing a change of root steps.

Total number of proof steps that could be found by REX / REI:

    jaq -s '[.[].branches | .closed - .descendant_changed] | add'
    jaq -s '[.[].branches | .closed - .root_changed] | add'

Total number of proofs requiring no root / descendant changes:

    cat *.p | jaq -s '[.[].branches | select(.root_changed == 0)] | length'
    254
    cat *.p | jaq -s '[.[].branches | select(.descendant_changed == 0)] | length'
    399

The first and second number shows how many of the found proofs we would have found
if we had forbade all non-essential backtracking / only descendant backtracking.
Forbidding all backtracking loses about half of the proofs, whereas
only forbidding descendant backtracking loses only one fourth of the proofs.
That means that restricted backtracking (EI) loses much more proofs than EX.

Total number of proofs requiring neither root nor descendant changes:

    cat *.p | jaq -s '[.[].branches | select(.root_changed == 0 and .descendant_changed == 0)] | length'
    254

So whenever there is a descendant changed, there is also some root changed.
That is OK, because the changed descendant is also some root.


2021-03-06
----------

As a long-outstanding TODO, I have evaluated the usage of `once_cell`'s
[`Lazy`](https://docs.rs/once_cell/1.7.2/once_cell/unsync/struct.Lazy.html) type
to replace my [`lazy_st`](https://docs.rs/lazy-st/0.2.2/lazy_st/) package in
[Kontroli](https://github.com/01mf02/kontroli-rs).
I did this because I discovered that the `Lazy` type will be
integrated into the Rust standard library.

First, I noted that I could not use `Lazy` as
a drop-in replacement of my current solution,
due to restrictions of the `FnOnce` trait.
I documented this in an [issue](https://github.com/matklad/once_cell/issues/143),
as well as a solution to my problem.
The solution follows `once_cell`'s `Lazy` type closely;
only the order of type arguments is swapped and
the `Into` instead of the `FnOnce` trait is used.
This allows this new `Lazy` type to be used as
a drop-in replacement of `lazy_st`'s `Thunk` type.
It looks as follows:

~~~ rust
use core::{cell::Cell, ops::Deref};
use once_cell::unsync::OnceCell;

pub struct Lazy<T, U> {
    from: Cell<Option<T>>,
    into: OnceCell<U>,
}

impl<T, U> Lazy<T, U> {
    /// Create a new lazy value with the given initial value.
    pub const fn new(from: T) -> Self {
        Lazy {
            from: Cell::new(Some(from)),
            into: OnceCell::new(),
        }
    }
}

impl<T: Into<U>, U> Lazy<T, U> {
    /// Force the evaluation of the lazy value and
    /// return a reference to the result.
    ///
    /// This is equivalent to the `Deref` impl, but is explicit.
    pub fn force(&self) -> &U {
        self.into.get_or_init(|| match self.from.take() {
            Some(from) => from.into(),
            None => panic!("Lazy instance has previously been poisoned"),
        })
    }
}

impl<T: Into<U>, U> Deref for Lazy<T, U> {
    type Target = U;
    fn deref(&self) -> &U {
        self.force()
    }
}
~~~

I evaluated Kontroli using:

    cat examples/bool.dk examples/sudoku/sudoku.dk examples/sudoku/solve_easy.dk | kocheck -

Before (using `lazy_st`'s `Thunk`):

    Time (mean ± σ):     690.2 ms ±   2.3 ms    [User: 659.7 ms, System: 31.4 ms]
    Range (min … max):   686.5 ms … 693.7 ms    10 runs

After (using my custom `Lazy` type based on `once_cell`):

    Time (mean ± σ):     717.2 ms ±   4.2 ms    [User: 683.7 ms, System: 34.4 ms]
    Range (min … max):   710.3 ms … 723.0 ms    10 runs

So using the `Lazy` type based on `once_cell` decreases performance.
This could be because its implementation contains two cells, whereas
`lazy_st`'s `Thunk` contains just a single cell.
Conclusion: I will stick with `lazy_st` for the time being.


2021-02-26
----------

I finished the CADE-28 article. Finally! It has been quite a stretch at the end ...
The result can be admired [here](http://cl-informatik.uibk.ac.at/users/mfaerber/cade-28.html).
Given that my sleep has suffered quite a bit due to the paper writing,
I will take the next week off. I truly have some garbage collection to do.

On Monday, I learned during the group lunch that
SWI-Prolog was made by
[Jan Wielemaker](https://en.wikipedia.org/wiki/Jan_Wielemaker) at the Vrije Universiteit, and that
[Andrew S. Tanenbaum](https://en.wikipedia.org/wiki/Andrew_S._Tanenbaum)
is strongly linked to the VU as well.
This led me also to discover one of the founders of logic programming,
Maarten van Emden, who has some nice essays on
[his web site](http://webhome.cs.uvic.ca/~vanemden/).

What follows are various commands that I used to analyse data for the CADE-28 evaluation.

To calculate the union of solved problems for different strategies:

All previously existing strategies (not containing shallow cut):

    for i in `eval echo $SETS`; do echo -n " |" `ls solved/$i/10s/* | grep -v shallow | xargs cat | sort -n | uniq | wc -l`; done

Two best strategies (RES and RED):

    for i in `eval echo $SETS`; do echo -n " |" `cat solved/$i/10s/leancop--conj--cutred--cutext* | sort -n | uniq | wc -l`; done

All strategies:

    for i in `eval echo $SETS`; do echo -n " |" `cat solved/$i/10s/* | sort -n | uniq | wc -l`; done

To calculate the times of solved problems:

For meanCoP:

    for i in `grep -l "% SZS status Theorem" o/bushy/10s/leancop--conj--cutred--cutextdeep/*.p`; do jaq '.user' < $i.time; done

For leanCoP:

    for i in `grep -l "p is a Theorem" out/bushy/jar/plleancop-171116-nodef-cut-conj/*`; do grep "elapsed" $i.time | cut -d "u" -f 1; done

For Vampire:

    for i in `grep -l "SZS status Theorem" out/bushy/jar/vampire-4.0-casc/*`; do grep "elapsed" $i.time | cut -d "u" -f 1; done

Every command is then piped to `sort -n | awk '{print $s " " NR}'`.
This data can be plotted using the approach from
<http://gedenkt.at/blog/plotting-integrated-data/>.

The raw output of the evaluation was saved as follows:

    tar cjf out.tar.bz2 --exclude=*.o --exclude-from <(find o/ -type f -size +1k) o/

I noticed that some inference statistics files were several megabytes large,
especially for the incomplete prover strategies.
That is why I exclude any of these that are over 1kB, because I expect those to
rarely correspond to a proof.
In the long run, it would be good to
detect such cases automatically and report Non-Theorem.


2021-02-05
----------

Last week, I have refined the backtracking data types.
In particular, [the week before](#2021-01-15),
I silently used singly-linked lists, hoping that
the increased speed for cloning lists might make up for the slow traversal.
It turns out I was wrong; using `Vec`s instead is clearly faster,
despite their cloning overhead.

This week, I found out that it makes sense to distinguish cut on
reduction and extension steps:
There is only one kind of cut on reduction steps, namely
the cut that excludes all other alternatives on solving a literal.
There are two kinds of cut on extension steps, namely
the cut that excludes all other alternatives on solving a literal (deep) and
the cut that excludes only those alternatives that start with the same extension step (shallow).
So far, what I called "shallow cut" always implied the cut on reduction steps.
I now made this optional, making the description of shallow cut simpler.
As it turns out, switching off the cut on reduction steps gives usually worse results.

I have also started running evaluations on several datasets.
To run these, I used the following command in the `cop-rs/eval` directory:

    SETS={bushy,chainy,TPTP-v6.3.0,miz40-deps.a15,flyspeck-top}
    CFGS=leancop--conj{,--cutred}{,--cutextshallow,--cutextdeep}
    eval make o/$SETS/1s/$CFGS -j40

The preliminary outcome on a 1 second timeout is shown in the following table.
In the cut column,
"R" stands for cut on reduction steps,
"ED" stands for deep cut on extension steps, and
"ES" stands for shallow cut on extension steps.
The best configuration is shown in bold.

| Cut           |   bushy |  chainy |     TPTP |     Miz40 |   FS-top |
| ------------- | ------: | ------: | -------: | --------: | -------: |
| None          |     479 |     172 |     1501 |      8046 |     3729 |
| R             |     581 |     206 |     1629 |     11632 |     3954 |
| ED            |     642 |     232 |     1712 |     12109 |     3830 |
| ES            |     695 |     210 |     1741 |     13121 |     4174 |
| RED           |     655 | **244** |     1718 |     12120 |     3870 |
| RES           | **723** |     238 | **1810** | **13894** | **4432** |

The table was created with the help of the following commands:

    eval make -f solved.mk solved/$SETS/1s/$CFGS
    for s in `eval echo $CFGS`; do echo -n $s; for i in `eval echo $SETS`; do echo -n " |" `cat solved/$i/1s/$s | wc -l`; done; echo; done
    sed -e 's/leancop--conj//' -e 's/--cutred/R/' -e 's/--cutext/E/' -e 's/shallow/S/' -e 's/deep/D/'
    pandoc -t gfm

I also had a quick look at which combination of strategies was most efficient,
that is, which are the three strategies of which the union of solved problems is maximal?
For the bushy dataset:

    for s1 in `eval echo $CFGS`; do
    for s2 in `eval echo $CFGS`; do
    for s3 in `eval echo $CFGS`; do
    echo `cat solved/bushy/1s/{$s1,$s2,$s3} | sort | uniq | wc -l` $s1 $s2 $s3;
    done;
    done;
    done | sed -e 's/leancop--conj//g' -e 's/--cutred/R/g' -e 's/--cutext/E/g' -e 's/shallow/S/g' -e 's/deep/D/g' | sort -n

It looks like the most efficient combination of strategies
is most frequently RES + RED + None or RES + RED + R.

As a small side project, I also made the connection prover
[`no_std`](https://rust-embedded.github.io/book/intro/no-std.html) compliant.
That means that it should be possible to compile it to platforms such as WASM,
enabling it for the use in web sites!
(Or I could do it like Jens and compile it to an iPod. ^^)
I have now assigned two students to make a web site allowing the usage of the prover.


2021-01-15
----------

This week, I have corrected the complete strategy of my prover.
To diagnose the problem, I looked for problems for which
(a) proofs were found by both my new prover and the original leanCoP,
(b) the number of inferences diverges, and
(c) the number of inferences is as small as possible (to ease the manual inspection).
After a few hours, I found the culprit:
Incomplete strategies always backtrack to
path and lemmas that are prefixes of the current path and lemmas, whereas
complete strategies may backtrack to
path and lemmas that are not prefixes of the current path and lemmas.
(I call the pair of path and lemmas *context*.)
However, so far in the prover, there was
only one global `Vec` for each path and lemmas,
which cannot capture the backtracking behaviour of the complete strategy.

The naive way to handle this problem is to clone the context
whenever one creates an alternative (to backtrack to).
However, in my experiments, this slowed the prover down
by about 50% in the incomplete strategies.
I determined this number by finding the problem
that is taking the longest time to solve via:

    for i in *.o; do FILE=`basename $i .o`.time; echo `jaq '.elapsed' < $FILE` $FILE ; done | sort -n

Then, I ran the prover with cloning contexts on this problem again,
measuring the difference.

To solve the problem more efficiently, I thought about using
the backtracking stack (see [2020-11-20](#2020-11-20)),
but I could not figure out a way to make this work here ---
it is probably not the right data structure for the context.
In the end, I went for a quick-and-dirty solution:
Whenever we use a complete strategy, we now
save a clone of the current context on creating an alternative, and
whenever we use an incomplete strategy, we only
save a pointer (containing path and lemmas length) to the context.
With this method, we reach good performance
for both complete and incomplete strategies:
On the bushy dataset with a timeout of 1 second, we solve
504 problems in the complete (`--conj`) configuration and
748 problems in an incomplete (`--conj --cut shallow`) configuration
(measured on my Intel NUC).
In the JAR paper, fleanCoP (leanCoP implemented in OCaml with streams)
with the same settings (`--conj`) and 10 seconds timeout
proves only 499 problems!
That means that we now solve more problems in just 1/10 of the time.

By the way, when evaluations are running and I want to get a prognosis of
the percentage of solved problems, I use the following command:

    echo scale=3\;`ls *.o | wc -l`/`ls *.p | wc -l` | bc

Together with `watch`, this can provide an interactive, progressive evaluation.

Apart from this, I went to university on Tuesday
(for the first time in the last 12 months!), where
I retrieved my access credentials.
I checked out the new building (into which I had never set foot before),
and it was quite nice.
A friendly soul even helped me for half an hour to set up printer access.

On Thursday, I came back to university to have some video conferences with
students who were interested in me supervising their bachelor's thesis.
I am now awaiting their reactions next week
to see whether they are still interested. ;)


2021-01-08
----------

Welcome to 2021!
I have gotten back to work quite refreshed from my Christmas holidays,
during which I have gotten used to sleeping much longer (about 10h).
I found this to vastly improve my productivity,
and even rendering my need for siestas obsolete. ;)

This week, I believe to have understood the behaviour
I previously described on [2020-11-27](#2020-11-27).
I think that the behaviour is indeed sound,
and my evaluations have indicated that
it greatly improves the prover power.

So what is this all about?
In leanCoP, we have two backtracking strategies:
cut, or no cut.
Suppose we have to refute a literal `p`.
With cut, if we are able to refute `p` starting with some extension step,
then this refutation of `p` is fixed, meaning that
even if at some later point we are not able to complete the proof,
we are never going to try to find an alternative refutation of `p`.
Without cut, leanCoP will also try to find alternative refutations of `p`,
backtracking to *anywhere inside* the refutation of `p`.
The strategy I discovered will also consider alternative refutations of `p`,
but it is also incomplete because it will only allow backtracking to
*the very first* proof step in the refutation of `p`, i.e.
it will choose a different extension step for `p`.
(Do not worry if you do not understand this
by my relatively hand-wavy description,
I have some graphics that should illustrate this way better.)
In that sense, the new strategy retains soundness,
because it still covers a (strict) subset of
the search space covered by the complete strategy.
However, it is "less incomplete" than the basic cut strategy,
because it will allow for more backtracking to happen.
Accordingly, I call "deep cut" what leanCoP usually calls "cut",
whereas "shallow cut" is my less invasive cut. (Think surgery.)
It is between cut and no cut.

To verify my understanding of the new strategy,
I created a test problem (`shallowcut.p`),
which deep cut can never solve,
but both shallow cut and the complete strategy can,
with the shallow cut strategy taking one inference step less.
This example has made me confident about my understanding of the issue.

To evaluate my prover with larger timeout, I really needed access to some cluster.
I found that the old server I used in Innsbruck (colo12)
is able to run a Rust toolchain,
which is not something to take for granted given that
I had already spent hours to install OCaml on it
(due to the server's ancient glibc).
Luckily, it took me less than five minutes to
install Rust and compile my prover on the server.
I evaluated the bushy and chainy datasets, each with a timeout of 10 seconds.
The results are as follows:

Configuration       | Bushy | Chainy
------------------- | ----: | -----:
cop-conj-deepcut    |   727 |    331
cop-conj-shallowcut |   845 |    293
Union               |   901 |    376

For the bushy dataset, my shallow cut strategy yields outstanding results:
Just the shallow cut strategy solves **16.2%** more problems than
the deep cut strategy (known to me as the previously best single strategy),
and together they solve **23.9%** more problems than deep cut,
indicating that the two strategies are relatively complementary.
For the chainy datasets, shallow cut proves fewer problems than deep cut,
but at least the union improves by 13.6%, still indicating nice complementarity.

(There seems to be a problem with the configuration "cop-conj",
which currently only proves 211 problems, which is way below
the 499 solved problems reported in my
[JAR paper](https://dx.doi.org/10.1007/s10817-020-09576-7)
for the corresponding fleanCoP configuration.
I have to investigate this.)

In the light of these very promising results,
I am considering to submit an article to
[CADE-28](https://www.cs.cmu.edu/~mheule/CADE28/),
giving me the following deadlines:

* Abstract   deadline: February 15, 2021
* Submission deadline: February 22, 2021

That gives me about six weeks to write.
In the article, I would like to cover the following topics:

* the promises/alternatives architecture
  (while it is mostly what Cezary has described,
  I believe to have a nice formulation of it),
* the backtracking stack to enable unrestricted backtracking
  (see [2020-11-20](#2020-11-20)),
* the new, less restricted backtracking strategy.

I have already started work on the article.
The current biggest obstacle for my work is ergonomics;
the table I am writing on right now is clearly not fit for work,
and I had to frequently stop working because of hand/arm pain.
Now that I use some cushions, it is a bit better,
but I am religiously awaiting my new furniture and chair I ordered.
Unfortunately, I also could not go to university yet due to quarantine ...

Apart from this, I would like to know how easy it would be to
integrate the new search strategy into, say, Prolog leanCoP,
or one of the various OCaml implementations.
My hypothesis is that the promise/alternative architecture is crucial for this,
but I would like to stand disproved.


2020-12-23
----------

I have greatly cleaned up the clausal proof search.
Furthermore, I have implemented a small proof checker
that is being run after proof search.
This should help me validate the correct function of the prover and
find out whether the behaviour I described on [2020-11-27](#2020-11-27)
is sound or not.
Unfortunately, before leaving Amsterdam for Tyrol, I forgot to
synchronise my evaluation infrastructure between my main computer and my laptop,
so I am currently without evaluation infrastructure.
I will wait for the evaluation until I have access to my main computer again.


2020-12-15
----------

Last week, I brought my jq clone, jaq, into a usable state.
Its functions are documented at: <https://github.com/01mf02/jaq>


2020-12-04
----------

This week, I took a break from my usual research in order to
create an alternative to a tool I was counting on to use for my research: `jq`.
(I am going to talk about `jq` 1.6 for the rest of this post.)
As mentioned last week, the startup time of `jq`
is simply too high for me to make it usable.
For example, to read a number and output it again, `jq` takes about 50ms!

~~~
$ hyperfine "echo '0' | jq '.'"
Benchmark #1: echo '0' | jq '.'
  Time (mean ± s):      52.5 ms ±   1.2 ms    [User: 51.7 ms, System: 1.0 ms]
  Range (min … max):    50.9 ms …  55.6 ms    53 runs
~~~

[The problem is known since at least 2017](https://github.com/stedolan/jq/issues/1411),
yet I found no sign of progress on solving it since.
As one of the basic premises of my research project was
to use more disciplined, less ad-hoc formats to capture data,
the JSON format imposed itself as an easily readable and writable,
de facto standard with lots of tool support.
Having a painless and fast way to interpret JSON data is thus indispensable for me.
However, the aforementioned speed bottlenecks of `jq` are detrimental to this goal.

I thus attempted to recreate a tool that permits using the same syntax as `jq`,
but should start up faster in order to allow for the processing of many small files.
My goal was to have a working tool at the end of this week
that should support a large set of operations that `jq` supports.
I baptised this tool `jaq` (to be pronounced like "Jacques") and
made it available at <https://github.com/01mf02/jaq>.

To motivate this a bit more, I give an example of a task that I wanted to perform.
When running my automated theorem prover, I output at certain points
how many inferences it has performed.
The output looks like this:

~~~
{ "pathlim" : 1 , "inferences" : 11 }
{ "pathlim" : 2 , "inferences" : 577 }
{ "pathlim" : 3 , "inferences" : 7393 }
{ "pathlim" : 4 , "inferences" : 110791 }
~~~

At the end, I want to sum the number of inferences to retrieve
the total number of inferences, i.e. 11 + 577 + 7393 + 110791.
With `jq`, I would obtain this as follows:

    jq -s '[.[].inferences] | add'

This command
reads the whole file into a single array (`-s` for "slurp"), then
obtains the array elements with `.[]`,
obtains the inferences values with `.inferences`,
puts the results back into an array `[.[].inferences]`, in order to finally
feed the resulting array with `|` to a function that adds the values.
This outputs the desired value "118772" when
being given the data above via standard input.

However, this takes `jq`, due the slow startup as mentioned above, about 50ms.
For a dataset of 2073 files of this shape, `jq` takes a whopping 114 seconds.
In contrast, my new tool, `jaq`, takes only about 8 seconds,
meaning that it is about 14 times faster on this particular input.
Of course, this does not mean that `jaq` is always faster than `jq`.
For example, take the [open Paris street name data](https://opendata.paris.fr/explore/dataset/noms_voies_actuelles_paris).
Let us try to get the names of all streets in Paris:

    jq '.[].fields["nom_de_voie"]' < noms_voies_actuelles_paris.json

Here, `jq` takes only about 280ms, whereas `jaq` currently takes about 350ms.
However, taking into account that this is the state after five days of development
with little focus on optimisation, whereas there are years of development behind `jq`,
I personally find that already impressive enough.

This amazing ratio between performance and development time
would not have been possible without the programming language Rust,
in which I wrote `jaq`.
In particular, the parser generator [`pest`](https://pest.rs/)
(with which I had no experience before the start of this week)
was crucial to rapidly create a parser for a large subset of the `jq` language.
Furthermore, the packages `serde_json` and `colored_json` to read and output JSON
allowed me to concentrate on the program logic as much as possible.

My plan is to implement all functions of `jq` in `jaq` which
`jaq` can currently parse, but not execute, such as
if-then-else, arithmetic on non-numeric values, and iteration over object values.
Furthermore, I would like to implement a few more useful functions,
such as `map`, `select`, and most importantly `recurse`,
which corresponds to a fixed-point combinator and should make `jaq` Turing-complete.
I estimate this to be doable in one week.
This should prepare the ground for other people to productively use `jaq`
as well as to extend it in order to more closely match `jq`'s functionality.

And then, it's back to theorem proving, baby! :)


2020-11-27
----------

This week I spent most of my time on eliminating differences between
my connection prover in Rust and its pendants written in Prolog/OCaml.
My main means of doing so was to look at the number of inferences
for the problems that both my prover and one of the other provers solved,
and to inspect the problem with the lowest differing number of inferences.
At the beginning of the week, nearly all problems were
solved with a different number of inferences.
At the end of the week, all problems that were
solved by one configuration of the OCaml prover were also solved by the Rust prover.
This involved fixing a number of bugs, including
the reordering of formulas according to the number of paths,
the order of contrapositive clauses,
the unfolding of logical equivalence like in leanCoP,
the order of equality formulas like in leanCoP, and
the recording of reduction steps as lemmas.

I also noticed that I had originally swapped
the order of promise and alternative storing on extension steps.
After [correcting this](https://github.com/01mf02/cop-rs/commit/adfba8ce72365702f1eebd73207ff5454e829e17),
the number of bushy problems solved reduced from 741 to about 663!
I am not completely sure yet what it means to swap the promise and alternative storing,
but I would like to find out
whether this is sound and
whether this corresponds to some setting already existent in a connection prover.
My guess is that the swapped order disables cut on extension steps
(`nocut3` in the OCaml prover),
but this is countered by the fact that
disabling cut on any proof step decreases the number of solved bushy problems.
There is still a slight chance that I *did* hit a really cool optimisation
(given that Jens also seems to have discovered the `cut` technique by accident),
so if I get run over by a car, please test my research hypothesis. :P

To analyse the number of inferences per problem,
my original plan was to use the really nice [`jq`](https://stedolan.github.io/jq/) tool.
Unfortunately, I noticed that the startup time of `jq` 1.6 amounts to about 50ms
regardless of the output (measured by `time echo "" | jq "."`),
so running `jq` for every inference statistics file produced by the prover
is quite slow and makes `jq` basically useless for me.
I thus resorted to good ol' `grep` & friends,
which worked fine after slightly adapting the output format (JSON).
Here are the commands to print the inferences for each solved problem:

For the OCaml prover:

    for i in `grep -l Theorem *.p`; do echo `grep Inf $i | cut -d ' ' -f 7` $i; done

For the Rust prover:

    for i in *.p; do if [ -e $i.o ]; then echo `tail -1 $i.infs | cut -d ' ' -f 8` $i; fi; done

It really itches me to make a small `jq` replacement that is actually fast ...

Anyway, a quite nice result from this week is that with 1 second timeout,
the Rust leanCoP proves 6.4% more problems (663) than
stackCoP, i.e. the fastest leanCoP version implemented in OCaml (623).
The inference rate seems to be about three times higher!
This makes it interesting to see how the Rust prover will fare against the C version ...


2020-11-20
----------

I realised that my understanding of backtracking I had until last week was flawed.
As I result, I realised that I needed to introduce
a new data structure to capture backtracking:
A stack whose state can be snapshot and restored.
The characteristic of this data structure is that nearly all operations are O(1),
including the creation and restoring of snapshots.
With this data structure, one can search for proofs very efficiently
with unrestricted as well as restricted backtracking.

The first benchmarks are promising;
I got from 4 solved bushy MPTP2078 problems to 585 problems (1s timeout, cut + conj - def).
("cut" stands for restricted backtracking,
"conj" for conjecture-directed search, and
"def" stands for definitional clausification.
"+" stands for "I use this", and "-" stands for "I do not use this".)
The previously best configuration in OCaml, using stacks for backtracking,
holds at 558 solved problems, using the same parameters.

Next week, I want to compare my prover with the C implementation written by Cezary,
and remove functional differences between the prover and its Prolog/OCaml/C pendants,
in order to assure that performance comparisons are fair.


2020-11-13
----------

As I wrote last week, I worked this week on ironing out bugs and setting up benchmarks.
As a result, I found and corrected several unsoundness bugs, and
my prover now supports proof output, conjecture-directed search, and restricted backtracking.
Furthermore, the prover now acts as a state machine, which has the nice side effect that
proofs can be recorded just by recording the relevant state transitions.
I also found today that the implementation technique that
Cezary describes in his TABLEAUX'15 paper seems to be limited to restricted backtracking,
because without unrestricted backtracking,
some assumptions done in the implementations no longer hold and the prover becomes unsound.
I have slightly generalised Cezary's technique in order to perform unrestricted backtracking.
However, I still need to debug the new backtracking mechanism,
as the prover currently claims to only solve 4 out of 2078 bushy MPTP2078 problems
with restricted backtracking enabled, whereas I would have expected more like 400. :/


2020-11-06
----------

This week, I worked on getting my new connection prover to a working and performant state.
This includes
iterative deepening,
equality handling,
clause variable minimisation,
mapping constants to string pointers for fast symbol comparison, and the likes.
The next week's work will be
to iron out bugs and to test the prover on actual TPTP problems,
measuring how it performs there, and removing potential bottlenecks.
You can follow development here: <https://github.com/01mf02/cop-rs>


2020-10-06
----------

This week, I learnt how to work again.

Working from home requires a certain discipline that I had forgotten about,
and I needed some time to get used to it again.
In particular, when not paying attention,
I spent very long periods in front of the computer, making few pauses.
As a result, I felt quite beaten down at the end of the day,
and in the end, I also slept badly and felt tired the next day.

To avoid this, I readopted the rhythm that I had imposed on myself already in Paris:

1. Breakfast
2. Work
3. Sport
4. Work
5. Lunch (+ sometimes siesta)
6. Work
7. Sport (+ tea)
8. Work

The work blocks are roughly 90 minutes long, and the sport blocks about 30 minutes.
I did this today, with
running to the supermarket in the morning sports block and
cycling in the forest in the afternoon sports block.
Result: I feel much better now!

Before having adopted this work rhythm, I felt increasingly depressed.
To counter this, I listened to some Austrian radio ---
thank you, [Ö1](https://oe1.orf.at/), for cheering me up.

I also watched a film, namely
[Castle in the Sky](https://en.wikipedia.org/wiki/Castle_in_the_Sky).
This film deeply moved me, inspiring in me feelings of nostalgia.
The world in the film struck me as so beautiful that I wanted to be part of it.
Many elements in Castle in the Sky reminded me of Japanese RPGs such as
[Final Fantasy](https://en.wikipedia.org/wiki/Final_Fantasy) or
[Chrono Trigger](https://en.wikipedia.org/wiki/Chrono_Trigger),
making me wonder to what extent that film had an influence on the RPG genre?
I actually watched the film two times on two consecutive evenings,
and the two days after, I listened to the whole film soundtrack.
The music makes this film even more special than it already would be without.
(Another property that the film shares with aforementioned RPGs.)

Apart from Japanese, I also intensified my knowledge of Spanish culture,
listening to [Mecano](https://en.wikipedia.org/wiki/Mecano),
in particular their album
[Descanso dominical](https://en.wikipedia.org/wiki/Descanso_Dominical),
to which I was introduced when visiting my cousin's family in Southern France.
I even translated some songs and tried to sing along the Spanish lyrics.
My current favourites are
"El cine" (the first song I heard from the band),
"No hay marcha en Nueva York" (incredibly funny lyrics),
"La fuerza del destino", and
"Laika" (very sad song once you understand the lyrics).
You can listen to the album [on YouTube](https://www.youtube.com/playlist?list=PLKpzEx0BAPRjAIcDi0PxVtAIYUA9n_4Tt).

My work pace has been negatively impacted by one thing:
As I previously mentioned, my apartment is below a major flying lane,
and from time to time, airplanes are literally flying over my head.
When they do, it is not only one plane, but up to thirty (!); one per minute.
The airplane traffic can occur anytime between 7 am and 10 pm
depending on the wind conditions, and
the noise is so loud that I can hear the planes despite closed windows and earplugs.
On the weekend, I was so desperate about the situation that
I went to various shops and tried several headphones with active noise cancelling.
Luckily, I found out since that new earplugs seem to
cancel the airplane sound much better than my old ones.
This, optionally combined with a white noise generator,
seems to cancel out the airplane sounds completely.
(Luckily, since the weekend,
there has been hardly any airplane traffic over my apartment.)
Another potential reason for me hearing the airplanes so well could also be that
one window in my flat seems to be not completely closing,
putting me into the same situation as last year in Paris
(where I enjoyed windows that would not even be legal to install anymore).
It seems to be part of my personal Murphy's Law that I'm always exposed to
some annoying noise source, be it airplanes, cars, neighbours etc.

Despite all this, my work is advancing steadily:
I have decided to work for now on
a reimplementation of leanCoP and nanoCoP in Rust,
in order to get used again to automated theorem proving.
So far, I have the preprocessing mostly done until (excluding) matrix generation.
Nothing groundbreaking so far, just regular work on foundations.


2020-10-01
----------

A new life starts: Work in Amsterdam has started!
And I'm (de facto) in quarantine. Yay!
In order not to go crazy, I start my notes now.

For people reading this who are not me (hello, by the way):
My project is about improving the performance of nonclausal automated theorem provers.
(If that means nothing to you, you may still try to continue reading my notes.
I will probably not be too technical for now.)
In a nutshell, my project will cover a more theoretical and a more practical portion:
One will be to better *understand* nonclausal reasoning, and
the other will be to *apply* what we learnt in order to improve nonclausal reasoning.

I came back from Paris yesterday by car with some remaining stuff to move to Amsterdam.
As Paris is a red Covid-zone for the Dutch,
I will try to keep my social interactions to a minimum for the next days.
(Thanks to the Dutch for relying on common sense and cooperation
rather than strict measures.)

Today, I'm all alone, which means that this is perfect to get some work done.
As my host at Vrije Universiteit Amsterdam,
[Jasmin Blanchette](http://www21.in.tum.de/~blanchet/),
is not here yet,
I have enjoyed my first work day at my cozy little apartment in Amstelveen
(except for when planes are flying over it).

So what did I do today?
I started by downloading and launching Isabelle2020 together with the newest AFP,
where <https://www.isa-afp.org/using.html> explains installation of the latter.
I then browsed some theories, in particular
[`Ordered_Resolution_Prover`](https://www.isa-afp.org/entries/Ordered_Resolution_Prover.html).
To play with it:

~~~ isabelle
theory Scratch
  imports
    Ordered_Resolution_Prover.FO_Ordered_Resolution
    Ordered_Resolution_Prover.FO_Ordered_Resolution_Prover
    Ordered_Resolution_Prover.Herbrand_Interpretation
begin

end
~~~

This is part of [IsaFoL](https://bitbucket.org/isafol/isafol/wiki/Home), which is
a loose collection of projects to formalise logic in Isabelle, similarly to
[IsaFoR](http://cl-informatik.uibk.ac.at/software/ceta/).
One nuisance I encountered was to find out:
given an article, where to find its corresponding formalisation?

I also looked at
[`SATSolverVerification`](https://www.isa-afp.org/entries/SATSolverVerification.html),
which like `Ordered_Resolution_Prover`
also contains a formalisation of clauses, literals and so on.
Its author, Filip Marić, is not listed on the IsaFoL site, so
I suppose that this is an independent development.

The `Ordered_Resolution_Prover` seems to be quite nice and modular,
having also a rather clean-looking formalisation of substitution and Herbrand models.
Quite likely some parts will be useful for my project as well ...
The project seems to be documented in the article
"[Formalizing Bachmair and Ganzinger's Ordered Resolution Prover](http://matryoshka.gforge.inria.fr/pubs/rp_article.pdf)".

And now for something completely different.

I found a TPTP parser library for Rust, written by a certain
Michael Rawson from Manchester from who I had already heard a talk at AITP.
It turns out that he has already written a connection prover in Rust
(thus anticipating a pet project of mine),
and furthermore, he called it [lazyCoP](https://github.com/MichaelRawson/lazycop)
like I called one of my provers.
I tried it, but could not get it to terminate on the supplied problems,
whereas with some test problems from my provers, it worked just fine.
While Rawson's lazyCoP shares the name with my lazyCoP,
it is not about lazy list processing, but about lazy paramodulation,
which I would expect to be a solution to
leanCoP's suboptimal performance on problems involving equality.
Unfortunately, in CASC, Rawson's lazyCoP seems to not have performed too well yet.
I hope that it will become better ...

So far, my computer memory of 8GB seems to be sufficient,
but running Isabelle at the same time as some automated theorem prover
becomes a daunting task.
Luckily, the old me from 2015 was foresighted enough to equip my computer with a 8GB RAM stick
(the maximum size for a single stick supported by my computer),
leaving space for 8GB more.
So if 16GB turn out to be sufficient, it makes sense to upgrade my current computer
(instead of buying a completely new one).

That's it for today.
