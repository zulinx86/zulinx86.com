---
title: 【Firecracker】Rate Limiter
emoji: "🧨"
type: "tech"
topics: ["virtualization"]
published: false
---


### `struct RateLimiter`

The rate limit is based on the token bucket algorithm.

[https://en.wikipedia.org/wiki/Token_bucket](https://en.wikipedia.org/wiki/Token_bucket)

> The token bucket algorithm is based on an analogy of a fixed capacity bucket into which tokens, ..., are added at a fixed rate.

Token replenishment is implemented using timerfd.

[https://man.freebsd.org/cgi/man.cgi?query=timerfd&sektion=2&format=html](https://man.freebsd.org/cgi/man.cgi?query=timerfd&sektion=2&format=html)

> DESCRIPTION
>        The  timerfd  system  calls  operate  on	 timers, identified by special
>        timerfd file descriptors.  These	calls are analogous to timer_create(),
>        timer_gettime(),	and timer_settime() per-process	timer  functions,  but
>        use a timerfd descriptor	in place of timerid.
> 
>        All  timerfd descriptors	possess	traditional file descriptor semantics;
>        they may	be passed to other processes, preserved	 across	 fork(2),  and
>        monitored  via  kevent(2),  poll(2),  or	select(2).  When a timerfd de-
>        scriptor	is no longer needed, it	may be disposed	of using close(2).

The token buckets are implemented as `struct TokenBucket`.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L300](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L300)
```rs
/// Rate Limiter that works on both bandwidth and ops/s limiting.
///
/// Bandwidth (bytes/s) and ops/s limiting can be used at the same time or individually.
///
/// Implementation uses a single timer through TimerFd to refresh either or
/// both token buckets.
///
/// Its internal buckets are 'passively' replenished as they're being used (as
/// part of `consume()` operations).
/// A timer is enabled and used to 'actively' replenish the token buckets when
/// limiting is in effect and `consume()` operations are disabled.
///
/// RateLimiters will generate events on the FDs provided by their `AsRawFd` trait
/// implementation. These events are meant to be consumed by the user of this struct.
/// On each such event, the user must call the `event_handler()` method.
pub struct RateLimiter {
    bandwidth: Option<TokenBucket>,
    ops: Option<TokenBucket>,

    timer_fd: TimerFd,
    // Internal flag that quickly determines timer state.
    timer_active: bool,
}
```


#### `RateLimiter::new()`

It constructs two token buckets: one for bandwidth and the other for ops/s.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L347](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L347s)
```rs
impl RateLimiter {
    /// Creates a new Rate Limiter that can limit on both bytes/s and ops/s.
    ///
    /// # Arguments
    ///
    /// * `bytes_total_capacity` - the total capacity of the `TokenType::Bytes` token bucket.
    /// * `bytes_one_time_burst` - initial extra credit on top of `bytes_total_capacity`,
    /// that does not replenish and which can be used for an initial burst of data.
    /// * `bytes_complete_refill_time_ms` - number of milliseconds for the `TokenType::Bytes`
    /// token bucket to go from zero Bytes to `bytes_total_capacity` Bytes.
    /// * `ops_total_capacity` - the total capacity of the `TokenType::Ops` token bucket.
    /// * `ops_one_time_burst` - initial extra credit on top of `ops_total_capacity`,
    /// that does not replenish and which can be used for an initial burst of data.
    /// * `ops_complete_refill_time_ms` - number of milliseconds for the `TokenType::Ops` token
    /// bucket to go from zero Ops to `ops_total_capacity` Ops.
    ///
    /// If either bytes/ops *size* or *refill_time* are **zero**, the limiter
    /// is **disabled** for that respective token type.
    ///
    /// # Errors
    ///
    /// If the timerfd creation fails, an error is returned.
    pub fn new(
        bytes_total_capacity: u64,
        bytes_one_time_burst: u64,
        bytes_complete_refill_time_ms: u64,
        ops_total_capacity: u64,
        ops_one_time_burst: u64,
        ops_complete_refill_time_ms: u64,
    ) -> io::Result<Self> {
        let bytes_token_bucket = TokenBucket::new(
            bytes_total_capacity,
            bytes_one_time_burst,
            bytes_complete_refill_time_ms,
        );

        let ops_token_bucket = TokenBucket::new(
            ops_total_capacity,
            ops_one_time_burst,
            ops_complete_refill_time_ms,
        );

        // We'll need a timer_fd, even if our current config effectively disables rate limiting,
        // because `Self::update_buckets()` might re-enable it later, and we might be
        // seccomp-blocked from creating the timer_fd at that time.
        let timer_fd = TimerFd::new_custom(ClockId::Monotonic, true, true)?;

        Ok(RateLimiter {
            bandwidth: bytes_token_bucket,
            ops: ops_token_bucket,
            timer_fd,
            timer_active: false,
        })
    }
// snipped
}
```

#### `RateLimiter::consume()`

`timer_active` was initialized with `false` in `RateLimiter::new()`.

Only if `timer_active == false`, it can consume tokens.

But when is `timer_active` set to `true`?

If it failed to reduce tokens from the bucket or overly consume tokens, `timer_active` is set to `true` and it registers a timer to replenish the bucket.

`timer_active` is set back to `false` in `RateLimiter::event_handler()` that is fired when the timer expires.

Thus, `timer_active` exists to avoid unnecessary process when the bucket is empty.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L390](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L390)

```rs
    /// Attempts to consume tokens and returns whether that is possible.
    ///
    /// If rate limiting is disabled on provided `token_type`, this function will always succeed.
    pub fn consume(&mut self, tokens: u64, token_type: TokenType) -> bool {
        // If the timer is active, we can't consume tokens from any bucket and the function fails.
        if self.timer_active {
            return false;
        }

        // Identify the required token bucket.
        let token_bucket = match token_type {
            TokenType::Bytes => self.bandwidth.as_mut(),
            TokenType::Ops => self.ops.as_mut(),
        };
        // Try to consume from the token bucket.
        if let Some(bucket) = token_bucket {
            let refill_time = bucket.refill_time_ms();
            match bucket.reduce(tokens) {
                // When we report budget is over, there will be no further calls here,
                // register a timer to replenish the bucket and resume processing;
                // make sure there is only one running timer for this limiter.
                BucketReduction::Failure => {
                    if !self.timer_active {
                        self.activate_timer(TIMER_REFILL_STATE);
                    }
                    false
                }
                // The operation succeeded and further calls can be made.
                BucketReduction::Success => true,
                // The operation succeeded as the tokens have been consumed
                // but the timer still needs to be armed.
                BucketReduction::OverConsumption(ratio) => {
                    // The operation "borrowed" a number of tokens `ratio` times
                    // greater than the size of the bucket, and since it takes
                    // `refill_time` milliseconds to fill an empty bucket, in
                    // order to enforce the bandwidth limit we need to prevent
                    // further calls to the rate limiter for
                    // `ratio * refill_time` milliseconds.
                    // The conversion should be safe because the ratio is positive.
                    #[allow(clippy::cast_sign_loss, clippy::cast_possible_truncation)]
                    self.activate_timer(TimerState::Oneshot(Duration::from_millis(
                        (ratio * refill_time as f64) as u64,
                    )));
                    true
                }
            }
        } else {
            // If bucket is not present rate limiting is disabled on token type,
            // consume() will always succeed.
            true
        }
    }
```

#### `RateLimiter::activate_timer()`

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L381](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L381)
```rs
    // Arm the timer of the rate limiter with the provided `TimerState`.
    fn activate_timer(&mut self, timer_state: TimerState) {
        // Register the timer; don't care about its previous state
        self.timer_fd.set_state(timer_state, SetTimeFlags::Default);
        self.timer_active = true;
    }
```

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L21](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L21)
```rs
const TIMER_REFILL_STATE: TimerState =
    TimerState::Oneshot(Duration::from_millis(REFILL_TIMER_INTERVAL_MS));
```

#### `RateLimiter::event_handler()`

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L471](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L471)
```rs
    /// This function needs to be called every time there is an event on the
    /// FD provided by this object's `AsRawFd` trait implementation.
    ///
    /// # Errors
    ///
    /// If the rate limiter is disabled or is not blocked, an error is returned.
    pub fn event_handler(&mut self) -> Result<(), RateLimiterError> {
        match self.timer_fd.read() {
            0 => Err(RateLimiterError::SpuriousRateLimiterEvent(
                "Rate limiter event handler called without a present timer",
            )),
            _ => {
                self.timer_active = false;
                Ok(())
            }
        }
    }
```

#### `RateLimiter::as_raw_fd()`

`RateLimiter` implements `AsRawFd` and returns the raw file descriptor of the internal timerfd so that users can monitor POLLIN events.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L509-L519](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L509-L519)
```c
impl AsRawFd for RateLimiter {
    /// Provides a FD which needs to be monitored for POLLIN events.
    ///
    /// This object's `event_handler()` method must be called on such events.
    ///
    /// Will return a negative value if rate limiting is disabled on both
    /// token types.
    fn as_raw_fd(&self) -> RawFd {
        self.timer_fd.as_raw_fd()
    }
}
```

[https://man7.org/linux/man-pages/man2/timerfd_create.2.html](https://man7.org/linux/man-pages/man2/timerfd_create.2.html)

>    Operating on a timer file descriptor
>        The file descriptor returned by timerfd_create() supports the
>        following additional operations:
> 
> ...
> 
>        poll(2)
>        select(2)
>        (and similar)
>               The file descriptor is readable (the select(2) readfds
>               argument; the poll(2) POLLIN flag) if one or more timer
>               expirations have occurred.
> 
>               The file descriptor also supports the other file-
>               descriptor multiplexing APIs: pselect(2), ppoll(2), and
>               epoll(7).


### `struct timerfd::TimerFd`

#### `TimerFd::new_custom()`

The timerfd was initialized with `TimerFd::new_custom(ClockId::Monotonic, true, true)`.

It uses the monotonic clock, non-blocking mode is enabled and the fd is closed on exec.

[https://docs.rs/timerfd/latest/timerfd/struct.TimerFd.html](https://docs.rs/timerfd/latest/timerfd/struct.TimerFd.html)

> ```rs
> pub fn new_custom(
>     clock: ClockId,
>     nonblocking: bool,
>     cloexec: bool
> ) -> IoResult<TimerFd>
> ```
> 
> Creates a new TimerFd.
> 
> By default, it uses the monotonic clock, is blocking and does not close on exec. The parameters allow you to change that.
> 
> Errors
>
> On Linux 2.6.26 and earlier, nonblocking and cloexec are not supported and setting them will return an error of kind ErrorKind::InvalidInput.
> 
> This can also fail in various cases of resource exhaustion. Please check timerfd_create(2) for details.

#### `TimerFd::set_state()`

[https://docs.rs/timerfd/latest/timerfd/struct.TimerFd.html](https://docs.rs/timerfd/latest/timerfd/struct.TimerFd.html)

> ```rs
> pub fn set_state(
>     &mut self,
>     state: TimerState,
>     sflags: SetTimeFlags
> ) -> TimerState
> ```
> 
> Sets this timerfd to a given TimerState and returns the old state.



### `enum timerfd::SetTimeFlags`

[https://docs.rs/timerfd/latest/timerfd/enum.SetTimeFlags.html](https://docs.rs/timerfd/latest/timerfd/enum.SetTimeFlags.html)

> ```rs
> pub enum SetTimeFlags {
>     Default,
>     Abstime,
>     TimerCancelOnSet,
> }
> ```

> Default
>   Flags to `timerfd_settime(2)`.
>   The default is zero, i. e. all bits unset.


### `struct TokenBucket`

`TokenBucket` provides the lower level interface for the rate limiter.

It has one-time initial burst, which means the bucket has its size + one-time burst tokens.

`refill_time` is how much time required to completely fill up the bucket.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L58](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L58)
```rs
/// TokenBucket provides a lower level interface to rate limiting with a
/// configurable capacity, refill-rate and initial burst.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct TokenBucket {
    // Bucket defining traits.
    size: u64,
    // Initial burst size.
    initial_one_time_burst: u64,
    // Complete refill time in milliseconds.
    refill_time: u64,

    // Internal state descriptors.

    // Number of free initial tokens, that can be consumed at no cost.
    one_time_burst: u64,
    // Current token budget.
    budget: u64,
    // Last time this token bucket saw activity.
    last_update: Instant,

    // Fields used for pre-processing optimizations.
    processed_capacity: u64,
    processed_refill_time: u64,
}
```

#### `TokenBucket::new()`

The refill rate (tokens per nanosecond) is calculated as (elapsed time since the last update) * ((bucket size) / (refill time)).

To make this calculation easier, it does some calculation ahead of time. Specifically, since it will compute (bucket size) / (refill time), it devides them by their greatest common divisor (GCD).

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L89](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L89)
```rs
impl TokenBucket {
    /// Creates a `TokenBucket` wrapped in an `Option`.
    ///
    /// TokenBucket created is of `size` total capacity and takes `complete_refill_time_ms`
    /// milliseconds to go from zero tokens to total capacity. The `one_time_burst` is initial
    /// extra credit on top of total capacity, that does not replenish and which can be used
    /// for an initial burst of data.
    ///
    /// If the `size` or the `complete refill time` are zero, then `None` is returned.
    pub fn new(size: u64, one_time_burst: u64, complete_refill_time_ms: u64) -> Option<Self> {
        // If either token bucket capacity or refill time is 0, disable limiting.
        if size == 0 || complete_refill_time_ms == 0 {
            return None;
        }
        // Formula for computing current refill amount:
        // refill_token_count = (delta_time * size) / (complete_refill_time_ms * 1_000_000)
        // In order to avoid overflows, simplify the fractions by computing greatest common divisor.

        let complete_refill_time_ns =
            complete_refill_time_ms.checked_mul(NANOSEC_IN_ONE_MILLISEC)?;
        // Get the greatest common factor between `size` and `complete_refill_time_ns`.
        let common_factor = gcd(size, complete_refill_time_ns);
        // The division will be exact since `common_factor` is a factor of `size`.
        let processed_capacity: u64 = size / common_factor;
        // The division will be exact since `common_factor` is a factor of
        // `complete_refill_time_ns`.
        let processed_refill_time: u64 = complete_refill_time_ns / common_factor;

        Some(TokenBucket {
            size,
            one_time_burst,
            initial_one_time_burst: one_time_burst,
            refill_time: complete_refill_time_ms,
            // Start off full.
            budget: size,
            // Last updated is now.
            last_update: Instant::now(),
            processed_capacity,
            processed_refill_time,
        })
    }
// snipped
}
```

#### `TokenBucket::reduce()`

If the initial one time burst is still available, consumes it.

Only if the consumed tokens is greater than the budget, it lazily replenishes the bucket. This laziness reduces performance cost.

If the number of tokens is greater than the bucket size, it consumes all the remaining budget and returns `BucketReduction::OverConsumption` that holds the rest of tokens.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L177](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L177)
```rs
    /// Attempts to consume `tokens` from the bucket and returns whether the action succeeded.
    pub fn reduce(&mut self, mut tokens: u64) -> BucketReduction {
        // First things first: consume the one-time-burst budget.
        if self.one_time_burst > 0 {
            // We still have burst budget for *all* tokens requests.
            if self.one_time_burst >= tokens {
                self.one_time_burst -= tokens;
                self.last_update = Instant::now();
                // No need to continue to the refill process, we still have burst budget to consume
                // from.
                return BucketReduction::Success;
            } else {
                // We still have burst budget for *some* of the tokens requests.
                // The tokens left unfulfilled will be consumed from current `self.budget`.
                tokens -= self.one_time_burst;
                self.one_time_burst = 0;
            }
        }

        if tokens > self.budget {
            // Hit the bucket bottom, let's auto-replenish and try again.
            self.auto_replenish();

            // This operation requests a bandwidth higher than the bucket size
            if tokens > self.size {
                crate::logger::error!(
                    "Consumed {} tokens from bucket of size {}",
                    tokens,
                    self.size
                );
                // Empty the bucket and report an overconsumption of
                // (remaining tokens / size) times larger than the bucket size
                tokens -= self.budget;
                self.budget = 0;
                return BucketReduction::OverConsumption(tokens as f64 / self.size as f64);
            }

            if tokens > self.budget {
                // Still not enough tokens, consume() fails, return false.
                return BucketReduction::Failure;
            }
        }

        self.budget -= tokens;
        BucketReduction::Success
    }
```

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L46](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L46)
```rs
/// Enum describing the outcomes of a `reduce()` call on a `TokenBucket`.
#[derive(Clone, Debug, PartialEq)]
pub enum BucketReduction {
    /// There are not enough tokens to complete the operation.
    Failure,
    /// A part of the available tokens have been consumed.
    Success,
    /// A number of tokens `inner` times larger than the bucket size have been consumed.
    OverConsumption(f64),
}
```

#### `TokenBucket::auto_replenish()`

First, it computes the elapsed time since the last refill in nanoseconds.

If the elapsed time is greater than or equal to the refill time, it fills the bucket up fully.
Else it does the calculation of (elapsed time) * (refill rate) using the preprocessed values.

[https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L124](https://github.com/firecracker-microvm/firecracker/blob/cbe79eaa516b201aa9158ee99f03e0e72c2a23ac/src/vmm/src/rate_limiter/mod.rs#L124)
```rs
    // Replenishes token bucket based on elapsed time. Should only be called internally by `Self`.
    #[allow(clippy::cast_possible_truncation)]
    fn auto_replenish(&mut self) {
        // Compute time passed since last refill/update.
        let now = Instant::now();
        let time_delta = (now - self.last_update).as_nanos();

        if time_delta >= u128::from(self.refill_time * NANOSEC_IN_ONE_MILLISEC) {
            self.budget = self.size;
            self.last_update = now;
        } else {
            // At each 'time_delta' nanoseconds the bucket should refill with:
            // refill_amount = (time_delta * size) / (complete_refill_time_ms * 1_000_000)
            // `processed_capacity` and `processed_refill_time` are the result of simplifying above
            // fraction formula with their greatest-common-factor.

            // In the constructor, we assured that (self.refill_time * NANOSEC_IN_ONE_MILLISEC)
            // fits into a u64 That means, at this point we know that time_delta <
            // u64::MAX. Since all other values here are u64, this assures that u128
            // multiplication cannot overflow.
            let processed_capacity = u128::from(self.processed_capacity);
            let processed_refill_time = u128::from(self.processed_refill_time);

            let tokens = (time_delta * processed_capacity) / processed_refill_time;

            // We increment `self.last_update` by the minimum time required to generate `tokens`, in
            // the case where we have the time to generate `1.8` tokens but only
            // generate `x` tokens due to integer arithmetic this will carry the time
            // required to generate 0.8th of a token over to the next call, such that if
            // the next call where to generate `2.3` tokens it would instead
            // generate `3.1` tokens. This minimizes dropping tokens at high frequencies.
            // We want the integer division here to round up instead of down (as if we round down,
            // we would allow some fraction of a nano second to be used twice, allowing
            // for the generation of one extra token in extreme circumstances).
            let mut time_adjustment = tokens * processed_refill_time / processed_capacity;
            if tokens * processed_refill_time % processed_capacity != 0 {
                time_adjustment += 1;
            }

            // Ensure that we always generate as many tokens as we can: assert that the "unused"
            // part of time_delta is less than the time it would take to generate a
            // single token (= processed_refill_time / processed_capacity)
            debug_assert!(time_adjustment <= time_delta);
            debug_assert!(
                (time_delta - time_adjustment) * processed_capacity <= processed_refill_time
            );

            // time_adjustment is at most time_delta, and since time_delta <= u64::MAX, this cast is
            // fine
            self.last_update += Duration::from_nanos(time_adjustment as u64);
            self.budget = std::cmp::min(self.budget.saturating_add(tokens as u64), self.size);
        }
    }
```
