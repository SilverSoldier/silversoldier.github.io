---
layout: post
title:  "Notes on Game Theory"
description: An introduction to game theory
img:
date: 2020-07-04  +0648
---

Reading up on Game Theory, just for fun! Currently, it is from a bunch of books, papers and online lecture notes (it is just from the one book, but hopefully, I will be inspired to look into certain topics in depth from various resources and I will include those as well). Planning to do a chapter a day (to be modified in terms of hours or topics etc. as time passes), here goes:

# Introduction

Game theory is the strategic interaction between multiple interdependant parties, where each parties decision depends on the other parties' decisions.

Note: Difference between optimization and game theory. I was under the assumption that they were both the same, but apparently they are not. Optimization has a clear idea of whether actions are optimal or not whereas game theory deals with human actions where the definition of optimality itself might be unclear.

Most economics applications fall into game theory and so does warfare and sociology. One important point is that a player's result does not only depend on its decision, but also the decisions of other players.

## Terminology
1. **Co-operative vs non co-operative game theory**:

  In co-operative game theory, we assume binding agreements between parties is possible. Examples include various cities or countries. In non co-operative game theory, agreements are not possible. This gives the feeling of byzantine or malicious entities which might deviate from any agreements and protocols. NOTE: It just models real world entities which may not be able to communicate with each other or cannot form a binding agreement with each other.

2. **Sequential vs Simultaneous Games**:
   In sequential games, there is a strict order of play (turns). In simultaneous games (as the name suggests), players move simultaneously. This simplifies that they do not act based on the other's inputs. For instance, *rock-paper-scissors* or *sealed-bid auctions* are simultaneous games
2. **Strategies**:
	
  Strategies are the choices available to the players. In case of sequential games, it represents the set of actions that each player takes.

3. **Payoffs**:

  A numerical value assigned to each possible outcome of combination of players' strategies. It takes into account various factors - winning amount (in case the game involves money, it could represent the actual amount of money), fairness (some players may rate fairness more than amount they win) etc. In case of random combination of strategies (mixed strategy), it represents the expected payoff.

4. **Rationality**:

  We assume that each player has a set of rankings of outcomes (based on payoffs) and chooses the best alternative over all applicable outcomes. NOTE: Players are not always selfish, we incorporate their ideas of fairness etc. into the payoff calculations. Similarly, they choose a long-term optimal strategy.

5. **Evolutionary/Dynamic Game Theory**:

  A more biologically motivated subspace, where we assume that players learn from previous games and their outcomes.

# Sequential Games
## Game/Decision Trees
Game trees are used to analyse sequential games. Decision nodes represent a particular player who acts at that node. All nodes are associated with the payoff values of all players at that stage.
Branches in the game tree represent the choices that can be taken at that decision node.

At each node, the player chooses the strategy which leads to best payoff for itself at a terminal node. Each player also assumes that the subsequent player will try to maximise its payoff.

In sequential games, it is obvious that one player gets to move first. In some games, there is a first-mover advantage, where the first player can get into an advantageous position and force the others' hands.
There are also second-mover advantage games, where the second mover can act on others' choices.

# Simultaneous-Move Games with Pure Strategies
In simultaneous move games, all players move at the same time. Instead of basing their choice on the choice that the players before and after them take, players try to guess the thought process of the other players who simultaneously do the same.

# Non-linear search
No, this is not a game theory topic. Instead of reading through the chapters of the book (I admit, I got bored in the middle), I wanted to see the applications of game theory in papers etc.

I am starting with the paper on Nash Equilibrium (Non-cooperative games, by John Nash). I intended to find some interesting papers which cited it, unfortunately it seems to be too popular (11000 odd citations). Let us see where this search leads us.

## Non-cooperative games, John Nash, 1951 (wow, ancient)
Basically deals with equilibrium points in non-cooperative game settings (where players cannot collaborate or communicate).

An equilibrium point is a set of strategies, where each player's payoff is maximised if others' strategies are fixed. A point to note is that we do not include strategies where 2 players together take a joint decision to improve their payoffs (since co-operation is not allowed).

We further distinguish non-cooperative games as non-zero sum and zero sum games. A finite game only allows each player to have a finite number of alternatives

## Two person cooperative games, John Nash, 1953
Finally understood a mixed strategy. A mixed strategy involves randomly choosing (with or without using probabilities) between multiple pure strategies.

# Resources
1. [MIT](https://ocw.mit.edu/courses/economics/14-126-game-theory-spring-2016/)
2. http://faculty.econ.ucdavis.edu/faculty/bonanno/PDF/GT_book.pdf
3. https://www.theorie.physik.uni-muenchen.de/lsfrey/teaching/archiv/sose_06/softmatter/talks/Heiko_Hotz-Spieltheorie-Handout.pdf
4. 
