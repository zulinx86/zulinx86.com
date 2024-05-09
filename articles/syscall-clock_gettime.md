---
title: clock_gettime() system call
emoji: "🕒"
type: "tech"
topics: ["linux", "clock"]
published: true
---

# クロックの種類

[clock_gettime(3) - Linux manual page](https://man7.org/linux/man-pages/man3/clock_gettime.3.html)

```c
       #include <time.h>

       int clock_gettime(clockid_t clockid, struct timespec *tp);
```

上記の通り、`clockid` 引数を通して、どのクロックから時間を取得するかを選ぶことができる。

全てはカバーしないが、以下のようなクロックの種類がある。
- `CLOCK_REALTIME`: Unix Epoch (1970-01-01 00:00:00 UT) から現在のシステム上の時刻までの経過時間を計測するクロック。管理者がシステムの時刻を変更することが可能であるため、それの影響を受ける。
- `CLOCK_MONOTONIC`: 起動してからシステムが稼働している間、単調に増え続けるクロック。`CLOCK_REALTIME` と違って、変更することができないため、処理時間を計測したりするのに使える。
- `CLOCK_BOOTTIME`: `CLOCK_MONOTONIC` とほぼ同様だが、システムが休止 (suspended) な時間もカウントするクロック。
- `CLOCK_PROCESS_CPUTIME_ID`: 呼び出したプロセスに使用された CPU 時間を計測するクロック。
- `CLOCK_THREAD_CPUTIME_ID`: 呼び出したスレッドに使用された CPU 時間を計測するクロック。


# データ構造
## `struct timespec`

[timespec(3type) - Linux manual page](https://man7.org/linux/man-pages/man3/timespec.3type.html)

```c
       #include <time.h>

       struct timespec {
           time_t     tv_sec;   /* Seconds */
           /* ... */  tv_nsec;  /* Nanoseconds [0, 999'999'999] */
       };
```

秒とナノ秒で構成され、コンピュータが扱いやすい形になっている。

## `struct tm`

[tm(3type) - Linux manual page](https://man7.org/linux/man-pages/man3/tm.3type.html)

```c
       #include <time.h>

       struct tm {
           int         tm_sec;    /* Seconds          [0, 60] */
           int         tm_min;    /* Minutes          [0, 59] */
           int         tm_hour;   /* Hour             [0, 23] */
           int         tm_mday;   /* Day of the month [1, 31] */
           int         tm_mon;    /* Month            [0, 11]  (January = 0) */
           int         tm_year;   /* Year minus 1900 */
           int         tm_wday;   /* Day of the week  [0, 6]   (Sunday = 0) */
           int         tm_yday;   /* Day of the year  [0, 365] (Jan/01 = 0) */
           int         tm_isdst;  /* Daylight savings flag */

           long        tm_gmtoff; /* Seconds East of UTC */
           const char *tm_zone;   /* Timezone abbreviation */
       };
```

より人間向けな日時の構造体となっている。


# サンプルコード
## `CLOCK_REALTIME`

`CLOCK_REALTIME` を使って時刻を取得し、それを `struct tm` に変換して表示する。

```c
#include <stdio.h>
#include <time.h>

int main() {
    struct timespec ts;
    struct tm *tm_info;
    char buffer[256];

    // Get the current CLOCK_REALTIME
    clock_gettime(CLOCK_REALTIME, &ts);

    // Convert to UTC time
    tm_info = gmtime(&ts.tv_sec);

    // Format time into a human-readable string
    strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", tm_info);
    printf("CLOCK_REALTIME (UTC):   %s\n", buffer);

    return 0;
}
```

実行結果
```
$ date; ./a.out
Mon Feb 26 15:27:49 UTC 2024
CLOCK_REALTIME (UTC):   2024-02-26 15:27:49
```

`date` コマンドで表示されるシステムの日時と一致している。


## `CLOCK_MONOTONIC`

`CLOCK_MONOTONIC` を使って、秒とナノ秒を表示する。

```c
#include <stdio.h>
#include <time.h>

int main() {
    struct timespec ts;

    // Get the current CLOCK_MONOTONIC time
    clock_gettime(CLOCK_MONOTONIC, &ts);

    // Print the result
    printf("CLOCK_MONOTONIC time: %ld seconds, %ld nanoseconds\n", ts.tv_sec, ts.tv_nsec);

    return 0;
}
```

実行結果
```
$ cat /proc/uptime; ./a.out
38587.06 4938551.11
CLOCK_MONOTONIC time: 38587 seconds, 63955741 nanoseconds
```

ちゃんとシステムの稼働時間 (`/proc/uptime`) と一致している。


## `CLOCK_PROCESS_CPUTIME_ID` / `CLOCK_THREAD_CPUTIME_ID`

CPU が使われた時間を、プロセス目線とスレッド目線で計測する。
CPU を意図的に使用するために、エラトステネスの篩を使って素数判定をする関数を使う。

処理手順は以下の通り。
1. 素数判定プログラムの実行
1. スレッドの作成
1. スレッド内で素数判定プログラムの実行
1. スレッド内で `CLOCK_THREAD_CPUTIME_ID` を使って時間計測
1. 親プロセスに戻ってきて `CLOCK_PROCESS_CPUTIME_ID` を使って時間計測

```c
#include <stdio.h>
#include <time.h>
#include <pthread.h>
#include <stdlib.h>

// Function to calculate prime numbers using the Sieve of Eratosthenes
void calculate_primes(int limit) {
    char* sieve = calloc(limit + 1, sizeof(char));
    if (!sieve) {
        printf("Memory allocation failed\n");
        return;
    }

    for (int i = 2; i * i <= limit; i++) {
        if (!sieve[i]) {
            for (int j = i * i; j <= limit; j += i) {
                sieve[j] = 1;
            }
        }
    }

    int prime_count = 0;
    for (int i = 2; i <= limit; i++) {
        if (!sieve[i]) {
            prime_count++;
        }
    }

    free(sieve);
    printf("Calculated primes up to %d, found %d primes\n", limit, prime_count);
}

void *thread_function(void *arg) {
    struct timespec thread_cpu_time;

    // Perform a CPU-intensive task: Calculate prime numbers
    calculate_primes(1e8);

    // Get the CPU time for this thread
    clock_gettime(CLOCK_THREAD_CPUTIME_ID, &thread_cpu_time);

    // Calculate and print the CPU time used by this thread
    double thread_time_used = thread_cpu_time.tv_sec + thread_cpu_time.tv_nsec / 1e9;
    printf("Thread CPU time used by thread: %f seconds\n", thread_time_used);

    return NULL;
}

int main() {
    pthread_t thread_id;
    struct timespec process_cpu_time;

    // Perform a CPU-intensive task: Calculate prime numbers
    calculate_primes(1e8);

    // Create a thread
    pthread_create(&thread_id, NULL, thread_function, NULL);
    // Wait for the thread to finish
    pthread_join(thread_id, NULL);

    // Get the CPU time for the process
    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &process_cpu_time);

    // Calculate and print the CPU time used by the process
    double process_time_used = process_cpu_time.tv_sec + process_cpu_time.tv_nsec / 1e9;
    printf("Process CPU time used by process: %f seconds\n", process_time_used);

    return 0;
}
```

実行結果
```
$ time ./a.out
Calculated primes up to 100000000, found 5761455 primes
Calculated primes up to 100000000, found 5761455 primes
Thread CPU time used by thread: 1.323552 seconds
Process CPU time used by process: 2.677082 seconds

real	0m2.678s
user	0m2.588s
sys	0m0.090s
```
