---
title: "Foul Play: A Competitive Pokémon Showdown Battle Bot"
date: 2025-05-11
slug: "foul-play"
keywords: [""]
draft: false
tags: [""]
---

[Foul Play](https://github.com/pmariglia/foul-play) is a [Pokémon Showdown](https://pokemonshowdown.com/) battle bot capable of playing a variety of singles formats in many
generations of Pokémon. It has achieved high rankings in several formats, including Random Battles, OU, and Battle
Factory. 

Foul Play utilizes a custom battle engine to perform root-parallelized Monte Carlo Tree Search on many battle states.
Foul Play's advanced client tracks the battle state, predicts opponent Pokémon attributes, and can intelligently infer
when information is revealed throughout the battle.

# Search
Foul Play uses a bespoke engine ([poke-engine](https://github.com/pmariglia/poke-engine)) and the
[Monte Carlo Tree Search (MCTS)](https://en.wikipedia.org/wiki/Monte_Carlo_tree_search) algorithm to explore many game
trees, being guided by a custom evaluation function rather than rollouts. MCTS is effective at playing Pokémon as
it naturally prioritizes exploring promising moves rather than searching all possible lines. Previous versions of the
bot used [expectiminimax](https://en.wikipedia.org/wiki/Expectiminimax) to select a move, but without aggressive  move-pruning, this approach would exhaustively
search every branch of the game tree to a certain depth, meaning search beyond about 5 turns ahead was practically
impossible without timing out. When performing MCTS, the most promising lines are usually
searched to a depth of 10 or more, while the uninteresting parts of the tree may only be searched to a depth of
2 or 3.

Expectiminimax also suffered from the problem of having the opponent's move being selected _after_ the bot's move,
which is not how Pokémon works: both players select their moves at the same time and without knowledge of their
opponent's move. To deal with simultaneous moves, MCTS is modified to use Decoupled Upper Confidence Bound applied to
Trees (DUCT), allowing moves to be selected simultaneously.
[This paper](https://dke.maastrichtuniversity.nl/m.winands/documents/sm-tron-bnaic2013.pdf) has a good explanation of
how MCTS can be modified to work with simultaneous move games and how DUCT works.

Randomness is dealt with by sampling a single outcome of a move pair during the MCTS selection phase.
Since `poke-engine` is able to generate all possible outcomes of a move pair along with their likelihoods, the search
tree can be built out with all of this information stored in the nodes. For example, if a move has a 10% chance to miss,
there will be 2 child nodes to choose from: a 90% node where the move hits and a 10% node where the move misses. These
nodes are selected based on  their likelihoods, so the search prioritizes paths that are more likely to occur.


# Engine
## Instruction Generation
To enable fast search during battles, [poke-engine](https://github.com/pmariglia/poke-engine)
was built as a Rust rewrite of the original Python engine. `poke-engine` operates by taking a battle state `S`, a
move pair `(m1,m2)`, and generating a list of every possible outcome. However, instead of returning a list of new states
`(S1, S2, ...)`, it returns a list of `instructions` that can be applied to and reversed
from the state to mutate it. This is important as Pokémon states can become quite large and costly to copy,
especially in later generations where game mechanics are more complicated.

Here is a simplified version of what instruction generation looks like for two moves.
```text
> generate-instructions shadowball tackle

Index: 0
StateInstruction: 
	Percentage: 80.00
	Instructions:
		Damage SideTwo: 184
		Damage SideOne: 48

Index: 1
StateInstruction: 
	Percentage: 20.00
	Instructions:
		Damage SideTwo: 184
		Boost SideTwo SpecialDefense: -1
		Damage SideOne: 48
```

`poke-engine` generated instructions for the `(shadowball, tackle)` move pair. Shadow ball is a 100% accurate move that
has a 20% chance of lowering the target's special-defense by one stage and tackle is a 100% accurate move with no
additional side effects. Two outcomes are generated: one when only damage happens and the other for when the
special-defense drop occurs.

## Damage Roll Grouping
The above example omits damage rolls and critical hits for simplicity, but in practice they are quite important. In
Pokémon, every move has 16 equally likely damage rolls - one for each value between 85% and 100% of the calculated
damage. Furthermore, a move has an approximate 5% chance of being a critical hit, meaning the damage dealt can have up
to 32 unique values. It is impractical to generate instructions for every possible damage roll and critical hit during
search, as each node would have too many children and the search would be quite shallow.

`poke-engine`'s solution is damage roll grouping. When considering damage rolls, what is often important is whether a
move will cause the opponent to faint. Rather than generating instructions for every possible damage roll, it
inspects the target Pokémon's remaining health and checks which damage rolls will and will not result in them fainting.
The resulting instructions are grouped by whether they cause a faint, each group's damage is averaged, and the
likelihoods are summed.

For example, let's say that the `shadowball` in the previous example has a 75% chance to faint its target and
the target has 191 HP remaining. This example ignores critical hits for simplicity.
```text
> generate-instructions shadowball tackle

Index: 0
StateInstruction: 
	Percentage: 75.00
	Instructions:
		Damage SideTwo: 191  # faint

Index: 1
StateInstruction: 
	Percentage: 20.00
	Instructions:
		Damage SideTwo: 187  # non-faint
		Damage SideOne: 48

Index: 2
StateInstruction: 
	Percentage: 5.00
	Instructions:
		Damage SideTwo: 187  # non-faint and spdef drop
		Boost SideTwo SpecialDefense: -1
		Damage SideOne: 48
```
_At least_ 191 damage is dealt 75% of the time, and in that case no more instructions are generated due to the target
fainting. The non-fainting damage rolls of (184, 186, 188, 190) in total have a 25% chance of occurring and have an
_average_ of 187 damage, so we group them together. Since there is (usually) no practical difference between
non-fainting
damage rolls they can be grouped together and explored as a single branch during MCTS. Note that branching due to
the special-defense drop still happens for the non-fainting case.

# Client
The client is responsible for communicating with Pokémon Showdown during a battle, keeping track of the battle state,
and calling `poke-engine` to perform MCTS.

## Hidden Information Extraction

Pokémon  is a game of hidden information, and to perform well the client needs to do more than just parse the
[Pokémon Showdown Protocol](https://github.com/smogon/pokemon-showdown/blob/master/sim/SIM-PROTOCOL.md) accurately into a battle state.
It needs to understand how to identify hidden information and use it to make better decisions.
These are some ways that Foul Play does this:

- Every time damage is dealt or taken by the opponent's Pokémon, Foul Play uses `poke-engine` to
simulate the damage roll and get a better idea of the opponent's Pokémon's stats.
- If move priorities match, the move-order reveals the min or max speed of the opponent's Pokémon
- If a weather was set by the opponent and lasts beyond 5 turns, the opponent's Pokémon must have the corresponding
weather extension item.
- If a Pokémon uses a status move, it is definitely not holding an Assault Vest
- If a Pokémon switches in and takes hazard damage, it definitely does not have Heavy-Duty Boots

All of these are extremely important for performance. The more information Foul Play has, the more accurately it can
predict sets.

## Set Prediction

For Foul Play set prediction is as important as, if not _more_ important than, accurate and fast search.
This becomes obvious when comparing Foul Play's performance in formats with known sets (like random battles)
to formats with open team building (like OU or UU).

Foul Play keeps a large list of possible sets for each opposing Pokémon and filters them down as the battle progresses
using the techniques mentioned above. In random formats each Pokémon has a small number of possible sets,
often fewer than a dozen, giving Foul Play a pretty good idea about what to expect.

In standard formats it gets more complicated, predictions rely on a combination of publicly available data:
- [Smogon usage stats](https://www.smogon.com/stats/)
- scraped teams from the [Smogon forums](https://www.smogon.com/forums/)
- [public replays](https://replay.pokemonshowdown.com/)

When it needs to select a move, Foul Play samples several possible battle states by filling in the unknown attributes
of the opponent's team. Each data source provides the likelihood of each attribute, so Foul Play can prioritize more
likely scenarios.

# Performance & Results

Foul Play performs best when it has reliable information about the opponent’s team, which is why random formats tend to
have stronger results.

While peak ELO and rank are listed, GXE\* and Glicko ratings, once stabilized,
are a more accurate metric for performance as competitive Pokémon can be a relatively high variance game.
Each GXE shown was measured only after the Glicko deviation dropped below 50, to ensure a stable rating.

## Standard Formats
| Format | GXE | Peak ELO | Peak Ladder Rank |
|--------|-----|----------|------------------|
| gen9ou | 80% | 1879     | Top 100          |
| gen5ou | 76% | ~1600    | Top 200          |
| gen3ou | 71% | 1815     | 4                |


## Random Formats
| Format                | GXE | Peak ELO | Peak Ladder Rank |
|-----------------------|-----|----------|------------------|
| gen1randombattle      | 75% | ~1450    | Top 500          |
| gen2randombattle      | 85% | ~1500    | Top 50           |
| gen3randombattle      | 82% | ~1600    | Top 20           |
| gen4randombattle      | 85% | 1728     | 3                |
| gen5randombattle      | 80% | 1600     | Top 30           |
| gen9randombattle      | 88% | 2341     | Top 50           |
| gen9randombattleblitz | 84% | 2018     | 1                |
| gen9battlefactory     | 81% | 1624     | 1                |

\*GXE is an interpretation of Glicko that estimates your chance to defeat a **random** opponent on the ladder

## Screenshots

### Random Battles
![randbats.png](randbats.png)

![gen12randbats.png](gen12randbats.png)

### Rank 3 Gen4 RandomBattle
![gen4rb.png](gen4rb.png)

### Rank 1 Gen9 RandomBattleBlitz
![rank1rbb.png](rank1rbb.png)

## Standard Battles

### Gen9 OU
![top100-medal.png](top100-medal.png)
![gen9ou.png](gen9ou80gxe.png)

### Gen 9 Battle Factory
![rank1bf.png](rank1bf.png)

### Gen3 OU Rank 4
![rank4gen3ou](rank4gen3ou.png)

## Replays

##### [Gen9OU Win](https://replay.pokemonshowdown.com/gen9ou-2311314676)
<iframe src="/embed/gen9ou-win.html" width="90%" height="430" style="border:none;"></iframe>

##### [Gen9OU Loss](https://replay.pokemonshowdown.com/gen9ou-2311209188)
<iframe src="/embed/gen9ou-loss.html" width="90%" height="430" style="border:none;"></iframe>

##### [Gen3 OU Win](https://replay.pokemonshowdown.com/gen3ou-2347439770-3nnrlnb5yr9wxp0s0rf32qfk1ibmh6gpw)
<iframe src="/embed/gen3ou-win.html" width="90%" height="430" style="border:none;"></iframe>

##### [Gen3 OU Loss](https://replay.pokemonshowdown.com/gen3ou-2347328488)
<iframe src="/embed/gen3ou-loss.html" width="90%" height="430" style="border:none;"></iframe>

##### [Gen9 RandomBattle Blitz Win](https://replay.pokemonshowdown.com/gen9randombattleblitz-2344182865?p2)
<iframe src="/embed/gen9randombattleblitz-win.html" width="90%" height="430" style="border:none;"></iframe>

##### [Gen9 RandomBattle Blitz Loss (vs MichaelderBeste!)](https://replay.pokemonshowdown.com/gen9randombattleblitz-2343347074)
<iframe src="/embed/gen9randombattleblitz-loss.html" width="90%" height="430" style="border:none;"></iframe>

