`a = day + 135081` makes 1600/Feb/29 day zero

`a - (a / 146097)` gets rid of the 400-year leap days, resulting in a calendar with a period of 100 years of 36,524 days each.

We now need a number corresponding to which century it is. Since the century starts on day number `1 + n*36524` in terms of a, we need to subtract 1 from a first

`146095 = (4 * 36524) - 1`, so we are effectively adding 4 to the century number while at the same time subtracting 1 from a.

`a = day + a - (a>>2)` - we are adding in the 3 leap years every 400 years. This gives us a calendar of 365.25 day years, or 1461/4 day years.

Day number 59 corresponds to 1970/Mar/01, which is the first day of "1970" in our altered calendar. When the day number is 59, a is 65. `1970 * 1461 - 65 * 4 = 2877910`. We add in the extra 1 for good measure, to ensure no rounding errors when dividing by 1461.

d is the new day number to 1st March in the year in question

We use a concise formula for the cumulative number of days in the month, namely `30*e + ((3*e)/5)`, where e runs from 4 for March to 14 for January and 15 for February.

| Month     | e   | `d=30*e+((3*e)/5)` | Diff to previous | `(5*d+4)/153` |
| --------- | --- | ------------------ | ---------------- | ------------- |
| March     | 4   | 122                | n/a              | 4 + 2/153     |
| April     | 5   | 153                | 31               | 5 + 4/153     |
| May       | 6   | 183                | 30               | 6 + 1/153     |
| June      | 7   | 214                | 31               | 7 + 3/153     |
| July      | 8   | 244                | 30               | 8             |
| August    | 9   | 275                | 31               | 9 + 2/153     |
| September | 10  | 306                | 31               | 10 + 4/153    |
| October   | 11  | 336                | 30               | 11 + 1/153    |
| November  | 12  | 367                | 31               | 12 + 3/153    |
| December  | 13  | 397                | 30               | 13            |
| January   | 14  | 428                | 31               | 14 + 2/153    |
| February  | 15  | 459                | 31               | 15 + 4/153    |

So our "months" are 153/5 days on average.

`a + 719600 - d` gives us 123 when the date is 1 March. So as we can see from the table above, the integer part of `(5*b-1)/153` gives us e.








