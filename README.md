_Based on algorithms in Jean Meeus' 1991 book "Astronomical Algorithms"_

**todo** spit and polish

```c
// DST Settings
//
//  dstOffset - The number of minutes daylight savings time adds to the clock (usually 60)
//  timezone - The time zone, in minutes, when DST is not in effect - positive east of Greenwich
//  startDowNumber - The index of the day-of-week in the month when DST starts - 0 for first, 1 for second, 2 for third, 3 for fourth and 4 for last
//  startDow - The day-of-week for the DST start calculation - 0 for Sunday, 6 for Saturday
//  startMonth - The number of the month that DST starts - 0 for January, 11 for December
//  startDayOffset - The number of days between the selected day-of-week and the actual day that DST starts - usually 0
//  startTimeOfDay - The number of minutes elapsed in the day before DST starts
//  endDowNumber - The index of the day-of-week in the month when DST ends - 0 for first, 1 for second, 2 for third, 3 for fourth and 4 for last
//  endDow - The day-of-week for the DST end calculation - 0 for Sunday, 6 for Saturday
//  endMonth - The number of the month that DST ends - 0 for January, 11 for December
//  endDayOffset - The number of days between the selected day-of-week and the actual day that DST ends - usually 0
//  endTimeOfDay - The number of minutes elapsed in the day before DST ends
//
// To determine what the `dowNumber, dow, month, dayOffset, timeOfDay` parameters should be, start with
// a sentence of the form "DST starts on the last Sunday of March (plus 0 days) at 03:00". Since it's
// the last Sunday, we have startDowNumber = 4, and since it's Sunday, we have startDow = 0. That it
// is March gives us startMonth = 2, and that the offset is zero days, we have startDayOffset = 0. The
// time that DST starts gives us startTimeOfDay = 3*60.
//
// "DST ends on the Friday before the second Sunday in November at 02:00" would give us endDowNumber=1,
// endDow=0, endMonth=10, endDayOffset=-2 and endTimeOfDay=120.
//
// Using Ukraine as an example, we have a time which is 2 hours ahead of GMT in winter (EET) and 3 hours
// in summer (EEST). DST starts at 03:00 EET on the last Sunday in March, and ends at 04:00 EEST on the
// last Sunday in October. So someone in Ukraine might use the parameters (60,120,4,0,2,0,180,4,0,9,0,240)
//

// Convert a y,m,d into a number of days since 1970. 0<=m<=11
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

// Given a set of DST change settings, calculate the time (in GMT
// seconds since 1970) that the change happens in year y
//
// If as_local_time is true, then returns the number of seconds in
// the timezone in effect, as opposed to GMT
long getDstChangeTime(int y, int dow_number, int month, int dow,
      int day_offset, int timeOfDay, bool is_start, int dst_offset,
      int timezone, bool as_local_time) {
  int m = month;
  int ans;
  if (dow_number == 4) { // last X of this month?
    if (++m > 11) { // Work backwards from 1st of next month
      y++;
      m-=12;
    }
  }
  ans = getDayNumberFromDate(y, m, 1); // (ans + 4) % 7 is 0 for SUN
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
  return 60l*ans;
}

// Returns the effective timezone in minutes east
//
// is_local_time is true if ms is referenced to local time,
// false if it's referenced to GMT
//
// if is_dst is not zero, then it will be set to true if DST is
// in effect
int getEffectiveTimeZone(long ms, bool is_local_time, bool *is_dst) {
  int y;
  long sec = ms/1000;
  long dstStart,dstEnd;

  getDateFromDayNumber(sec/86400,&y,0,0);
  dstStart = getDstChangeTime(y, startDowNumber, startMonth, startDow,
      startDayOffset, startTimeOfDay, 1, dstOffset, timezone,
      is_local_time);
  dstEnd = getDstChangeTime(y, endDowNumber, endMonth, endDow,
      endDayOffset, endTimeOfDay, 0, dstOffset, timezone,
      is_local_time);
  // Now, check all permutations and combinations, noting that whereas
  // in the northern hemisphere, dstStart<dstEnd, in the southern
  // hemisphere dstEnd<dstStart
  if (sec < dstStart) {
    if (sec < dstEnd) {
      if (dstStart < dstEnd) {
        // Northern hemisphere - DST hasn't started yet
        if (is_dst) *is_dst = false;
        return dstSetting[1];
      } else {
        // Southern hemisphere - DST hasn't ended yet
        if (is_dst) *is_dst = true;
        return dstSetting[0] + dstSetting[1];
      }
    } else { // dstEnd <= sec < dstStart
      // Southern hemisphere - DST has ended for the winter
      if (is_dst) *is_dst = false;
      return dstSetting[0];
    }
  } else { // sec >= dstStart
    if (sec >= dstEnd) {
      if (dstStart < dstEnd) {
        // Northern hemisphere - DST has ended
        if (is_dst) *is_dst = false;
        return dstSetting[1];
      } else {
        // Southern hemisphere - DST has started
        if (is_dst) *is_dst = true;
        return dstSetting[0] + dstSetting[1];
      }
    } else { // sec >= dstStart, sec < dstEnd
      // Northern hemisphere - DST has started for the summer
      if (is_dst) *is_dst = true;
      return dstSetting[0] + dstSetting[1];
    }
  }
}
```

`a = day + 135081` makes 1600/Feb/29 day zero

`a - (a / 146097)` gets rid of the 400-year leap days, resulting in a calendar with a period of 100 years of 36,524 days each.

We now need a number corresponding to which century it is. Since the century starts on day number `1 + n*36524` in terms of a, we need to subtract 1 from a first

`146095 = (4 * 36524) - 1`, so we are effectively adding 4 to the century number while at the same time subtracting 1 from a.

`a = day + a - (a>>2)` - we are adding in the 3 leap years every 400 years. This gives us a calendar of 365.25 day years, or 1461/4 day years.

Day number 59 corresponds to 1970/Mar/01, which is the first day of "1970" in our altered calendar. When the day number is 59, a is 65. `1970 * 1461 - 65 * 4 = 2877910`. We add in the extra 1 for good measure _(actually is this the reason?! - needs to be checked)_, to ensure no rounding errors when dividing by 1461.

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
