# The Gregorian Calendar, and DST

## Converting from y/m/d into days since 1970/Jan/01

To make our lives easier when it comes to leap years and 29<sup>th</sup> February, we define λ, ε and δ such that

- *λ= y-1* for January, February; *λ = y* for March to December

- *ε = 2 to 11* for March to December; *ε = 12, 13* for January, February

- *δ = d - 1*

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

| Month     | ε   | `d=30*ε+((3*ε)/5)` | Diff to previous | `(5*d+4)/153`  |
| --------- | --- | ------------------ | ---------------- | -------------- |
| March     | 4   | 122                | n/a              | 4 ( + 2/153 )  |
| April     | 5   | 153                | 31               | 5 ( + 4/153 )  |
| May       | 6   | 183                | 30               | 6 ( + 1/153 )  |
| June      | 7   | 214                | 31               | 7 ( + 3/153 )  |
| July      | 8   | 244                | 30               | 8              |
| August    | 9   | 275                | 31               | 9 ( + 2/153 )  |
| September | 10  | 306                | 31               | 10 ( + 4/153 ) |
| October   | 11  | 336                | 30               | 11 ( + 1/153 ) |
| November  | 12  | 367                | 31               | 12 ( + 3/153 ) |
| December  | 13  | 397                | 30               | 13             |
| January   | 14  | 428                | 31               | 14 ( + 2/153 ) |
| February  | 15  | 459                | 31               | 15 ( + 4/153 ) |

In other words, if we take March 1<sup>st</sup> as day number d=122, then if we use integer division, `(5*d+4)/153` will give us ε.

We first need to find out how many 400-year leap days have occurred. Noting that day number -135081 corresponds to 1600/Feb/29, the number of 400-year leap days is `a=(day+135081)/146097` (using integer division, as always).

The quantity *a* will be zero from 1600/Feb/29 to 2000/Feb/28, and will become 1 on 2000/Feb/29. Hence if we take `b=day-a`, we will remove the extra leap day that is inserted every 400 years. We will then be left with a quantity *b* that increases by 36524 days every 100 years. So the next task is to find out how many 100-year leap days have been removed.

The answer is, of course, `(b-1)/36524`. We need to subtract 1 from *b* to account for the fact that 1600/Mar/01 has a value of b of 1. We modify this equation slightly so that a new quantity *c* will be greater than zero - we use `c=(b-1+4*36524)/36524` or `c=(b+146095)/36524`.

The quantity *c*, therefore, increases by 1 every March 1<sup>st</sup> in each century year, regardless of whether it's a 400-year or not. So the quantity `d = day+c-(c/4)` re-instates the 3 "missing" century leap days in every 400 year period. *d*, therefore, increases by `4*365+1 = 1461` days every 4 years.


























