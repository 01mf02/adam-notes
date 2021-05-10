# Amsterdam notes

This is the research notebook for my FWF project in Amsterdam.


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
    for i in `cat cop/eval/solved/$s/jar/plleancop-171116-nodef-cut-conj | sed 's|\(...\)|Problems/\1/\1|'`; do
    cat cop-rs/eval/o/TPTP-v6.3.0/10s/meancop--conj--cutsrei/$i.time;
    done | jaq -s '[.[].user] | add'
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
