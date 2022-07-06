# The Gregorian Calendar, and DST

## Introduction

The [Gregorian Calendar](https://en.wikipedia.org/wiki/Gregorian_calendar) is the calendar most widely used in the world today, and so conversions regarding that calendar are a common task for computers.

[Daylight savings time](https://en.wikipedia.org/wiki/Daylight_saving_time) ("DST") is another time-related phenomenon that computers have to deal with on a regular basis.

The algorithms for dealing with the calendar and DST are often cumbersome, involving look-up tables and inefficient case-by-case logic. I present below algorithms which do not require look-up tables, and which I believe are as efficient as possible. These algorithms take advantage of integer arithmetic present in most modern computer languages.

## Acknowledgement

The algorithms for converting between day number and date are based on algorithms presented by Jean Meeus in his 1991 book *Astronomical Algorithms*.

## Converting from y/m/d into days since 1970/Jan/01

To make our lives easier when it comes to leap years and 29<sup>th</sup> February, we define λ, ε such that

- *λ= y-1* for January, February; *λ = y* for March to December

- *ε = 2 to 11* for March to December; *ε = 12, 13* for January, February

This moves the leap day to the end of the "year" *λ-1*, which means that if the "year" *λ* is a leap year it should start one day later than it otherwise would. And, the definition of *ε* helps us with the varying lengths of the months, as if we examine the quantity `30*ε + ((3*ε+6)/5)` (where the division is strictly integer division), we get

| Month     | ε   | `30*ε + ((3*ε+6)/5)` | Difference to previous |
| --------- | --- | -------------------- | ---------------------- |
| March     | 2   | 62                   | n/a                    |
| April     | 3   | 93                   | 31                     |
| May       | 4   | 123                  | 30                     |
| June      | 5   | 154                  | 31                     |
| July      | 6   | 184                  | 30                     |
| August    | 7   | 215                  | 31                     |
| September | 8   | 246                  | 31                     |
| October   | 9   | 276                  | 30                     |
| November  | 10  | 307                  | 31                     |
| December  | 11  | 337                  | 30                     |
| January   | 12  | 368                  | 31                     |
| February  | 13  | 399                  | 31                     |

Hence we see that `30*ε+((3*ε+6)/5)+d-63` is the number of days that have elapsed since the start of the "year" *λ*.

So, the number of days that have elapsed can be calculated with

1. `365 * λ` for the basic length of the year

2. add `λ/4` for leap years

3. subtract `λ/100` for non-leap century years

4. add `λ/400` for the 400-year leap day rule

5. add `30*ε+((3*ε+6)/5)+d`

6. subtract a constant so that 1970/Jan/01 is day zero

In C, we have

```c
// Convert y,m,d into a number of days since 1970, where 0<=m<=11
// https://github.com/deirdreobyrne/CalendarAndDST
int getDayNumberFromDate(int y, int m, int d) {
  int ans;

  if (m < 2) {
    y--;
    m+=12;
  }
  ans = (y/100);
  return 365*y + (y>>2) - ans + (ans>>2) + 30*m + ((3*m+6)/5) + d - 719531;
}
```

## Converting days since 1970/Jan/01 to y/m/d

For this conversion, we use a slightly different definition of ε, namely ε is 4 to 13 for March to December, and 14 and 15 for January and February. This gives us

| Month     | ε   | `d=30*ε+((3*ε)/5)+1` | Diff to prev. | `(5*d-1)/153`  |
| --------- | --- | -------------------- | ------------- | -------------- |
| March     | 4   | 123                  | n/a           | 4 ( + 2/153 )  |
| April     | 5   | 154                  | 31            | 5 ( + 4/153 )  |
| May       | 6   | 184                  | 30            | 6 ( + 1/153 )  |
| June      | 7   | 215                  | 31            | 7 ( + 3/153 )  |
| July      | 8   | 245                  | 30            | 8              |
| August    | 9   | 276                  | 31            | 9 ( + 2/153 )  |
| September | 10  | 307                  | 31            | 10 ( + 4/153 ) |
| October   | 11  | 337                  | 30            | 11 ( + 1/153 ) |
| November  | 12  | 368                  | 31            | 12 ( + 3/153 ) |
| December  | 13  | 398                  | 30            | 13             |
| January   | 14  | 429                  | 31            | 14 ( + 2/153 ) |
| February  | 15  | 460                  | 31            | 15 ( + 4/153 ) |

In other words, if we take March 1<sup>st</sup> as day number *d=123*, then if we use integer division, `(5*d-1)/153` will give us ε.

The first thing we need to do is to move our day zero to a useful date in the past when the 400-year cycle of the calendar started. We choose 1600/Feb/29 - the last leap day before the start of a run of 3 non-leap century years. This date corresponds to a day number of -135081. Hence we set `a=day+135081` as our starting point.

We first need to correct for the 400-year leap days. We define the quantity `b=a/146097` (using integer division, as always). Hence the quantity *b* will be zero from 1600/Feb/29 to 2000/Feb/28, and will become 1 on 2000/Feb/29. Hence if we take `c=a-b`, we will remove the extra leap day that is inserted every 400 years. We will then be left with a quantity *c* that increases by 36524 days every 100 years. So the next task is to find out how many 100-year leap days have been removed.

The answer is, of course, `(c-1)/36524`. We need to subtract 1 from *c* to account for the fact that 1600/Mar/01 has a value of *c* of 1. We modify this equation slightly so that a new quantity *d* will be greater than zero - we use `d=(c-1+4*36524)/36524` or `d=(c+146095)/36524`.

The quantity *d*, therefore, increases by 1 every March 1<sup>st</sup> in each century year, regardless of whether it's a 400-year or not. So the quantity `e = day+d-(d/4)` re-instates the 3 "missing" century leap days in every 400 year period. The quantity *e,* therefore, increases by an average of 365.25 each year.

We are now able to calculate the year *λ*. We add a constant to *e* (to make the answer come out as 1970 on the appropriate dates) and divide by 365.25. However, using integer division, we cannot divide by 365.25, so we need to multiply everything by 4 and then divide by `365.25*4=1461`.

So we want a formula of the form `λ=(4*e+constant)/1461`. In order to determine the value of the constant, we pick some values for *e* for which we know the corresponding value of *λ* and, using the relations that the minimum value of the constant would be `1461*λ-4*e` and the maximum value would be `1461*λ-4*e+1460` we can find the value of the constant.

| Date        | λ    | Day | e   | Const. Min | Const. Max |
| ----------- | ---- | --- | --- | ---------- | ---------- |
| 1971/Mar/01 | 1971 | 424 | 430 | 2877911    | 2879371    |
| 1972/Feb/29 | 1971 | 789 | 795 | 2876451    | 2877911    |

Hence the correct constant to use is 2877911, and `λ=(4*e+2877911)/1461`.

We are now close to calculating ε. We wish to calculate a quantity which is 123 on 1<sup>st</sup> March. Noting that *e=796* on 1<sup>st</sup> March 1972, and `365*λ+(λ/4)` is 720273 when *λ=1972,* if we calculate the quantity `e+719600-365*λ-(λ/4)` we get 123 for that date. Hence `f=e+719600-365*λ-(λ/4)` and `ε=(5*f-1)/153`.

We note that `30*ε+((3*ε)/5)` is one less than the day number corresponding to the 1st of the month, hence our day-of-month is `f-30*ε-((3*ε)/5)`.

This gives us our final algorithm

```c
// Convert a number of days since 1970 into y,m,d. 0<=m<=11
// https://github.com/deirdreobyrne/CalendarAndDST
void getDateFromDayNumber(int day, int *y, int *m, int *date) {
  int a = day + 135081;
  int b,c,d;
  a = (a-(a/146097)+146095)/36524;
  a = day + a - (a>>2);
  b = ((a<<2)+2877911)/1461;
  c = a + 719600 - 365*b - (b>>2);
  d = (5*c-1)/153;
  if (date) *date=c-30*d-((3*d)/5);
  if (m) {
    if (d<14)
      *m=d-2;
    else
      *m=d-14;
  }
  if (y) {
    if (d>13)
      *y=b+1;
    else
      *y=b;
  }
}
```

## Daylight savings time

The algorithm to account for daylight savings time is quite simple. Just calculate when daylight savings time starts and ends in the current year, and then figure out whether DST is in effect.

All of the current (June 2022) [rules for determining when DST starts and ends](https://en.wikipedia.org/wiki/Daylight_saving_time_by_country) (with the exception of the rules for Iran) can be summarised as

```
The [Friday before|day of]
    the [first|second|fourth|last]
    [Thursday|Friday|Saturday|Sunday]
    of [February|March|April|September|October|November]
    at (time-of-day)
```

We use a set of 12 numbers to configure our DST rules. These are

- *dstOffset* - The number of minutes daylight savings time adds to the clock (usually 60)
- *timezone* - The time zone, in minutes, when DST is not in effect - positive east of Greenwich
- *startDowNumber* - The index of the day-of-week in the month when DST starts - 0 for first, 1 for second, 2 for third, 3 for fourth and 4 for last
- *startDow* - The day-of-week for the DST start calculation - 0 for Sunday, 6 for Saturday
- *startMonth* - The number of the month that DST starts - 0 for January, 11 for December
- *startDayOffset* - The number of days between the selected day-of-week and the actual day that DST starts - usually 0
- *startTimeOfDay* - The number of minutes elapsed in the day before DST starts
- *endDowNumber* - The index of the day-of-week in the month when DST ends - 0 for first, 1 for second, 2 for third, 3 for fourth and 4 for last
- *endDow* - The day-of-week for the DST end calculation - 0 for Sunday, 6 for Saturday
- *endMonth* - The number of the month that DST ends - 0 for January, 11 for December
- *endDayOffset* - The number of days between the selected day-of-week and the actual day that DST ends - usually 0
- *endTimeOfDay* - The number of minutes elapsed in the day before DST ends

To determine what the `dowNumber, dow, month, dayOffset, timeOfDay` parameters should be, start with a sentence of the form *"DST starts on the last Sunday of March (plus 0 days) at 03:00"*. Since it's the last Sunday, we have startDowNumber = 4, and since it's Sunday, we have startDow = 0. That it is March gives us startMonth = 2, and that the offset is zero days, we have startDayOffset = 0. The time that DST starts gives us startTimeOfDay = 180.

*"DST ends on the Friday before the second Sunday in November at 02:00"* would give us endDowNumber=1, endDow=0, endMonth=10, endDayOffset=-2 and endTimeOfDay=120.

Using Ukraine as an example, we have a time which is 2 hours ahead of GMT in winter (EET) and 3 hours in summer (EEST). DST starts at 03:00 EET on the last Sunday in March, and ends at 04:00 EEST on the last Sunday in October. So someone in Ukraine might use the parameters (60,120,4,0,2,0,180,4,0,9,0,240).

Hence our algorithm is

```c
// Given a set of DST change settings, calculate the time (in GMT
// minutes since 1970) that the change happens in year y
//
// If is_start is true, then the given parameters are referring to
// the start of DST
//
// If as_local_time is true, then returns the number of minutes in
// the timezone in effect, as opposed to GMT
//
// https://github.com/deirdreobyrne/CalendarAndDST
int getDstChangeTime(int y, int dow_number, int dow, int month,
    int day_offset, int timeOfDay, int dst_offset,
    int timezone, bool is_start, bool as_local_time) {
  int ans;
  if (dow_number == 4) { // last X of this month?
    if (++month > 11) { // ... work backwards from 1st of next month
      y++;
      month-=12;
    }
  }
  ans = getDayNumberFromDate(y, month, 1); // (ans + 4) % 7 is 0 for SUN
  // ((14 - ((ans + 4) % 7) + dow) % 7) is zero if 1st is our dow,
  // 1 if 1st is the day before our dow etc
  if (dow_number == 4) {
    ans += ((14 - ((ans + 4) % 7) + dow) % 7) - 7;
  } else {
    ans += 7 * dow_number + (14 - ((ans + 4) % 7) + dow) % 7;
  }
  ans = (ans + day_offset) * 1440 + timeOfDay;
  if (!as_local_time) {
    ans -= timezone;
    if (!is_start) ans -= dst_offset;
  }
  return ans;
}

// Returns the effective timezone in minutes east
//
// params is the set of 12 numbers described above
//
// ms is the number of milliseconds since 1970 GMT
//
// is_local_time is true if ms is referenced to local time,
// false if it's referenced to GMT
//
// if is_dst is not zero, then it will be set to true if DST is
// in effect
//
// https://github.com/deirdreobyrne/CalendarAndDST
int getEffectiveTimeZone(double ms, int *params, bool is_local_time,
    bool *is_dst) {
  int y;
  int minutes = (int)(ms/60000);
  int dstStart,dstEnd;
  bool dstActive;
      
  getDateFromDayNumber((int)(minutes/1440),&y,0,0);
  dstStart = getDstChangeTime(y, params[2], params[3],
    params[4], params[5], params[6], params[0], params[1], 1,
    is_local_time);
  dstEnd = getDstChangeTime(y, params[7], params[8], params[9],
    params[10], params[11], params[0], params[1], 0, is_local_time);
  //
  // Now, check all permutations and combinations, noting that whereas
  // in the northern hemisphere, dstStart<dstEnd, in the southern
  // hemisphere dstEnd<dstStart
  //
  // ((minutes >= dstStart) | (minutes < dstEnd)) ^ (dstStart < dstEnd)
  // ((minutes < dstStart) | (minutes >= dstEnd)) ^ (dstEnd < dstStart)
  //
  if (minutes < dstStart) {
    if (minutes < dstEnd) {
      if (dstStart < dstEnd) {
        // Northern hemisphere - DST hasn't started yet
        dstActive=false;
      } else {
        // Southern hemisphere - DST hasn't ended yet
        dstActive=true;
      }
    } else { // dstEnd <= minutes < dstStart
      // Southern hemisphere - DST has ended for the winter
      dstActive=false;
    }
  } else { // minutes >= dstStart
    if (minutes >= dstEnd) {
      if (dstStart < dstEnd) {
        // Northern hemisphere - DST has ended
        dstActive=false;
      } else {
        // Southern hemisphere - DST has started
        dstActive=true;
      }
    } else { // minutes >= dstStart, minutes < dstEnd
      // Northern hemisphere - DST has started for the summer
      dstActive=true;
    }
  }
  if (is_dst) *is_dst=dstActive;
  return dstActive ? params[0]+params[1] : params[1];
}

```
