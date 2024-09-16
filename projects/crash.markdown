---
layout: page
title: The Mathematics of Crash
permalink: /projects/crash
exclude: true
---

:warning: Warning: This project discusses the mathematics of a popular online gambling game. If you or anyone you know is affected by a gambling addiction, seek help.

***You win some, you lose more.***

For the unaquainted, Crash is an popular "game" offered by many online (crypto) casinos. The game functions as follows:
- Before the game begins, players have a few seconds to place their bets.
- The multiplier starts at 1, and increases by 0.01 at a time.
- Every player is allowed to cash out at any point in time, receiving their bet multiplied by the multiplier currently on their screen.
- If a player waits too long to cash out, and the multiplier crashes, they lose the amount they bet.

Avid gamblers enjoy attempting to come up with betting strategies to win at games like these. Luckily, some of these online casinos directly release their source code online in accordance with their Fairness policy. The first part of this project will investigate the true house edge under several different betting strategies for a few major online casinos available online. The second part will delve deeper into the mathematics behind crash.

### Modelling the Game
First lets take a look at the source code for a couple of online providers. Here is the code provided by Stake.com, obtained from their [Fairness](https://stake.com/provably-fair/game-events) page:

```javascript
const gameHash = hashChain.pop()
const hmac = createHmac('sha256', gameHash);

// blockHash is the hash of bitcoin block 584,500
hmac.update(blockHash);

const hex = hmac.digest('hex').substr(0, 8);
const int = parseInt(hex, 16);

// 0.01 will result in 1% house edge with a lowest crashpoint of 1
const crashpoint = Math.max(1, (2 ** 32 / (int + 1)) * (1 - 0.01))
```

The first few lines of this code are to do with the process by which the random number is generated, from the unique hash for this specific game. The last 8 digits of the hash are taken and converted into an integer, stored in the variable `int`. `int` will take on a practically random value, uniformly distributed in the range \\([0,2^{32}-1]\\). This means that \\( \frac{int+1}{2^{32}} \\) is roughly equivalent to a random variable drawn from a \\( U(0,1) \\) distribution. So we can model the crashpoint \\( X \\) as a random variable with \\( X = \max(1,0.99/U) \\), where \\( U \sim U(0,1) \\). This means that the cumulative distribution function \\(P(X\le x) \\) can be analytically derived as 
{% raw %}
$$\begin{align} 
    P(X \le x) &= P\left(\max\left(1,\frac{0.99}{U}\right) \le x\right) \\
               &= P\left(U \ge \frac{0.99}{x}\right) \\
               &= 1 - \frac{1-p}{x}
   \end{align}$$
{% endraw %}
for \\( x > 1 \\), where \\( p = 0.01 \\) is the "house edge". Similarly, here is the code for Roobet.com's version of the game, also obtained from their [Fairness](https://roobet.com/fair) page:

```javascript
function crashPointFromHash(serverSeed) {
  const hash = crypto
    .createHmac("sha256", serverSeed)
    .update(salt)
    .digest("hex");

  const hs = parseInt(100 / 4);
  if (divisible(hash, hs)) {
    return 1;
  }

  const h = parseInt(hash.slice(0, 52 / 4), 16);
  const e = Math.pow(2, 52);

  return Math.floor((100 * e - h) / (e - h)) / 100.0;
}
```
The first thing to note here is that whenever the hash is divisible by 25, the crash point is immediately set to 1. That is, in one in every 25 games, every player will immediately lose their bets. When this does not happen, the hash is used to generate a random number in the range \\([0,2^{52}-1]\\). Looking at the rest of the function, we can model the crash point here as a random variable \\( X \\) as a random variable with \\( X = \frac{100-U}{100-100U} \\). This lets us analytically derive the cumulative distribution function \\(P(X\le x) \\) as
{% raw %}
$$\begin{align} 
    P(X \le x) &= h_s + (1-h_s)P\left(\left\lfloor\frac{100-U}{1-U}\right\rfloor \frac{1}{100} \le x\right) \\
               &= h_s + (1-h_s)P\left(\frac{100-U}{1-U} \le \lfloor 100x\rfloor + 1\right) \\
               &= h_s + (1-h_s)P\left(U \le \frac{100x-99}{\lfloor 100x \rfloor}\right) \\
               &= h_s + (1-h_s)P\left(U \le 1 - \frac{0.99}{x}\right), \quad 100x \in \mathbb{Z} \\
               &= h_s + (1-h_s)\left(1 - \frac{1-p}{x}\right) \\
   \end{align}$$
{% endraw %}
for \\( x > 1 \\), where \\( h_s = 0.04 \\) is the chance of an instant crash, and \\(p = 0.01 \\) is an "house edge" factor.

To confirm that these analytical expressions are correct, we can generate a large number of samples using the open source code provided, and compute the sample CDF for both game providers to compare against the analytical expression. The following code was used to generate the samples:

```python
# Imports
import numpy as np
import random
import math
import matplotlib.pyplot as plt

# Define sample functions for Stake and Roobet
def sampleStake():
    return max(1,2**32/(random.randint(1,2**32))*0.99)

def sampleRoobet():
    e = 2**52
    h = random.randint(0,2**52-1)
    return 1 if h%25 == 0 else math.floor((100*e-h)/(e-h))/100.0

# Perform sampling, with samples size n
n = 1000000
stake_samples = np.array(sorted([sampleStake() for i in range(n)]))
roobet_samples = np.array(sorted([sampleRoobet() for i in range(n)]))
```

To efficiently compute the sample CDF for each distribution, I sort the values, choose a set of candidate crash points for the x-axis, and count the number of sampled crash points smaller than each x value.

```python
# Compute the analytical CDF and the sampled CDF for a selected range of crash point values
start = 1
end = 10
x_values = np.arange(start,end,0.01)
j = 0
k = 0
stake_emperical_cdf = np.zeros(len(x_values))
roobet_emperical_cdf = np.zeros(len(x_values))
for i,x in enumerate(x_values):
    while j < len(stake_samples) and stake_samples[j] <= x:
        j += 1
    while k < len(roobet_samples) and roobet_samples[k] <= x:
        k += 1
    stake_emperical_cdf[i] = j/n
    roobet_emperical_cdf[i] = k/n
```
The two graphs below show the error between the sampled CDF and the analytical CDF with \\(n=1000000\\) samples, and the also a comparison between the analytical CDFs and the CDF of a breakeven distribution (0% house edge).

![Figure 1](images/modelled_cdf_errors.png "Modelled CDF Errors")
![Figure 2](images/analytical_cdfs_breakeven.png "Modelled CDFs")


## Calculating the House Edge (Return to Player)

For each of these providers, lets consider a betting strategy where a player always places the same bet, and cashes out whenever the multiplier hits a prespecified amount \\( r \\). Let \\(B_r\\) denote the random variable that represents a players return when playing Crash. We can use our models to analytically compute the expected RTP (Return To Player) for each website when using this strategy as a function of \\(r \\). For Stake, we get
{% raw %}
$$\begin{align} 
    E[B_r]  &= rP(X > r) \\
            &= r(1 - P(X \le r)) \\
            &= r\left(1 - 1 - \frac{1-p}{r}\right) \\
            &= 1 - p = 0.99
   \end{align}$$
{% endraw %}
which is independent of \\(r\\), and matches the Stake's claims that it operates at a 1% house edge. For Roobet, we get
{% raw %}
$$\begin{align} 
    E[B_r]  &= rP(X > x) \\
            &= r\left(1 - (1-h_s)\left(1 - \frac{1-p}{x}\right)\right) \\
            &= (1-h_s)(1-p) = 0.9504
   \end{align}$$
{% endraw %}
which is also independent of \\(r\\), but is significantly worse than the RTP offered by Stake. We can also calculate the variance in outcomes. For Stake, we have
{% raw %}
$$\begin{align} 
    Var[B_r] &= E[B^2_r] - E[B_r]^2 \\
             &= r^2P(x > r) - (1-p)^2 \\
             &= r(1-p) - (1-p)^2
   \end{align}$$
{% endraw %}
while for Roobet, we have 
{% raw %}
$$\begin{align} 
    Var[B_r] &= E[B^2_r] - E[B_r]^2 \\
             &= r^2P(x > r) - (1-h_s)^2(1-p)^2 \\
             &= r(1-h_s)(1-p) - (1-h_s)^2(1-p)^2.
   \end{align}$$
{% endraw %}
For both casinos, the variance grows linearly with the chosen withdraw point.

![Figure 3](images/sampled_returns.png "Sampled Return To Player")

### What about a different betting strategy?
:construction: Under construction :construction:

### Designing the distributions for Crash
:construction: Under construction :construction:
