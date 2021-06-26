## Motivation

Predicting transfer successes would make you rich.

## Introduction

We’ll try and create a model the predicts the change in a player’s
performance after a transfer based on the previous league and the next
league. In this process, we will get a more general mapping between
pairs of leagues allowing us to extract some more insights.

## Inputs

We use
[VAEP](https://github.com/soccer-analytics-research/fot-valuing-actions),
a way to measure the value of individual actions in football. The link
has more details but in short, it calculates a probability of a team
eventually scoring a goal as a result of a player performing a certain
action.

\[@canzhiye\](<https://twitter.com/canzhiye>) has fortunately computed
this for all the players playing in a bunch of leagues for the last some
years.
\[@TonyElHabr\](<https://twitter.com/TonyElHabr/status/1405946557237796871>)
recently used it in a model to solve the same problem as me which is
where this discussion started and he was kind enough to help me get
access to the data to try my approach on it too. We have different
models though so you should go and read his post as well.

## Model Structure

This model will centre on a player’s VAEP/minute value as a measure
their performance.

Imagine we had just four leagues that we were trying to model. You could
go from each league to any other league and that would look a little
like this -

![](README_files/figure-markdown_strict/model_explainer_1-1.png)

Each link is a path between two leagues and a player could travel in
either direction on that link. So a player could go from the EPL to La
Liga or from the La Liga to the EPL, and similarly for any other pair of
leagues.

Imagine a player moved from the EPL to the La Liga. This player’s VAEP
per minute in their last season in the EPL was 0.005 when in the EPL and
in their first season in La Liga it was 0.006. This is how this would
look like in this visualisation.

![](README_files/figure-markdown_strict/model_explainer_1_example-1.png)

Based on this one observation, we’d say any player going from the EPL to
La Liga can expect their VAEP/minute to go to 0.006/0.005 = 120% of what
it was in the EPL. Let us ascribe a value of 120% to the EPL - La Liga
link. I’m going to call this value the observed VAEP conversion factor.

Let’s do away with links that allow travelling in either direction and
keep only links that let you travel in one direction. This means players
moving from La Liga to the EPL which would need a different link to the
link above. This is what our graph now looks like, with two links
connecting each pair of leagues, one of the links from league a to
league b, and the other from league b to league a. It’s a little painful
to cleanly draw arrows for so many links so going to skip that but
hopefully the description makes it clear.

![](README_files/figure-markdown_strict/model_explainer_2-1.png)

One of the possibilities to model the La Liga to EPL link is to just
assume that it is the inverse of the EPL to La Liga. In the earlier
example, the EPL to La Liga link was 120% so the La Liga to EPL link
would then become 1/120% = 83.33%. I don’t take this approach and I let
them remain independent of each other to allow for the possibility that
this relationship may not exist between the two links.

Another kind of transfer possible is also to a different team within the
same league. Players could gain or lose on the VAEP / minute metric even
for those transfers so we need our model to worry about that too. We
would need another kind of link for this which starts and ends at the
same league.

![](README_files/figure-markdown_strict/model_explainer_3-1.png)

Based on actual transfers we can calculate a median observed VAEP
conversion factor the links over which players moved. A couple of things
to note about this:

-   We can also measure how confident we are about the median based on
    the number of transfers over that link - the more the transfers the
    more confident we can be that the median we have calculated is close
    to the actual VAEP conversion factor of the link. For eg. if we
    looked at only Eden Hazard moving from EPL to La Liga we’d think the
    VAEP conversion factor is ( proabably, I’m guessing ) very low from
    the EPL to La Liga, but if we add Cristiano Ronaldo, Modric,
    Coutinho, and all the other players that made a move then we would
    get a better idea of typically what the observed VAEP conversion
    factor is.

-   We need to incorporate time by giving more importance to recent
    transfers and less importance to transfers that happened very long
    ago. Therefore we’d give higher importance to Hazard’s observed VAEP
    conversion factor, lesser to Ronaldo’s, somewhere in between for
    Coutinho, and so on.

It is possible that some of these links have seen very few transfers or
haven’t seen any transfers recently. We may not have a median observed
conversion factor or have one with very low confidence on it. What we
can do is use links between other leagues and these two leagues to try
and adjust the observed median conversion to get some more confidence.
For eg. let us assume we have never seen a transfer between Ligue 1 to
the EPL but we have seen transfers from Ligue 1 to La Liga and La Liga
to EPL then we can combine the knowledge from the links between these
two leagues and La Liga to try and derive or adjust the value for the
Ligue 1 to EPL link.

Let us make the assumption that moving from Ligue 1 to the EPL directly
would result in a very similar conversion factor as moving from Ligue 1
to La Liga to EPL. In other words, we can estimate the conversion factor
of Ligue 1 to EPL as being equal to the conversion factor of Ligue 1 to
La Liga \* conversion factor of La Liga to EPL. We could also include
more roundabout ways of moving from Ligue 1 to EPL, say. Ligue 1 to
Serie A to La Liga to EPL, and adjust our Ligue 1 to EPL link’s
conversion factor further.

This approach is inherently making an assumption. Imagine we have two
Ligue 1 players with a similar VAEP/minute in Ligue 1 who moved to the
EPL later in their careers. Let’s say player 1 played in the La Liga
between their Ligue 1 and EPL stint while player 2 directly moved from
Ligue to the EPL. The calibration process described above kind of
ignores the La Liga stint and suggests both players would have a similar
VAEP/minute as each other in the EPL as well. This assumption is
probably not true but for now we’ll stick with it. More about this
later.

## Model Evaluation

One way to measure the quality of the calibration is to evaluate the
difference in conversion factors on various paths between the same
leagues, for eg. how different is the value of the Ligue 1 to EPL
conversion factor compared to the Ligue 1 to La Liga conversion factor
\* La Liga to EPL conversion factor. A well calibrated model should have
a very low difference between the two.

Another way to measure the quality of the calibration is to evaluate how
far the calibrated conversion factors are from the observed median
conversion factors. We’d want the two values to be close to each other,
especially for pairs of league where we have seens lots of recent
transfers and are confident about our observed median.

## Model Calibration

The calibration process iteratively makes adjustments to the conversion
factors over links in a manner that the two evaluation criteria improve.
It keeps making adjustments until the value of the objectives don’t
change by much since at that point the adjustments are probably of too
little magnitude to be of consequence.

## Results

I had to exclude China Super League from the list of leagues because it
shows up in fourth position on here behind Serie A and messes a little
with the other leagues as well. I imagine moving to China is very
different from moving to a European or South American club and there
might be other things beyond just the sporting ability playing a part
there.

This is what the model suggests are the conversion factors between pairs
of leagues.

![](README_files/figure-markdown_strict/graph_viz-1.png)

Numbers &gt; 1 indicates moving from the league on the left to the
league on bottom usually see the player increase their VAEP/minute by
that factor. The order of the leagues is based on how many leagues have
a conversion factor of &gt; 1 and is roughly an indication of league
strength although strictly speaking it isn’t that. The order is mostly
sensible with a few debatable ones though like Portugal’s Primeira Liga
being above Germany’s Bundesliga.

It’s interesting that for some of the leagues, a transfer within the
same league typically see a drop in the VAEP/minute metric and the
conversion factor is almost never &gt; 1 across any of the leagues.

If the earlier discussion about having the two links between a pair of
leagues modelled as inverse of each other were true then we’d see
diagonally opposite elements being reciprocals of each other, for eg. a
transfer from Spain La Liga to the Scotland Premiership typically
results in the VAEP/minute becoming 1.75 times or 175% and a transfer in
the other direction see it become 0.57 times which is approximately =
1/1.75 so the inverse relationship holds.

Here is a comparison of every conversion factor with the inverse of it’s
corresponding opposite link -

![](README_files/figure-markdown_strict/corresponding_links_comparison_viz-1.png)

They lie pretty much along the x = y line indicating that even though we
didn’t force the model to keep the inverse relationship, the calibration
reached a solution which upheld that relationship.

This is how much the model suggested conversion factors differ from the
observed factors.

![](README_files/figure-markdown_strict/edges_viz-1.png)

![](README_files/figure-markdown_strict/edges_deviation_viz-1.png)

Pairs of leagues which have seen lots of transfers between them have
very similar values from the model and observation, which is a good
sign. A good side effect of our calibration approach is that the model
has chosen to dampen the cases where it say unusually high or low
conversion factors, which is often between pairs of leagues which have
seen very few transfers and therefore low confidence. I kept the recency
aspect out of this chart just for easier interpretation but we keep
recency also as an input in how we decide confidence in the model.

Let us look at the other criteria of comparing the conversion factor
between two leagues based on the various ways in which players could
move between those two leagues. We will do this by comparing the
conversion factor from league a to league b with the conversion factors
of all paths which involved having up to two other leagues in between
league a and b, eg. league a -&gt; b compared to league a -&gt; c -&gt;
b, league a -&gt; c -&gt; d -&gt; b, etc.

![](README_files/figure-markdown_strict/graph_length_2_viz-1.png)

![](README_files/figure-markdown_strict/graph_length_3_viz-1.png)

The differences are usually very low which means our model was able to
arrive at a solution where all the links were consistent with each other
and in terms of independence of the path followed between two leagues.

## What Next

First - this is a very limited model. It is useful but limited. It is
limited because it models the VAEP conversion factor of players who
*have* moved between leagues which may or may not be a good estimate for
a random player moving between leagues. Players who *have* moved are not
a random sample of players so it is a biased view.

It might also just be useful as a macro model and not useful to predict
the VAEP we can expect from specific transfers. Remember we’re
predicting the median conversion using this model. Here is how the
observed VAEP conversion for each individual transfers compares with the
median suggested by the model.

![](README_files/figure-markdown_strict/calculating_deviations-1.png)

The general trend is reflected in the model’s values but there are large
deviations. You should expect Depay’s output to fall to 90% of his
current output but you can’t be sure.

I’ve been trying to model this difference too but no success yet. In
good news though I also included inputs in this model to check if our
assumption of a -&gt; b being similar to a -&gt; c -&gt; b was wrong but
there is no indication for that either yet. You win some you lose some.
