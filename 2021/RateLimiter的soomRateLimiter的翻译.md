How is the RateLimiter designed, and why?   

 RateLimiter 是如何设计的，为什么？

 The primary feature of a RateLimiter is its "stable rate", the maximum rate that is should allow at normal conditions. This is enforced by "throttling" incoming requests as needed, i.e. compute, for an incoming request, the appropriate throttle time, and make the calling thread wait as much.

 RateLimiter的主要特征是它的 "稳定速率"，即在正常情况下应该允许的最大速率。这是由 "throttling(限流) "传入的请求来执行的，也就是说，对于传入的请求，计算出适当的节流时间，并让调用线程等待同样时间。



The simplest way to maintain a rate of QPS is to keep the timestamp of the last granted request, and ensure that (1/QPS) seconds have elapsed since then. For example, for a rate of QPS=5 (5 tokens per second), if we ensure that a request isn't granted earlier than 200ms after the last one, then we achieve the intended rate. If a request comes and the last request was granted only 100ms ago, then we wait for another 100ms. At this rate, serving 15 fresh permits (i.e. for an acquire(15) request) naturally takes 3 seconds.

保持QPS速率的最简单的方法是保留最后一个被批准的请求的时间戳，并确保从那时起已经过了（1/QPS）秒。例如，对于一个QPS=5的速率（每秒5个令牌），如果我们确保一个请求的批准时间不早于上一个请求后的200ms，那么我们就达到了预期的速率。如果一个请求来了，而上一个请求在100ms前才被批准，那么我们就再等100ms。在这种速度下，提供15个新的许可（即对于一个acquisition(15)请求）自然需要3秒。

It is important to realize that such a RateLimiter has a very superficial memory of the past: it only remembers the last request. What if the RateLimiter was unused for a long period of time, then a request arrived and was immediately granted? This RateLimiter would immediately forget about that past underutilization. This may result in either underutilization or overflow, depending on the real world consequences of not using the expected rate.

必须认识到，这样的RateLimiter对过去的记忆是非常肤浅的：它只记得最后的请求。如果RateLimiter有很长一段时间没有被使用，然后有一个请求到来，并立即被批准了，那该怎么办？这个RateLimiter会立即忘记过去的低利用率。这可能会导致利用不足或溢出，至于那种这取决于实际项目不使用预期速率的所产生的状况。

Past underutilization could mean that excess resources are available. Then, the RateLimiter should speed up for a while, to take advantage of these resources. This is important when the rate is applied to networking (limiting bandwidth), where past underutilization typically translates to "almost empty buffers", which can be filled immediately.

过去的利用不足可能意味着有多余的资源可用。然后，RateLimiter应该加速一段时间，以利用这些资源。这一点对于那些被限制了带宽速率的网络环境中特别重要，过去的利用率不足通常转化为 "几乎空的buffers（网络）"，这时候可以立即填补新的数据。

On the other hand, past underutilization could mean that "the server responsible for handling the request has become less ready for future requests", i.e. its caches become stale, and requests become more likely to trigger expensive operations (a more extreme case of this example is when a server has just booted, and it is mostly busy with getting itself up to speed).

另一方面，过去的低利用率可能意味着 "负责处理请求的服务器对未来的请求准备不足"，即它的缓存变得陈旧，请求变成昂贵的触发操作（这个例子的一个更极端的情况是，当一个服务器刚刚启动，它大部分时间都在忙着让自己进入状态【忙于获取数据】）。

To deal with such scenarios, we add an extra dimension, that of "past underutilization", modeled by "storedPermits" variable. This variable is zero when there is no underutilization, and it can grow up to maxStoredPermits, for sufficiently large underutilization. So, the requested permits, by an invocation acquire(permits), are served from:
 - stored permits (if available)
 - fresh permits (for any remaining permits)

为了处理这种情况，我们增加了一个额外的维度，即 "past underutilization（过去的利用不足）"，用 "storedPermits（存储许可） "变量来存储进行模拟。当没有利用不足时，这个变量为零，对于足够大的利用不足，它可以增长到maxStoredPermits。因此，通过调用acquisition(permits)请求的许可，将从以下方面参考提供。

 - stored permits (如果有的话）
 - fresh permits (剩余的许可证)

How this works is best explained with an example:

用一个例子来解释这一点是最好的:

 For a RateLimiter that produces 1 token per second, every second that goes by with the RateLimiter being unused, we increase storedPermits by 1. Say we leave the RateLimiter unused for 10 seconds (i.e., we expected a request at time X, but we are at time X + 10 seconds before a request actually arrives; this is also related to the point made in the last paragraph), thus storedPermits becomes 10.0 (assuming maxStoredPermits >= 10.0). At that point, a request of acquire(3) arrives. We serve this request out of storedPermits, and reduce that to 7.0 (how this is translated to throttling time is discussed later). Immediately after, assume that an acquire(10) request arriving. We serve the request partly from storedPermits, using all the remaining 7.0 permits, and the remaining 3.0, we serve them by fresh permits produced by the rate limiter.

对于一个每秒产生一个token的RateLimiter来说,每秒钟限速器没有被使用，我们就为storedPermits 加1。

假设我们让RateLimiter闲置了10秒（即我们期望在X时间有一个请求，但我们在X+10秒时才有一个请求实际到达；这也与上一段中的观点有关），因此storedPermits变成10.0（假设maxStoredPermits>=10.0）。在这一点上，一个acquire(3)的请求到达。我们从存储的许可中为这个请求提供服务，并将其减少到7.0（这如何转化为节流时间将在后面讨论）。紧接着，假设一个acquire(10)的请求到达。我们从存储的许可中为这个请求提供部分服务，使用所有剩余的7.0个许可，剩下的3.0个，我们用使用速率限制器的fresh permits产生的为它们提供。



 We already know how much time it takes to serve 3 fresh permits: if the rate is
"1 token per second", then this will take 3 seconds. But what does it mean to serve 7 stored permits? As explained above, there is no unique answer. If we are primarily interested to deal with underutilization, then we want stored permits to be given out /faster/ than fresh ones, because underutilization = free resources for the taking. If we are primarily interested to deal with overflow, then stored permits could be given out /slower/ than fresh ones. Thus, we require a (different in each case) function that translates storedPermits to throttling time.

 我们已经知道提供3个新的许可需要多少时间：如果速率是
"每秒1个标记"，那么这将需要3秒。但是，提供7个存储的许可证意味着什么呢？正如上面所解释的，没有唯一的答案。如果我们主要关注的是处理利用率不足的问题，那么我们希望stored permits （存储的许可）比fresh ones（新的许可）发放得更快，因为利用率不足=有可用资源可以用。如果我们主要关心的是处理溢出问题，那么储存的stored permits （存储的许可）比fresh ones（新的许可）发放得慢一些。因此，我们需要一个（在每种情况下都不同的）函数，将存储的许可进行节流时间

 This role is played by storedPermitsToWaitTime(double storedPermits, double permitsToTake). The underlying model is a continuous function mapping storedPermits (from 0.0 to maxStoredPermits) onto the 1/rate (i.e. intervals) that is effective at the given storedPermits. "storedPermits" essentially measure unused time; we spend unused time buying/storing permits. Rate is
"permits / time", thus "1 / rate = time / permits". Thus, "1/rate" (time / permits) times
"permits" gives time, i.e., integrals on this function (which is what storedPermitsToWaitTime() computes) correspond to minimum intervals between subsequent requests, for the specified number of requested permits.

 这个角色由 storedPermitsToWaitTime(double storedPermits, double permitsToTake) 扮演。底层模型是一个连续的函数，将storagePermits（从0.0到maxStoredPermits）映射到在给定storagePermits下有效的1/率（即间隔）。"storedPermits "本质上是衡量未使用的时间；我们用未使用的时间购买/储存许可证。速率是
"permits / time（许可证/时间）"，因此 "1 / rate = time / permits（1/率=时间/许可证）"。因此，"1/rate" (time / permits) "1/率"（时间/许可证）乘以
"permits许可证 "给出了时间，也就是说，这个函数上的积分（也就是storagePermitsToWaitTime()计算的内容）对应于后续请求的最小间隔，为指定数量的请求许可证。



 Here is an example of storedPermitsToWaitTime: If storedPermits == 10.0, and we want 3 permits, we take them from storedPermits, reducing them to 7.0, and compute the throttling for these as a call to storedPermitsToWaitTime(storedPermits = 10.0, permitsToTake = 3.0), which will evaluate the integral of the function from 7.0 to 10.0.

 下面是一个 storedPermitsToWaitTime 的例子：如果 storedPermits == 10.0，而我们想要 3 个许可证，我们从 storedPermits 中拿出来，把它们减少到 7.0，然后计算这些的节流，作为对 storedPermitsToWaitTime(storePermits = 10.0, permitsToTake = 3.0) 的调用，这将评估从 7.0 到 10.0 的函数积分。

 Using integrals guarantees that the effect of a single acquire(3) is equivalent to { acquire(1); acquire(1); acquire(1); }, or { acquire(2); acquire(1); }, etc, since the integral of the function in [7.0, 10.0] is equivalent to the sum of the integrals of [7.0, 8.0], [8.0, 9.0], [9.0, 10.0] (and so on), no matter what the function is. This guarantees that we handle correctly requests of varying weight (permits), /no matter/ what the actual function is - so we can tweak the latter freely. (The only requirement, obviously, is that we can compute its integrals).

使用积分保证了单个acquisition(3)的效果等同于{ acquire(1); acquire(1); acquire(1); }，或者{ acquire(2); acquire(1); }，等等，因为[7.0, 10.0]中函数的积分等同于[7.0, 8.0]、[8.0, 9.0]、[9.0, 10.0]（等等）的积分之和，不管这个函数是什么。这保证了我们能正确处理不同权重（许可）的请求，/不管/实际的函数是什么--所以我们可以自由调整后者。(显然，唯一的要求是我们能够计算其积分）。)

 Note well that if, for this function, we chose a horizontal line, at height of exactly (1/QPS), then the effect of the function is non-existent: we serve storedPermits at exactly the same cost as fresh ones (1/QPS is the cost for each). We use this trick later.



 请注意，如果对于这个函数，我们选择一条水平线，高度正好是（1/QPS），那么这个函数的效果就不存在了：我们以与新鲜许可完全相同的成本（1/QPS是每个许可的成本）提供存储许可。我们在后面会用到这个技巧。

 If we pick a function that goes /below/ that horizontal line, it means that we reduce the area of the function, thus time. Thus, the RateLimiter becomes /faster/ after a period of underutilization. If, on the other hand, we pick a function that goes /above/ that horizontal line, then it means that the area (time) is increased, thus storedPermits are more costly than fresh permits, thus the RateLimiter becomes /slower/ after a period of underutilization.

如果我们选取一个函数，去/低于/那条水平线，这意味着我们减少了函数的面积，从而减少了时间。因此，RateLimiter在利用不足一段时间后会变得/更快。另一方面，如果我们选择一个函数，去/高于/那条水平线，那么它意味着面积（时间）增加，因此，存储许可比新的许可更昂贵，因此，RateLimiter在利用不足一段时间后变得/更慢/。



 Last, but not least: consider a RateLimiter with rate of 1 permit per second, currently completely unused, and an expensive acquire(100) request comes. It would be nonsensical to just wait for 100 seconds, and /then/ start the actual task. Why wait without doing anything? A much better approach is to /allow/ the request right away (as if it was an acquire(1) request instead), and postpone /subsequent/ requests as needed. In this version, we allow starting the task immediately, and postpone by 100 seconds future requests, thus we allow for work to get done in the meantime instead of waiting idly.

 最后，但不是最不重要的：考虑一个速率为每秒1个许可的RateLimiter，目前完全没有使用，而一个昂贵的acquisition(100)请求到来。如果只是等待100秒，然后再开始实际的任务，那是毫无意义的。为什么什么都不做就等待？一个更好的方法是立即/允许/请求（就像它是一个acquisition(1)请求一样），并根据需要推迟/后续/请求。在这个版本中，我们允许立即启动任务，并将未来的请求推迟100秒，因此我们允许在此期间完成工作，而不是闲置等待。



This has important consequences: it means that the RateLimiter doesn't remember the time of the_last_ request, but it remembers the (expected) time of the _next_ request. This also enables us to tell immediately (see tryAcquire(timeout)) whether a particular timeout is enough to get us to the point of the next scheduling time, since we always maintain that. And what we mean by "an unused RateLimiter" is also defined by that notion: when we observe that the "expected arrival time of the next request" is actually in the past, then the difference (now - past) is the amount of time that the RateLimiter was formally unused, and it is that amount of time which we translate to storedPermits. (We increase storedPermits with the amount of permits that would have been produced in that idle time). So, if rate == 1 permit per second, and arrivals come exactly one second after the previous, then storedPermits is _never_ increased -- we would only increase it for arrivals _later_ than the expected one second.

RateLimiter不记得最后一次请求的时间，但它记得下一次请求的（预期）时间，这个产生了重大的设计影响。（见tryAcquire(timeout)）这总情况下也能够让我们立即告诉某个指定的超时时间是否能以让我们到达下一个调度时间点，这个时间我们总是坚持维护着这个时间。而我们所说的 "unused(未使用) RateLimiter "也是由这个概念产生的。unused RateLimiter指定是：当我们观察到 "下一个请求的预期到达时间 "实际上是在发生在过去，那么差值（现在-过去）就是RateLimiter的真实未使用的时间，我们正是通过这个unsed时间操作storagePermits数量，(即是我们用闲置时间产生的许可证数量来增加存储许可证)。因此，如果rate==每秒1个许可证，而到达的时间正好在前一个秒数之后，那么storagePermits就永远不会增加。--我们只会在到达时间比预期的一秒时间早的时候增加数量。

> unused RateLimiter的概念，和与storagePermit的关系。 说明的是，只在实际到达时间比预计到达时间更早的时候，增加storagePermits



This implements the following function where coldInterval = coldFactor * stableInterval.

这实现了下面的函数，其中coldInterval = coldFactor * stableInterval

```
 * <pre>
   *          ^ throttling
   *          |
   *    cold  +                  /
   * interval |                 /.
   *          |                / .
   *          |               /  .   ← "warmup period" 是  
   *          |              /   .     thresholdPermits and maxPermits 之间的四边形区域
   *          |             /    .
   *          |            /     .
   *          |           /      .
   *   stable +----------/  WARM .
   * interval |          .   UP  .
   *          |          . PERIOD.
   *          |          .       .
   *        0 +----------+-------+--------------→ storedPermits
   *          0 thresholdPermits maxPermits
   * </pre>
```

Before going into the details of this particular function, let's keep in mind the basics:

在深入这个特定函数的细节之前，让我们记住一些基本知识

<ol>
   <li>The state of the RateLimiter (storedPermits) is a vertical line in this figure.
 <li>When the RateLimiter is not used, this goes right (up to maxPermits)
   <li>When the RateLimiter is used, this goes left (down to zero), since if we have storedPermits, we serve from those first
<li>When _unused_, we go right at a constant rate! The rate at which we move to the right is  chosen as maxPermits / warmupPeriod. This ensures that the time it takes to go from 0 to maxPermits is equal to warmupPeriod.
 <li>When _used_, the time it takes, as explained in the introductory class note, is equal to the integral of our function, between X permits and X-K permits, assuming we want to spend K saved permits.
   </ol>

<ol> 
   <li>RateLimiter的状态（storedPermits）在这个图中是一条垂直线。
 <li>当RateLimiter没有被使用时，它向右移动（直到maxPermits）。
   <li>当RateLimiter被使用时，它向左走（向下到零），因为如果我们有storagePermits，我们首先从这些许可中服务。
<li>当_未使用时，我们以恒定的速度向右走 我们向右移动的速度被选为maxPermits / warmupPeriod。这确保了从0到maxPermits所需的时间等于warmupPeriod。
 <li>当_使用时，所需的时间，正如介绍性的课堂笔记所解释的，等于我们的函数的积分，在X许可和X-K许可之间，假设我们想花K保存许可。
   </ol>



In summary, the time it takes to move to the left (spend K permits), is equal to the area of the function of width == K.

综上所述，向左移动的时间（spend K permits），等于面积的 宽等于K的函数的面积。

Assuming we have saturated demand, the time to go from maxPermits to thresholdPermits is equal to warmupPeriod. And the time to go from thresholdPermits to 0 is warmupPeriod/2. (The reason that this is warmupPeriod/2 is to maintain the behavior of the original implementation where coldFactor was hard coded as 3.)

假设我们有饱和的需求，则maxPermits到thresholdPermits的时间就会等于warmupPeriod。而从 thresholdPermits 到 0 的时间是 warmupPeriod/2。（之所以是 warmupPeriod/2 是为了保持原始实现中的 coldFactor 被硬编码为 3）。)



It remains to calculate thresholdsPermits and maxPermits.
 <ul>
   <li>The time to go from thresholdPermits to 0 is equal to the integral of the function between 0 and thresholdPermits. This is thresholdPermits * stableIntervals. By (5) it is also equal to warmupPeriod/2. Therefore<blockquote>thresholdPermits = 0.5 * warmupPeriod / stableInterval</blockquote>
   <li>The time to go from maxPermits to thresholdPermits is equal to the integral of thefunction between thresholdPermits and maxPermits. This is the area of the picturedtrapezoid, and it is equal to 0.5 * (stableInterval + coldInterval) * (maxPermits -thresholdPermits). It is also equal to warmupPeriod, so<blockquote>maxPermits = thresholdPermits + 2 * warmupPeriod / (stableInterval + coldInterval)</blockquote>
 </ul>



剩下的就是计算thresholdsPermits和maxPermits。

 <ul>
   <li>从thresholdPermits到0的时间，等于0和thresholdPermits之间的函数的积分。这就是 thresholdPermits * stableIntervals。因此<blockquote>thresholdPermits = 0.5 * warmupPeriod / stableInterval</blockquote>。
   <li>从maxPermits到thresholdPermits的时间等于thresholdPermits和maxPermits之间函数的积分。这就是picturedtrapezoid的面积，它等于0.5 * (stableInterval + coldInterval) * (maxPermits -thresholdPermits)。它也等于 warmupPeriod，所以<blockquote>maxPermits = thresholdPermits + 2 * warmupPeriod / (stableInterval + coldInterval) </blockquote>
 </ul>
