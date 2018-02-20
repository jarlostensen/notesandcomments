### A word about std::chrono::high_resolution_clock on Windows
In VisualStudio `std::chrono::high_resolution_clock` has since version 2015 been implemented using QCP [bug 719443](https://web.archive.org/web/20141212192132/https://connect.microsoft.com/VisualStudio/feedback/details/719443/). See also [Acquiring high-resolution time stamps](https://msdn.microsoft.com/en-us/library/windows/desktop/dn553408(v=vs.85).aspx).
Some of the key benefits of using `std::chrono` are obviously portatbility across platforms, and not having to write and maintain the code. 
However, there are some issues with using `std::chrono` for accurate timing that we need to be aware of.

Firstly, it obviously has some overhead above and beyond direct use of QCP, and this overhead is dependent on how well your compiler has optimised the code since it is brought in as a template header, i.e. not as a binary. A debug build and a release build will therefore yield significantly different timing results if you use `std::chrono`, as opposed to using QCP directly. 

Secondly it reports its accuracy in a way which is, strictly speaking, incorrect. Whereas QCP's accuracy is variable according to your HW, `std::chrono` always reports nanosecond accuracy, i.e. `std::chrono::high_resolution_clock::period`. However, since it's implemented using QCP (on Windows) it obviously can't be more accurate than it is, and a nanosecond resolution is therefore incorrect. Arguably it should return the same value for its resolution as the underlying implementation, but that's a debate for the standards committee.

Let's dive straight in and take measurements of the shortest interval (i.e. accuracy) of both `std::chrono` and QCP, to see what these two things mean in practice;

``` c++
    void ShortestIntervalMeasurementsQCPandChrono()
    {
        // ===================================
        // std::chrono::high_resolution_clock

        using clock = std::chrono::high_resolution_clock;
        using unit = std::chrono::nanoseconds;

        intmax_t chrono_samples[1000];
        size_t chrono_sample_count = 0;

        // just loop fast, taking measurements of the loop time 
        // We could unroll this as well, for even higher speed, but as the measurements will show this has little practical effect
        auto c_prev = clock::now();
        for (unsigned i = 0; i < 1000; ++i)
        {
            auto c_now = clock::now();
            chrono_samples[chrono_sample_count++] = intmax_t(std::chrono::duration_cast<unit>(c_now - c_prev).count());
            c_prev = c_now;
        }
   
        // ===================================
        // QCP 

        intmax_t qcp_samples[1000];
        size_t qcp_sample_count = 0;
        LARGE_INTEGER qcp_prev;
        LARGE_INTEGER qcp_now;
        QueryPerformanceCounter(&qcp_prev);
        
        for (unsigned i = 0; i < 1000; ++i)
        {
            QueryPerformanceCounter(&qcp_now);
            qcp_samples[qcp_sample_count++] = intmax_t(qcp_now.QuadPart - qcp_prev.QuadPart);
            qcp_prev = qcp_now;
        }

        auto chrono_mean = mean(chrono_samples,chrono_sample_count);
        auto chrono_stddev = stddev(chrono_samples, chrono_sample_count);
        auto qcp_mean = mean(qcp_samples,qcp_sample_count);
        auto qcp_stddev = stddev(qcp_samples, qcp_sample_count);

        LARGE_INTEGER perf_freq;
        QueryPerformanceFrequency(&perf_freq);
        // QCP returns values in ticks, and each tick is 1 over the "performance frequency" of the system
        // this corresponds to the smallest measurable interval, in nanoseconds, for QCP
        // on a particular 4.2 GHz i5 used for this test this value was 30.9291 nanoseconds
        auto ticksToNs = 1000000000.0/double(perf_freq);

        std::cout << "\n\tmean; qcp: " << (qcp_mean*ticksToNs) << ", chrono: " << chrono_mean << "\n";
        std::cout << "\tstddev; qcp: " << (qcp_stddev*ticksToNs) << ", chrono: " << chrono_stddev << "\n";
    }
```

A run of the above on an i5-7500K @ 4.2 GHz yields the following results

```
Debug build
mean; qcp: 30.9291, chrono: 436.718
stddev; qcp: 0, chrono: 343.791
```

and

```
Release build
mean; qcp: 30.9291, chrono: 309.285
stddev; qcp: 0, chrono: 0.452857
```

!!! note 
    runs on other hardware give the same results for QCP, and the same degree of variation for `std::chrono`

Notice in particular the large discrepancy between Debug and Release builds for `std::chrono`, a discrepancy which is practically absent when using QCP.
Also note the exact correspondence between the smallest measurable interval for QCP, `ticksToNs`, and the measurements we get in the loop. The optimised (and debug) versions of the loop both execute so fast that QCP is unable to resolve it and consequently always returns a value of 1 tick.
Whereas QCP is steady, `std::chrono` shows more variance and has a much higher mean for the smallest measurable interval, therefore 

**`std::chrono` is not suitable for very high resolution, stable, time measurement, for these scenarios you should always use QCP or a plaform equivalent.**
