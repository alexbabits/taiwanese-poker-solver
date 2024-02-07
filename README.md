## Road Map

🟢 Function that can take in 12 unconfigured cards and exhaustively find all combinations of (4, 4, 2, 2) with the NLHE strong and weak hand requirements.

🟢 Function that can take in both player's configured hands and a board, and compute the winner and exact points.

🟢 Function to evaluate batches of games.

🟡 Try to compute a sufficient amount of batches brute force, otherwise we will need algorithm to reduce brute force method.

🔴 Implement any regret minimization algorithm that is time efficient enough to produce useful Nash equilibrium results.

🔴 Swap out the toy game PLO hand to PLO8 once we have a ranking mechanism for PLO8, changing relevant things that need done for that.

## Taiwanese Poker Rules and Points:
Usually a 2 player game where each player is dealt 12 cards from a single 52 card deck.

**Unconfigured**: The 12 cards initially dealt to the player are "unconfigured". 
**Configured**: The player must sort cards into four cateogries of hands (PLO8, PLO, NLHE strong, NLHE weak). Once the player sorts the cards, they are considered "configured". The board is then dealt and winner is determined based on the points system described below. Assume 1 point = $1.

For the NLHE hands, their "strength" is based on their absolute strength BEFORE the board is dealt: (22 is stronger than AK). (AA is strongest, 23 is weakest). If a player makes two NLHE hands the same (KT and KT), then the player arbitrarily places them into the stronger and weaker category.

1. Holdem Hand (2 cards) (Weaker) - 1 Point
2. Holdem Hand (2 cards) (Stronger) - 2 Points
3. PLO Hand (4 cards) - 3 Points
4. PLO8 Hand (4 cards) - 4 Points (+4 for win all, +3 for Win 3/4, +2 for Win 2/4, +1 for Win 1/4, +0 for lose all)


--------------------------------------------------------------------------------------------------------

## Simplified Toy Game
Because there is no PLO8 Poker Hand Evaluator for python I need to first make a solver that involves just NLHE and PLO, replacing the PLO8 with another PLO hand:

1. Holdem Hand (2 cards) (Weaker) - 1 Point
2. Holdem Hand (2 cards) (Stronger) - 2 Points
3. PLO Hand (4 cards) - 3 Points (Order arbitrarily chosen by player before board is dealt)
4. PLO Hand (4 cards) - 4 Points (Order arbitrarily chosen by player before board is dealt)

The order of the PLO hands (stronger/weaker) doesn't matter when we change the PLO8 hand to a PLO hand because in the real Taiwanese game the PLO and PLO8 hands are always "separate" and require no strength ranking sorting before the board is dealt. We don't have to worry about the order of the tuple for the PLO hands, the player can arbitrarily place each PLO hand into one of the categories should be fine for our toy game, and fine for adaptation from (PLO, PLO) to (PLO, PLO8).

--------------------------------------------------------------------------------------------------------

## Project Architecture Goals

* A way to efficiently determine who wins, and by how many points.
* A way to pass in bulk batches of P1 and P2 hand configurations and boards.
* A way to efficiently converge on a Nash Equilibrium solution based on batches passed in.


--------------------------------------------------------------------------------------------------------

## Poker Hand Evaluation Theory

Hierarchy of Strength: royal flush > straight flush > quads > full house > flush > straight > three of a kind > two pair > one pair > High Card...

* Royal Flush: A, K, Q, J, T all the same suit.
* Straight Flush: 9, 8, 7, 6, 5 all the same suit.
* Four of a Kind: J, J, J, J, 7. All 4 cards the same rank.
* Full House: T, T, T, 9 ,9. Three of a kind, with a pair.
* Flush: 4, J, T, 9, 8. Five cards of same suit, but NOT in sequence
* Straight: 9, 8, 7, 6, 5. Five cards in sequence, but NOT of same suit.
* Three of a kind: 7, 7, 7, K, 3. Three cards of same rank.
* Two Pair: 3, 3, 4, 4, Q. Two different pairs.
* One Pair: K, K, 8, 4, 7. Two cards of same rank.
* High Card: A, 8, 9, 4, 3. None of the above hands, highest cards rank plays.

| Hand Value      | Unique   | Distinct |
|-----------------|----------|----------|
| Straight Flush  | 40       | 10       |
| Four of a Kind  | 624      | 156      |
| Full House      | 3744     | 156      |
| Flush           | 5108     | 1277     |
| Straight        | 10200    | 10       |
| Three of a Kind | 54912    | 858      |
| Two Pair        | 123552   | 858      |
| One Pair        | 1098240  | 2860     |
| High Card       | 1302540  | 1277     |
| TOTAL           | 2598960  | 7462     |


--------------------------------------------------------------------------------------------------------

## PHE Library Used

```python
# "PH Evaluator" Examples
# Should be able to also use integers based on the ` cards_to_int` dictionary.
# I think the first 5 cards should always be the board?
def determine_players_NLHE_hand_strength():
    rank1 = evaluate_cards("9c", "4c", "4s", "9d", "4h", "Qc", "6c")
    rank2 = evaluate_cards("9c", "4c", "4s", "9d", "4h", "2c", "9h")

def determine_players_PLO_hand_strength():
    rank1 = evaluate_omaha_cards("4c", "Qc", "6c", "7s", "8s", "2c", "9c", "As", "Kd")
    rank2 = evaluate_omaha_cards("4c", "Qc", "6c", "7s", "8s", "6s", "9s", "Ts", "Js")
```

```python
# This should be an exact replica of the dictionary used by PH Evaluator. Just for reference if you need.
# https://github.com/HenryRLee/PokerHandEvaluator/blob/master/python/tests/test_card.py
card_to_int = {
    "As": 51, "Ah": 50, "Ad": 49, "Ac": 48,
    "Ks": 47, "Kh": 46, "Kd": 45, "Kc": 44,
    "Qs": 43, "Qh": 42, "Qd": 41, "Qc": 40,
    "Js": 39, "Jh": 38, "Jd": 37, "Jc": 36,
    "Ts": 35, "Th": 34, "Td": 33, "Tc": 32,
    "9s": 31, "9h": 30, "9d": 29, "9c": 28,
    "8s": 27, "8h": 26, "8d": 25, "8c": 24,
    "7s": 23, "7h": 22, "7d": 21, "7c": 20,
    "6s": 19, "6h": 18, "6d": 17, "6c": 16,
    "5s": 15, "5h": 14, "5d": 13, "5c": 12,
    "4s": 11, "4h": 10, "4d": 9, "4c": 8,
    "3s": 7, "3h": 6, "3d": 5, "3c": 4,
    "2s": 3, "2h": 2, "2d": 1, "2c": 0
}
```

--------------------------------------------------------------------------------------------------------

## Simplified Game Theory Defintions
* **Monte Carlo Simulations**: Used to efficiently explore the outcomes of actions by repeatedly running simulations based on random or targeted areas.
* **Utility**: Value attributed to a certain action during a node in a round based on a fixed perspective, where the other player's action is fixed. In zero sum games, the utility of an action from Alice will result in the opposite utility for * Bob.
* **Regret**: Difference between utility of the action we chose, and the utility of the other actions at that node.
* **Nash equilibrium**: State where none of the players can increase their expected utility by changing their strategy.
* **Regret Minimization**: Seeks a Nash Equilibrium by updating strategy to minimize regret. It's a form of rudimentary reinforcement learning. We want to match regrets, meaning more promising configurations of hands should be chosen more often.
* **CFRM**: Counterfactual Regret Minimization Algorithm for finding Nash Equilibrium in two-player zero-sum games, iteratively playing the game against itself, minimizing regret for not having played alternative strategies.
* **EV**: Expected Value of an action in terms of utility in regard to achieving Nash Equilibrium.


**Rock Paper Scissors**:
* Alice plays rock. Bob plays paper. 
* Alice's action to play rock results in utility of -1 for her for that round iteration.
* The utility of paper would have been 0, and +1 for scissors. 
* Alice regrets not playing paper, and even more so regrets not playing scissors. 
* Alice's regret with regard to paper: `u(paper, paper) - u(rock, paper) = 0 - (-1) = +1`. 
* Alice's regret with regard to scissors: `u(scissors, paper) - u(rock, paper) = +1 - (-1) = +2`

The general idea is to calculate the regret of actions, and update the strategy based on the regret for the next iteration. If you compute the average of strategies over many iterations, you should end close to Nash equilibrium.

The goal of CFRM is to iteratively refine strategies to minimize regret, thereby converging towards a Nash Equilibrium without needing to exhaustively compute every possible outcome.

--------------------------------------------------------------------------------------------------------

## Combination Math

Order does not matter when configuring an unconfigured 12 card hand into buckets of (4,4,2,2). This means any way you want to compute it, it's always the same number of combos.

NOTE: User will always define and input Player 1's exact 12 unconfigured cards that get dealt to him. (And for simplification, we could also have player 2's exact 12 unconfigured cards be known).

With infinitely fast computation we could brute force a perfect answer on how to optimize the configuration of the unconfigured 12 cards to produce the highest expected Value (EV) and thus (minimized regret) as close to Nash Equilibrium as possible. This assumes the user will input a specific hand to study:

* Player 1 has a specific hand they want to look at = 1
* All combinations of player 2 hands must be dealt `C(40,12)` = 5,586,853,480 (40 remaining cards after P1 is dealt)
* All combinations of boards must be dealt `C(28,5)` = 98,280. (28 remaining cards, board is 5 cards)
* All configurations of player 1 hands must be exhausted. `C(12,4) * C(8,4) * C(4,2) * C(2,2)` = 207,900
* All configurations of player 2 hands must be exhausted. `C(12,4) * C(8,4) * C(4,2) * C(2,2)` = 207,900

Total combinations:
1 * 5,586,853,480 * 98,280 * 207,900 * 207,900 = 2.37 x 10^25

Test sample:
1 * 1 * 50 * 50 * 50 = 125,000

This small sample assumes user inputs BOTH player 1 and player 2 exact hands. This sample represents every 50 configurations of player 1 hand with every 50 configurations of player 2 hand with all 50 boards. Computed in 7 seconds. Ideally, we would want something at least like 2,000 player 1 and player 2 configurations (~1% of total), and 3,000 boards (3% of total board possibilities).

--------------------------------------------------------------------------------------------------------

## Poker Evaluation Resources

* PH Evaluator: https://github.com/HenryRLee/PokerHandEvaluator
* 7,462 length hashtable for hand ranks: http://suffe.cool/poker/7462.html
* Very good YT Video on topic: https://www.youtube.com/watch?v=TM_sMACxSzY
* Cactus Kev Poker Hand Evaluator Homepage: http://suffe.cool/poker/evaluator.html
* Rust PLO8 (PLO hi-lo) Poker Hand Evaluator: https://github.com/Nydauron/playing-cards 
* Java PLO8 (PLO hi-lo) Poker Hand Evaluator: https://github.com/gordwilling/OmahaHiLo 
* Article on hash tables for hand rankings: https://joshgoestoflatiron.medium.com/july-17-evaluating-poker-hands-with-lookup-tables-and-perfect-hashing-c21e056da130


## Monte Carlo Regret Minimization & Nash Equilibrium Resources

* Good overview: http://modelai.gettysburg.edu/2013/cfr/cfr.pdf
* Python Implementation: https://github.com/LuanAdemi/CounterfactualRegretMinimization/tree/master
* Another good overview: https://mlanctot.info/files/papers/nips09mccfr.pdf
* Yet another good overview: https://nn.labml.ai/cfr/index.html
