[reference from](http://hackaday.com/2015/12/28/v8-javascript-fixes-horrible-random-number-generator/)
根據 V8 官方描述，V8 用 PRNG (pseudo-random number generator) 實作 `Math.random()` 是十分糟糕的，V8 是 Google 為了 Chrome 所開發的 Javascript 引擎，而現在也用在 Node.js 以及其他地方，但是近六年來這樣的狀況卻是鮮少被注意，如今這問題已經有被提出以及改善。

In this article, I’ll take you on a trip through the math of randomness, through to pseudo-randomness, and then loop back around and cover the history of the bad PRNG and its replacements. If you’ve been waiting for an excuse to get into PRNGs, you can use this bizarre fail and its fix as your excuse.

一開始，來看一句名言：

	許多人認為隨機數的算數產生方法可以是 `sin` 函數來呈現，但這其中有個問題常被提出來，沒有一個數是真正的隨機產生，你只能用一些方法來隨機產生一些數，但是這些方法都不是嚴謹的算術方法。 - John von Neumann

John von Neumann 不用多說就是一個天才。他對於隨機變數有深入且重要的說明以及相關的數學定義。


###RANDOM VARIABLES

在進階機率的課程上所學的是以數學家觀點來看隨機，所以對於 `random number` 任何上過課的人應曾經苦惱過，但這不是賣弄學問，這是基礎。

數字不是隨機的。從以前到現在，我們都知道他們是甚麼。我們用數字來計數。我們甚至把數字擴展到不可數的部分以及非實數的部分。但是有個部分數字無法表示，那就是隨機。7 還是 7，從亞里斯多德到現在都是一樣的。畢竟，沒有隨機的觀點，數字更容易用在計數上。

[img random_variable]

如果想要用數學角度了解隨機，那麼就必須知道函數。我們可以從函數得到數字，這些數字不是隨機的，他們只是從隨機過程得到的結果。這個函數需要一個隨機性質的參數來產生。數學家曾表示 `隨機變數` 是一個函數，而他的值是跟世界狀態是有相關的。如果說在時間 `t` 跟世界的相關狀態就是 `St`，那麼隨機變數就是 `xt = f(St)`

如果從現在 (time t) 到明天 (time t+1) 的世界狀態是不可預測的話，那麼明天所得到的隨機變數 x 也將不可預測。如果你說得出未來世界可能的狀態以及機率，那麼你也可以算出未來 x 的值。

現在假設我擲一個骰子，舉例來說，我可以非常確定各個面朝上的狀態在世界中的機率是多少，但我不能說我明天即將擲出三或四的點數。一旦擲出，所得到的就只是簡單的數字，4 永遠就是 4。

這也就是 von Neumann 所提到的＂...沒有所謂的隨機數字－只有一些方法可以去隨機產生數字...＂如果我們知道一種方法，這個方法可以數學方式從狀態 St 到 St+1 精準預測到時間 t 的結果，那麼明天的所有結果也就不可能無法預測。

###PRNGS?
If you can write down how St evolves over time, then your function isn’t random, thus all of the computer-implemented PRNGs aren’t random either. (That’s the “pseudo-“.) Then what are they? What should they do, if they can’t produce randomness? Here’s a list of three criteria, where each one builds on the previous ones.
如果你可以寫下一段時間 St 的內容，那麼你設計的函數就不是隨機，因此所有計算機實作出來的 PRNGs 都不是隨機。這也是為什麼要加上虛擬的前綴詞。那麼他們又是甚麼，如果他們不能產生隨機，他們可以拿來做甚麼呢？這裡有三個準則，每個都建立在前一個準則上。

[img histo]

如果要拿 PRNG 來做點小事情的話，你應該會想要輸出他所有可能的值。就像你拿到 8-bit PRNG，[1] 你會想要透過他取出在 0 到 255 的每個值，基於如此，[2] 你會希望每個可能的結果都有相同的出現頻率 (以一個 uniform PRNG 來說)，最後，[3] 即使你有著前幾次的結果，你仍然無法預測出下一次結果 (xt+1)。

A PRNG that covers all possible values is said to have “full period”, and one where all outcomes occur with equal frequency is “equi-distributed”. 
These criteria are fairly straightforward to work out in math or to test — just take a lot of values and see if there are too many of one or none of another. 
這些準則對於數學或測試來說是相當的簡單明瞭－
If a PRNG does’t cover all the values, it can’t really be evenly distributed. 
If it’s not evenly distributed, you’ll be able to have a limited kind of predictive ability; if five comes up too often, just predict five.
一個 PRNG 涵蓋了所有可能出現的值，可說是 "full period"，然而所有的結果出現的頻率一致，我們可以說是 "equi-distributed"。

Predictability can be even more subtle, though, and there are a bunch of interesting statistical tests. Or course, “predictability” is a bit of a misnomer. We know how the state updates, so the word “predictability” only really makes sense if we pretend that we don’t already know St+1, and only focus on the history of the x values. We’re in von Neumann’s “state of sin” after all.

###AND JAVASCRIPT
Which brings us to V8 Javascript’s Math.random(). For the last six years, the algorithm that’s been used has been pretty horrible. So think of our three criteria presented above and have a look at the following plot of its output. (Or look up at the banner again.) Which desired criteria fail?

Untitled drawing

If you said “full-period” you were right. There are holes where random numbers just don’t occur, although a conclusive test requires more than a plot. And if you said “equi-distribution” you were also right. Have a look at those dark bands. Those are numbers that occur more frequently than they should.

Finally, if you said “unpredictability” you were mostly right. The “good” news is that the bands are basically horizontal stripes, which means that even though some y-axis values are over-represented, they don’t seem to depend on the x-axis value. But because the y-axis values are unconditionally poorly distributed, you can predict in the regions with the dark bands, and your prediction will be better than chance.

So of three possible criterion to judge a PRNG, this one scores a zero, based on simply looking at a plot of some values. The code that generated the images is here. (Test your browser’s PRNG.)

To quantify the above observations, the official V8 Javascript post that I linked above notes that the coverage is only 232 values out of the possible 252 uniformly-distributed values that a 64-bit float can represent. This is a huge failure of the “full-period” criterion.

birthday-cake
This is actually a problem
Additionally, there are short cycles that depend on the particular choice of the starting state. That is, for unlucky state choices, the period before the PRNG repeats is even shorter. Now 232 possible numbers seems like a lot, until you realize that you’re short of the available values by a factor of 220*. That is, you’re missing 99.999905% of the space. And the Birthday problem makes this shortfall a big deal.

Better tests of predictability in PRNGs will look at many higher dimensions, and test if the outcomes are dependent on each other at various lags and with varying amounts of previous output used to predict the next value. TestU01 is now the state-of-the-art in PRNG testing, and is easily downloadable so you can put your favorite PRNGs to the test if you’d like. Other, perennial favorites include the original Diehard battery of tests (get it?) and the improved Dieharder battery (will the puns stop?!). But this PRNG is so bad-looking on its face that there’s no need to beat the dead horse.

###WHAT HAPPENED?
There’s a great writeup of the whole debacle in the blog of [Mike Malone], the CTO of Betable. His company used Math.random() in Node.js on their servers to assign per-session tokens to users. They thought they were fine, because the chances of a collision were vanishingly small if the PRNG was doing its job. They estimated that they’d have a one-in-six-billion chance of a collision in the next 300 years. (They’re wrong — they can’t get more values out of the PRNG than the full state cycle, which is 264. But they’re basically right in that we shouldn’t see a collision in our lifetimes.)

In fact, they had a collision in March on a system they rolled out in February. Oops! This lead [Mike] to do a very in-depth analysis of the flawed PRNG, which is worth a read. He also points out the prophetic comment from [Dean McNamee] on the code change that introduced the flawed PRNG:

I would have gone with Mersenne Twister since it is what everyone else uses (python, ruby, etc).

Indeed.

The story of the algorithm that got chosen, “MWC1616” is even stranger. It seems that [George Marsaglia], the author of the original Diehard tests above, and developer of the Mersenne Twister, posted up one version of this routine on Jan 12, 1999 and then an improved version on Jan. 20. People who read the thread to the end got the good one, and that includes Numerical Recipes and many others. Somehow, the poor folks implementing the PRNG for V8 Javascript just got the wrong one.

###WHAT’S NEXT?
But the bug was found and patched. In the end, V8 Javascript is going with an XorShift generator, which seems to be state-of-the-art and passes all of the statistical tests in all of the test suites we mentioned above. In addition, it’s extremely fast, requiring only a few bit-shift and XOR operations. It’s new, which is always a problem, but it tests extremely well.

XorShift128+ was also merged into Mozilla and Safari as well as Chrome. You should be getting better random numbers soon.

If you want to play around with these PRNGs, here’s the code for MWC1616 (no!):

```
uint32_t state0 = 1;
uint32_t state1 = 2;
uint32_t mwc1616() {
  state0 = 18030 * (state0 & 0xffff) + (state0 << 16);
  state1 = 30903 * (state1 & 0xffff) + (state1 << 16);
  return state0 << 16 + (state1 & 0xffff);
}
```

and here’s the same for XorShift128+ (yay!):

```
uint64_t state0 = 1;
uint64_t state1 = 2;
uint64_t xorshift128plus() {
  uint64_t s1 = state0;
  uint64_t s0 = state1;
  state0 = s0;
  s1 ^= s1 << 23; s1 ^= s1 >> 17;
  s1 ^= s0;
  s1 ^= s0 >> 26;
  state1 = s1;
}
```

And if all this pseudo-RNG stuff has got you craving for some real, honest-to-goodness, no-knowledge-of-the-state-update-function, randomness you’ve got a couple of good options: use radioactive decay or combine radio noise and quantum tunneling. Finally, if you’d like your randomness certified, check out the US National Institute of Standards and Technology’s Randomness Beacon.
