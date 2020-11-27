# Amsterdam notes

This is the research notebook for my FWF project in Amsterdam.


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
