---
layout: post
title: BENCHMARK FILE OPEN PERFORMANCE
tags: benchmark, file, open
---

로그를 기입할 때마다 새로운 파일을 오픈하고 로그를 작성한 후에 파일을 닫는 방식으로 로그 모듈을 구현하려 합니다. I/O 중에서 OPEN 과 CLOSE 가 일어나기 때문에 파일을 열고 로그를 작성하는 것에 비하여 조금의 시간이 더 소요될 것이라 예상이 됩니다. 그래서 OPEN/CLOSE 를 수행하는 것과 그렇지 않고 OPEN 후 모든 로그를 작성한 후에 로그 파일을 다는 것에 대한 성능 측정을 진행하고 이해할 수 있는 범위 내에 존재한다고 하면 OPEN/CLOSE 를 수행하는 것으로 로그 라이브러리를 만들어 볼까 합니다. 아래의 코드는 로그 모듈로 사용할 출력 함수를 구현한 것입니다.

```c
#if 1
extern void xlogout(unsigned int type, const char * file, int line, const char * func, const char * format, ...)
{
    if(__mask & type)
    {
#if 1
        struct timespec start = { 0, };
        clock_gettime(CLOCK_REALTIME, &start);
#endif // 

        struct timespec spec = { 0, };
        struct tm m = { 0, };
        unsigned long threadid = pthread_self();

        clock_gettime(CLOCK_REALTIME, &spec);
        gmtime_r((time_t *) &spec.tv_sec, &m);

        char filepath[1024 + 64];
        snprintf(filepath, 1024 + 64, "%s/%04d-%02d-%02d-%s-%ld.log",
                                    __path == ((char *) 0) ? "" : __path,
                                    m.tm_year + 1900,
                                    m.tm_mon + 1,
                                    m.tm_mday,
                                    xlogtypeupperstr(type),
                                    threadid);


        FILE * fp = fopen(filepath, "a+");

        if(fp)
        {
            va_list ap;
            va_start(ap, format);
            fprintf(fp, "%04d-%02d-%02d %02d:%02d:%02d.%09ld [%s] %s:%d %s %lu ",
                        m.tm_year + 1900,
                        m.tm_mon + 1,
                        m.tm_mday,
                        m.tm_hour,
                        m.tm_min,
                        m.tm_sec,
                        spec.tv_nsec,
                        xlogtypeupperstr(type),
                        file,
                        line,
                        func,
                        threadid);
            fprintf(fp, format, ap);
            fprintf(fp, "\n");
            fclose(fp);
            va_end(ap);
        }
#if 1
        struct timespec end;
        clock_gettime(CLOCK_REALTIME, &end);
        struct timespec diff;
        diff.tv_sec = end.tv_sec - start.tv_sec;
        diff.tv_nsec = end.tv_nsec - start.tv_nsec;
        if(diff.tv_nsec < 0)
        {
            diff.tv_sec = diff.tv_sec - 1;
            diff.tv_nsec = 1000000000 + diff.tv_nsec;
        }
        printf("%ld.%09ld\t%ld.%09ld\t%ld.%09ld\n", start.tv_sec, start.tv_nsec, end.tv_sec, end.tv_nsec, diff.tv_sec, diff.tv_nsec);
#endif // 
    }
}
#else
FILE * fp = NULL;

extern void xlogout(unsigned int type, const char * file, int line, const char * func, const char * format, ...)
{
    if(fp == NULL)
    {
        char filepath[1024 + 64];
        snprintf(filepath, 1024 + 64, "%s/test.log",
                                    __path == ((char *) 0) ? "" : __path);
        fp = fopen(filepath, "a+");
    }
    if(__mask & type)
    {
#if 1
        struct timespec start = { 0, };
        clock_gettime(CLOCK_REALTIME, &start);
#endif // 

        struct timespec spec = { 0, };
        struct tm m = { 0, };
        unsigned long threadid = pthread_self();

        clock_gettime(CLOCK_REALTIME, &spec);
        gmtime_r((time_t *) &spec.tv_sec, &m);

        if(fp)
        {
            va_list ap;
            va_start(ap, format);
            fprintf(fp, "%04d-%02d-%02d %02d:%02d:%02d.%09ld [%s] %s:%d %s %lu ",
                        m.tm_year + 1900,
                        m.tm_mon + 1,
                        m.tm_mday,
                        m.tm_hour,
                        m.tm_min,
                        m.tm_sec,
                        spec.tv_nsec,
                        xlogtypeupperstr(type),
                        file,
                        line,
                        func,
                        threadid);
            fprintf(fp, format, ap);
            fprintf(fp, "\n");
            va_end(ap);
        }
#if 1
        struct timespec end;
        clock_gettime(CLOCK_REALTIME, &end);
        struct timespec diff;
        diff.tv_sec = end.tv_sec - start.tv_sec;
        diff.tv_nsec = end.tv_nsec - start.tv_nsec;
        if(diff.tv_nsec < 0)
        {
            diff.tv_sec = diff.tv_sec - 1;
            diff.tv_nsec = 1000000000 + diff.tv_nsec;
        }
        printf("%ld.%09ld\t%ld.%09ld\t%ld.%09ld\n", start.tv_sec, start.tv_nsec, end.tv_sec, end.tv_nsec, diff.tv_sec, diff.tv_nsec);
#endif // 
    }
}
#endif 
```

파일을 오픈한 상태에서 HELLO WORLD + 로그 포맷을 기입하는데 걸리는 시간은 평균 0.000001133 즉, 1 millisecond 정보다. 최대는 0.004892306 초이며 최저 소요 시간은 0.000000732 초이다. 로그를 작성할 때마다 파일을 열고 닫는 작업을 수행하는 로직의 평균 작업 시간은 0.000008332 이고, 최대 작업 소요 시간은 0.003489747, 최저 소유 시간은 0.000006153 이다. 파일을 오픈하고 닫는 로직을 구현하는데, 걸리는 시간은 작성하는 것과 비교하여 8배의 차이를 보인다. 그렇기 때문에, OPEN/CLOSE 로직을 수행하는 것을 로직이 포함하지 않는 것이 더 좋아 보인다. 대략 2배 미만의 소요 시간을 보였다면, OPEN/CLOSE 를 넣었을 것이지만 유니세컨드 단위라고 하여도 8배의 차이를 보이기 때문에, 파일을 열고 닫는 로직은 필요한 경우 한번으로 즉, OPEN/CLOSE 를 최소화시키는 것으로 구현할 필요가 잉어보인다.

| CATEGORY | ALWAYS OPEN/CLOSE | ONCE OPEN/CLOSE |
| -------- | ----------------- | --------------- |
| AVERAGE  | 0.000008332       | 0.000001133     |
| MAXIMUM  | 0.003489747       | 0.004892306     |
| MINIMUM  | 0.000006153       | 0.000000732     |

그래서 아래의 로직처럼 변경하여 THREAD 별 로그를 출력할 수 있도록 로직을 변경하였다.

```c
extern void xlogout(unsigned int type, const char * file, int line, const char * func, const char * format, ...)
{
    if(__mask & type)
    {
        struct timespec spec = { 0, };
        struct tm m = { 0, };
        unsigned long threadid = pthread_self();

        clock_gettime(CLOCK_REALTIME, &spec);
        gmtime_r((time_t *) &spec.tv_sec, &m);

        // GET THREAD SPECIFIC DATA
        if(key == (pthread_key_t *)(0))
        {
            key = calloc(sizeof(pthread_key_t), 1);
            if(pthread_key_create(key, xlogthreaddatarem) == 0)
            {
                xthreadlog * threadlog = calloc(sizeof(xthreadlog), 1);
                if(pthread_setspecific(*key, threadlog) != 0)
                {
                    pthread_key_delete(*key);
                    free(key);
                    key = (pthread_key_t *)(0);
                }
            }
            else
            {
                free(key);
                key = (pthread_key_t *)(0);
            }
        }

        if(key)
        {
            xthreadlog * threadlog = (xthreadlog *) pthread_getspecific(*key);
            if(threadlog)
            {
                int date = (m.tm_year + 1900) * 10000 + (m.tm_mon +1) * 100 + m.tm_mday;
                if(threadlog->fp == (FILE *)(0) || threadlog->date != date)
                {
                    if(threadlog->fp)
                    {
                        fclose(threadlog->fp);
                    }

                    char path[1024 + 64];
                    snprintf(path, 1024 + 64, "%s/%04d-%02d-%02d-%lu.log",
                                              __path ? __path : "",
                                              m.tm_year + 1900,
                                              m.tm_mon + 1,
                                              m.tm_mday,
                                              threadid);
                    threadlog->fp = fopen(path, "a+");
                    threadlog->date = date;
                }

                if(threadlog->fp)
                {
                    va_list ap;
                    va_start(ap, format);
                    fprintf(threadlog->fp, "%04d-%02d-%02d %02d:%02d:%02d.%09ld [%s] %s:%d %s %lu ",
                                           m.tm_year + 1900,
                                           m.tm_mon + 1,
                                           m.tm_mday,
                                           m.tm_hour,
                                           m.tm_min,
                                           m.tm_sec,
                                           spec.tv_nsec,
                                           xlogtypeupperstr(type),
                                           file,
                                           line,
                                           func,
                                           threadid);

                    fprintf(threadlog->fp, format, ap);
                    fprintf(threadlog->fp, "\n");
                    va_end(ap);
                }
            }
        }
    }
}
```

| CATEGORY | ALWAYS OPEN/CLOSE | ONCE OPEN/CLOSE | THREAD LOG ONCE OPEN/CLOSE |
| -------- | ----------------- | --------------- | -------------------------- |
| AVERAGE  | 0.000008332       | 0.000001133     | 0.000001088                |
| MAXIMUM  | 0.003489747       | 0.004892306     | 0.000337471                |
| MINIMUM  | 0.000006153       | 0.000000732     | 0.000000741                |

스레드별 ONCE OPEN/CLOSE 로그는 약 1 unisecond 의 평균 수행 시간을 나타내고 있다.
최대 소요 시간은 0.000337471 이고, 최소 수행 시간은 0.000000741 이다.

----


결론적으로 OPEN/CLOSE 를 출력할 때마다 수행하는 것은 약 8배의 수행 시간을 더 소요시킨다.
그렇기 때문에, 로그를 파일로 출력하는 것은 OPEN/CLOSE 를 덜 수행시키도록 로직을 변경하는 것이 좋다.
