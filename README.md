# Amsterdam notes

This is the research notebook for my project in Amsterdam.


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
