```
---
layout: post
Correlated Randomness in Slay the Spire
comment_issue_id: 2
---
```

Here are three true statements about the game of Slay the Spire:

1. If your first combat encounter is not a Cultist, then your first mystery room (? room) will be an event. If your second combat encounter is a cultist, then there is only about a 25% chance that your second mystery room will be an event. And if your second mystery room turns out to be your second combat encounter in act 1, then that combat encounter will be against a Cultist.
2. If your first card reward from your first encounter is uncommon, then a potion will drop as well (assuming no potions from Neow). If your second card reward is uncommon but your first is common, then as long as you don't encounter any potions beforehand, the third combat of the run will drop a potion.
3. If your third card reward from your first encounter is uncommon, and you have five combat encounters before the first treasure chest in act 1, then the chest is very likely to be a small chest. If you have six combat encounters before the first chest and your third card reward is uncommon, then the chest is very likely to contain gold (but also a lower-rarity relic).

At first glance, these statements might seem crazy. What does the type of your first combat encounter have to do with the first mystery room? And how could the rarity of your card rewards influence your potion drops? It all comes down to the way randomness works in Slay the Spire, and in particular, correlations between the different random number generators.

## The random number generators of Slay the Spire

When you start a game of Slay the Spire, you are assigned (or choose) a 64-bit seed. This seed is used to initialize several random number generators, each in charge of a different aspect of the game. But many of these random number generators are initialized to the same state. Specifically, this piece of code is executed at the start of a run:

```java
public static void generateSeeds() {
    logger.info("Generating seeds: " + Settings.seed);
    monsterRng = new Random(Settings.seed);
    eventRng = new Random(Settings.seed);
    merchantRng = new Random(Settings.seed);
    cardRng = new Random(Settings.seed);
    treasureRng = new Random(Settings.seed);
    relicRng = new Random(Settings.seed);
    potionRng = new Random(Settings.seed);
    // The following rngs are actually re-initialized each floor. We will discuss them later.
    monsterHpRng = new Random(Settings.seed);
    aiRng = new Random(Settings.seed);
    shuffleRng = new Random(Settings.seed);
    cardRandomRng = new Random(Settings.seed);
    miscRng = new Random(Settings.seed);
}
```

Random number generators always produce the same sequence of values, when starting at the same state. Therefore, to use an example, the first value produced by `monsterRng` will match the first value produced by `eventRng`.  However, there are many ways to interpret the bits produced by a random number generator. Often, the game will ask for a random real number between 0 and 1, or a random integer between 0 and 99. These two requests do very different things to the raw output of the random number generator. Examining the [implementation](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/math/RandomXS128.java), we see that generating a [random float between 0 and 1](https://github.com/libgdx/libgdx/blob/aa01d6f194f45eb1ea7f0465e98fcbcc3da98cf6/gdx/src/com/badlogic/gdx/math/RandomXS128.java#L133) throws away all but 24 of the bits of randomness, then divides the resulting number by 2^24. But [generating a random int between 0 and 99](https://github.com/libgdx/libgdx/blob/aa01d6f194f45eb1ea7f0465e98fcbcc3da98cf6/gdx/src/com/badlogic/gdx/math/RandomXS128.java#L109) performs a right shift on the original 64-bit random output (to force a positive long), then takes the remainder after dividing by 100.

The first calls to `cardRng` and `potionRng` in a run ask for a random integer from 0 to 99, but the first calls to `eventRng` and `monsterRng` ask for a random number between 0 and 1. Therefore, we can predict the result of the first call to `potionRng` by knowing the result of the first call to `cardRng`, but we cannot infer what the result of the first call to `eventRng` was by observing `cardRng`. For that, we would use the first call to `monsterRng`.

## Predicting mystery rooms using combat encounters

The code to generate the first three regular combat encounters of act 1 does something like this (all variable and function names except `monsterRng` are mine:

```pseudocode
combat_encounters = (empty ArrayList)
while size(combats) < 3:
	random_output = monsterRng.random()
	chosen_monster = get_monster(random_output)
	if chosen_monster not in combat_encounters:
		add chosen_monster to combat_encounters
```

Here, `get_monster` takes a random number from 0 to 1 and turns it into a combat encounter. In order to prove my first bullet point in the introduction, all we need to know is that any value less than 0.25 results in a Cultist. But in order, there is an equal chance of Cultist, then Jaw Worm, then 2 Lice, then Small Slimes.

The code to generate the result of a mystery room looks something like this:

```pseudocode
random_output = eventRng.random()
if random_output < monster_chance:
	return MONSTER
random_output -= monster_chance
if random_output < shop_chance:
	return SHOP
random_output -= shop_chance
if random_output < treasure_chance:
	return TREASURE
else:
	return EVENT
```

Here, `monster_chance` starts at 10%, `shop_chance` starts at 3%, and `treasure_chance` starts at 2%. Whenever the relevant choice is not selected, its chance increases by the stated values (10% / 3% / 2%). When a monster, shop, or treasure is selected, its chance resets to the starting. Thus, if the first call to `eventRng` produces 0.12, then the first mystery room is a shop, and the chances of a monster, shop, treasure, and event respectively become 20%, 3%, 4%, and 73% for the next mystery room.

Putting everything together, we know that if the first combat is not a cultist, then the first call to `monsterRng` produces at least 0.25. But because `monsterRng` starts at the same state as `eventRng`, the first call to `eventRng` also produces a value of at least 0.25. Because any value greater than 0.15 results in an event, the first mystery room of the game must be an event.

For the second part of the statement, suppose we know that the first combat of the game was Small Slimes and the second combat of the game was a Cultist, and we would like to predict the chance of getting an event in the second mystery room. Since the first combat of the game was not a Cultist, the first mystery room of the game was an event. Therefore, the chance of an event in the second mystery room is 70%. If the second call to `eventRng` is greater than 0.3, an event will be produced. Letting $X$ be the value of this second rng call and letting $C$ be the event that the second combat of the game is a Cultist, we have:
$$
\begin{align*}
\mathbb{P}(X > 0.3 | C) &= \frac{\mathbb{P}(C | X > 0.3) \mathbb{P}(X > 0.3)}{\mathbb{P}(C)}\\
&= \frac{\frac{1}{3} \left( \frac{0.25}{0.7} \right) 0.7}{\frac{1}{3}} && \text{(.25/.7 to roll slimes again given $X > 0.3$)}\\
&= 0.25
\end{align*}
$$
So in this case, we have only a 25% chance of getting an event in the second room. The chance would only be 20% if the first combat encounter was a Jaw Worm, though. As the number of observed calls to `eventRng` and `monsterRng` increases, it becomes more difficult to do any of these calculations by hand. The main confounding factor is that after the first call to `monsterRng`, a random number of calls are discarded between each outcome that is observed, as the same combat encounter is rolled again. A computer program could perform the precise calculations, but I am personally not interested in writing such a program. I would like to see the RNG predictability issue fixed, instead.

## Predicting potion drops using card rarity

Another instance where the RNG calls can line up involves potion drops and card rarity. This is best illustrated by examining the first combat of the run. Assuming no potions have been generated on floor 0, the first call to `potionRng` occurs after the first combat of the game. Unlike with `eventRng` and `monsterRng` above, this time, the call is asking for a random integer between 0 and 99 (inclusive). If the outcome is less than 40, then a potion will drop from the first combat encounter. In most cases, the first call to `cardRng` also occurs after the first combat. Just as with `potionRng`, this call asks for a random integer between 0 and 99, and is used to determine the rarity of the first card. If the outcome is less than 35, the card will be uncommon (again, barring taking potions or certain boss relics from Neow). Therefore, we can conclude that if the first card in your first card reward is uncommon, you will also get a potion drop.

We can extend this idea a little bit further. Suppose you are entering your third combat encounter of the run, but still have not encountered any potions, including in shops or from Alchemize. A potion will then drop if the third call to `potionRng` returns a value less than 60. The third call to `cardRng` determines the rarity of your second card reward. If that card is uncommon, then we can conclude that the third call to `potionRng` is less than 36, and therefore the a potion will drop from this combat. If the card is common, we have only about a 38% chance of getting a potion.

However, once the first potion appears, it is difficult to use this method to predict future potion drops, as a random, unknown number of calls to `potionRng` will be made to determine which potion to generate. This is similar to the issue in using the correlation between `eventRng` and `monsterRng` late in the run, but worse, as the number of potion calls is much more variable, and less likely to be the smallest possible number.

## Predicting/Manipulating chests

When you enter a treasure room and open its chest, the type of chest and its contents are determined through calls to `treasureRng`. As with the card rarities and potions, `treasureRng` requests a random number between 0 and 99 to determine the chest type. If the number is less than 50, a small chest will be generated. Then, when the chest is opened, another number between 0 and 99 is requested. This determines the rarity of the relic offered. For a small chest, a common relic is generated for any roll less than 75. Gold is generated if the roll is also less than 50. The tricky thing with `treasureRng` though is that it is not only used for treasure rooms. It also selects the amount of gold dropped from combats, resulting in one extra call per combat encounter.

If we want to use the correlation between `cardRng` and `treasureRng` to predict chests and their contents, we first need to know how many combats are occurring before the treasure room. If there are 5 combats first, then the sixth call to `treasureRng` will determine the type of chest. The sixth call to `cardRng` usually also determines the rarity of the third card reward (unless a duplicate card was generated and discarded, which is rather unlikely). Again, if this third card is uncommon, then the relevant roll was less than 50. Therefore, we can expect a small chest.

If instead of five combats before the treasure room, we only have four combats before the treasure room, we cannot predict the type of chest which will appear. However, assuming the third card reward is still uncommon, we can expect that the chest will contain the lowest possible rarity of relic, but will also contain gold. The full details about the relic rarities and gold amounts found in chests can be found in my [reference spreadsheet](https://docs.google.com/spreadsheets/d/1ZsxNXebbELpcCi8N7FVOTNGdX_K9-BRC_LMgx4TORo4/edit#gid=113279270&range=A24:B41), for the curious.

## Randomness in combat

As I mentioned earlier, five of the random number generators are re-initialized each floor, to the same state. These largely are used to handle the different types of randomness that occur during combat, including the max HP of enemies, the AI of enemies, and the outcomes of random cards and potions. Luckily, I can't think of too much you can do to exploit this. For example, there does not seem to be a way to predict the AI of any monsters based on their random HP values.

## Plea to the developers

If Casey and Anthony happen to see this post, I would like to please request a fix for this bug. Luckily, the opportunities for exploiting the interaction between the different random number generators seem to be fairly limited, based on what I currently believe. However, I believe players should never be incentivized to exploit the predictability of rng in order to play optimally. Maximizing the benefit from this predictability requires intimate knowledge of implementation details of the game, and in my opinion, is tedious and unnatural. Even just having the knowledge in this post feels to me like an exploit that is always accidentally active. While I and many others will continue to play, consciously ignoring the existence of the bug, it would at least grant significant peace of mind to know that independent events are truly independent. I know I have reported many bugs for the game, but this is the one that I would currently most like to see fixed.