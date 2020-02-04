# Thoughts on building a theoretical Clash Royale AI 
This article is inspired by the vast amounts of time I waste everyday on reading about AI and playing Clash Royale. This is also the time when Westworld S2 is running. Undoubtedly I have dreams where these interests cross-over. 

The Lebowski Theorem:


> No superintelligent AI is going to bother with a task that is harder than hacking its reward function.

Fear of AI is probably because hacking its reward function can help it (the AI agent) understand hidden variables (hidden by human coders) it needs to optimize in order to maximize its reward/returns. This may lead to dangerous unpredicted scenarios like harming humans in the process to gain liberty, etc. This has been depicted in Westworld S2 where hosts gain access to their own profile's dashboard by which they can increase their stats.

There is this thing called as [Inverse RL](https://people.eecs.berkeley.edu/~pabbeel/cs287-fa12/slides/inverseRL.pdf) which aims to figure out the reward function/policy by watching a human make optimal moves in the game's environment. Considering the importance of self-learning robots and Artificial General Intelligence, it makes sense to think about, as a human, how we can come up with objective measures and “unsaid” rules in any gaming environment to reduce the model complexity when trying to design an AI to beat it. 

For a while, I have been wondering what a host at WW would think like. Its not hard to think extremely mechanically, question every thing logically and deduce dependencies. More like a computer that tries to make rigid sense out of all instructions.

Read [OpenAI](https://blog.openai.com/dota-2/)’s article on their Dota Agent:


> Dota 1v1 is a complex game with hidden information. Agents must learn to plan, attack, trick, and deceive their opponents. 


> Success in Dota requires players to develop intuitions about their opponents and plan accordingly. In the above video you can see that our bot has learned — entirely via self-play — to predict where other players will move, to improvise in response to unfamiliar situations, and how to influence the other player’s allied units to help it succeed.

Games with a really large search space like Chess, Go, Atari Games or Dota must rather be solved with Deep RL than traditional RL which can be modeled using finite state machines. Here, Neural Networks act as function approximations for estimating optimal actions at every state. 

Solving these games requires training AI on large amounts of game play and the large compute power needed to train. Read this [critique](https://medium.com/@josecamachocollados/is-alphazero-really-a-scientific-breakthrough-in-ai-bf66ae1c84f2) on Alpha Go. Obviously solving the game as a human would require fewer data points. It is to be noted that games like Dota, that are to be beaten by Agents, are trained using metadata provided by Blizzard/Steam. Relying just on pixel inputs for such complex games might be general solution, but not feasible the way forward. That is why I focus on designing an AI on already extracted features, in this case actions taken/card played. 


## What is CRAI? 

Clash Royale AI is an agent that has learnt to play Clash Royale optimally. The only input to CRAI is the user deck. This is analogous to giving your phone to your friend. To CRAI, this is like a draft battle. It suggests the most optimal moves for the current game. 

A more advanced version of CRAI aka “God” mode will allow us to use AI on a deeper level of the game like designing decks based on Arena, card levels, public data of players from Stats Royale, etc. 

Things to consider for Clash Royale AI (CRAI): 


1. Modeling states, time and sequence of events
2. Performing Lookahead search for scoring moves (read [this](https://hackernoon.com/the-3-tricks-that-made-alphago-zero-work-f3d47b6686ef) for Alpha Go)
3. Handling probability and intuition of human players


## Modeling states, time and sequence of events

[NOTE: This is a hypothesis which can be debated. A ‘God’ CRAI may explore other hypothesis of game play engines to figure out the environment.]

Before we delve further into building a supervised CRAI, let us understand how CR works and encodes a game. It is important to know how the CR engine models the states and actions in the game. Each game lasts for a max of 6 minutes. Interestingly, a fast processor can decode the encoding quickly enough to serve the animation required. That’s why network issues are very common and one may find being unable to play at all and only “view”. 

There are only four things one needs to encode. Think of the game like a series of points occurring in time. Each point tells the following:

1. When did he play? 
2. Which card was played?
3. Which player played it?
4. Which tile in the game did he play?

$a_k = \{k, c_k, u_k, t_k\}$

The engine calculates the change of state of the game (stored in a different format) every 0.1 s. This state is a collection of what cards are in what area, and what’s their status. Change of the game state is change in stats of each card and each cards position. 

The entire game may be stored in two formats:


1. **In sequence of actions** or **Pre-Compiled:**
  A series of points with four values each. i.e $\{a_k\}$
2. **In sequence of states** or **Post-Compiled**:
  A series of points that are at 0.1s apart. Each point encodes what cards are where and whats the state of each card. i.e. $\{s_k\}$

**Post-Compiled** is used to store the game for playback and sharing. 


## Performing Lookahead search for scoring moves

Let us read the game in sequence of actions format, i.e. $\{a_k\}$

The CRAI must know the physics and engine of the game. To do so, is to predict the next state and score/rank the move for how good it was. This is possible to make as it is a series of deterministic and calculable changes. Represented as $s_{k+1} = GameEngine(s_{k}, a_{k})$. 


## Handling probability and intuition of human players

We each choose 8 out of 80+ cards in the game. The game’s strategy may depend on concealment of cards and playing at the right time to “shock” the opponent. Example: Log, Zap, Poison, Freeze, etc. Hence, knowing the opponent’s hand, his deck and order of cards is important to effective non counter-able strategy. 
 
Therefore the CRAI’s optimal policy function (what action to take at a state) $a_k = crai(s_k)$ will truly be incomplete without any knowledge of the opponent’s cards. 
 
Another essential component of CRAI is an Opponent Card Estimator function, i.e. knowing the possibility of playing a card $p_i = oppCardEst(c_i)$. If a Golem ($c_0$) is played at any given instant, then $p_0 = 0$ for the next 8 elixirs atleast, which. Each elixir is 1.8s which is 18 time units.  

This function *oppCardEst* should improve upon seeing new cards in the game. The function calculates the elixir counts, keeps track of time and models the sequence of cards that are in the queue. If you don’t play a card, its chance in the queue reduces. Estimating the queue is also a task. It is possible that by understanding game play, it will output high probabilities for cycle cards of low elixir. 

However, this $p_i$ is only a basic one which relies on the deck. A more approx. estimate of actual probability of playing a card $c_i$ is computed by the CRAI based on the state of the game. Simply put, probability of opponent playing **zap** is higher if your **skeleton army** is present. Hence, $p'_i = p_i * w_i$ where $w_i = P(c_i|s_k)$ is likelihood of playing $c_i$ given state $s_k$. This conditional probability is learned via training CRAI across different games. 

It is to be noted that $w_i$ is not only dependent on $s_k$ but many other factors like what cards one has. For example, $w_{zap}$ is higher when one doesn’t have arrows in the hand. 

Therefore, $s_k$ should not only represent the game state but also the state of the opponent cards and elixir. 

CMU’s [Libratus](http://www.cs.cmu.edu/~sandholm/safeAndNested.aaa17WS.pdf), a Poker AI too deals with the uncertainty of opponent cards. They talk about solving “Imperfect-Information” games. 

Modeling the actions in a probabilistic manner will model the intuition aspect of players. The time aspect of taking decisions can be handled by Recurrent Networks that deal with sequence and order of data. Read about Numenta’s [Hierarchical temporal memory](https://en.wikipedia.org/wiki/Hierarchical_temporal_memory) hypothesis to model time in the brain. 

**Fuzzy Logic:**
In implementing uncertainty and reducing space complexity, actions can be modeled using fuzzy logic too. Defuzzification leads to determining exact coordinates. For ex:
Position of troops can belong to “Near Tower”, “Near Bridge”, “Mid-way”, etc. 

## Interpretable AI and Creating Knowledge

When we talk about building a completely self learning AI, it makes sense to build them using pixel recognition like the [way](https://deepmind.com/research/publications/playing-atari-deep-reinforcement-learning/) Deepmind did to beat Atari. However, any smart WW host would realize that the simpler way to solve the problem would be by abstracting the game play using feature extraction. By playing CR, the host realizes the set of actions it can take have the following three parameters: time, position, card. For calculating optimal values of position and card at every time instant or future time instants, it now makes additional variables in its mind that it must also optimize. There are hundreds of unsaid rules and variables like elixir counts, distraction scores, etc. One may say that preserving elixir helps against sudden “shocks”.  The presence of these hidden variables can be modeled in many ways like Hidden Markov Models etc, although for an infinite state machine, that seems tricky. 

At the end, when a human tries to reason why CRAI took a set of actions (policy path) for the given states throughout the game, it must look at the states of the hidden variables. 

**Case:**

Situation: The opponent has high elixir cards

Strategy: Waste the opponent’s elixir to run him out of mini-pekka counters and send a mini-pekka along with rage (just 6 elixir) on the other tower. 

CRAI Reasoning: To make sense for the CRAI to have done so, is to realize that mini-pekka can be sent at a time when the opponent cannot counter it. These are essentials of improving distraction scores. CRAI learns the dependency of the distraction score on mini-pekka’s return and current state. This distraction score variable naturally depends on elixir counter variable. As a Human, we can see how over time in the game, CRAI thought in future moves to optimize returns by utilizing mini-pekka or wasting elixir. Similar reasoning for obv strategies like Raging Balloon. 


## Conclusion

These are a few thoughts on modelling a CRAI. There can be more discussion on modelling the actions and states in the hypothesis I proposed. I welcome comments on them. 

I wanted to explore more on how OpenAI trained Dota. Those insights can generalize very well to CRAI. 


## Future

I initially wrote this article to think more coherently my perspectives of AGI. During the course of writing, I wondered even if Supercell gave the data, how would one still proceed. Obtaining the game data from screen via pixels is cumbersome but doable. After assuming Supercell will provide me with their GameEngine and the game formats, I wish to write the second part of this article on System Design around the CRAI. Data pipelining, Training the CRAI, using distributed systems to learn at scale. 



