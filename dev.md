# The Gregorian Calendar, and DST

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
int getDayNumberFromDate(int y, int m, int d) {
  int ans;

  if (m < 2) {
    y--;
    m+=12;
  }
  ans = (y/100);
  ans = 365*y + (y>>2) - ans + (ans>>2) + 30*m + ((3*m+6)/5) + d - 719531;
  return ans;
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

We are now able to calculate the year *λ*. We add a constant to *e* (to make the answer come out as 1970 on the appropriate dates) and divide by 365.25. However, using integer division, we cannot divide by 365.25, so we need to multiply everything by 4 and then divide by `365.25*4=1461`. It has been found that the correct constant to use is 2877911. Hence `λ=(4*e+2877911)/1461`.

We are now close to calculating ε. We wish to calculate a quantity which is 123 on 1<sup>st</sup> March. Noting that *e=796* on 1<sup>st</sup> March 1972, and `365*λ+(λ/4)` is 720273 when *λ=1972,* if we calculate the quantity `e+719600-365*λ-(λ/4)` we get 123 for that date. Hence `f=e+719600-365*λ-(λ/4)` and `ε=(5*f-1)/153`.

We note that `30*ε+((3*ε)/5)` is one less than the day number corresponding to the 1st of the month, hence our day-of-month is `f-30*ε-((3*ε)/5)`.

This gives us our final algorithm

```c
// Convert a number of days since 1970 into y,m,d. 0<=m<=11
void getDateFromDayNumber(int day, int *y, int *m, int *date) {
  int a = day + 135081;
  int b,c,d,e;
  a = (a-(a/146097)+146095)/36524;
  a = day + a - (a>>2);
  c = ((a<<2)+2877911)/1461;
  d = 365*c + (c>>2);
  b = a + 719600 - d;
  e = (5*b-1)/153;
  if (date) *date=b-30*e-((3*e)/5);
  if (m) {
    if (e<14)
      *m=e-2;
    else
      *m=e-14;
  }
  if (y) {
    if (e>13)
      *y=c+1;
    else
      *y=c;
  }
}
```


























