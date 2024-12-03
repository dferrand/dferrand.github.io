---
layout: post
title:  "Advent of Code 2024 day 2, RPG edition"
date:   2024-12-02 20:35:00 -0400
categories: adventofcode rpg
---

This is the second day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/2).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing unusual data. Each line is a report. Each report contains multiple levels, separated by spaces.

Each report can be safe or unsafe. To be safe, the following conditions must be met:
* the levels are either all increasing or all decreasing
* any two adjacent levels differ by at least one and at most three

We need to calculate how many reports are safe.

The IFS file can easily be read using [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

Each report can be devided into levels with the %SPLIT RPG built-in function. We can iterate over the levels extracted by %SPLIT with the for-each instruction.

On the first level, we don't need to do any check.

On the second level, we determine if the report is increasing or decreasing and we check if it differs from the first one by a value of 1, 2 or 3.

Starting at the third level, we check if the direction of the report is correct and if the difference from the previous level is between 1 and 3.

As soon as a level is incorrect, the report is considered unsafe.

If we go over all levels without any incorrect check, the report is safe.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr isSafe ind;
  data varchar(255);
end-pr;

dcl-s fileName varchar(50);

dcl-s data varchar(255);
dcl-s safeReports int(10) inz(0);

fileName = %trim(input);

// Read the input data from the IFS one line at a time
exec sql declare c1 cursor for select cast(line as varchar(255)) from table(qsys2.ifs_read_utf8(path_name => :fileName));

exec sql open c1;

// Read first line
exec sql fetch from c1 into :data;

// Loop until the end of the file
dow sqlcode = 0;

  // If the report is safe we increment the safe counter
  if isSafe(data);
    safeReports += 1;
  endif;

  // Read next line
  exec sql fetch from c1 into :data;
enddo;

exec sql close c1;

snd-msg *info 'Safe reports: ' + %char(safeReports) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc isSafe;

dcl-pi *n ind;
  data varchar(255);
end-pi;

dcl-s value char(5);        // Current value as a string
dcl-s previous int(10);     // Previous value
dcl-s current int(10);      // Current value as an integer
dcl-s index int(10) inz(1); // Current index of the value in the report (starts at 1)
dcl-s increasing ind;       // *on if the reports values are increasing, *off if decreasing

// Split the report into values and loop over each value
for-each value in %split(data);
  current = %int(value); // Convert the current value to integer

  if index > 1; // We only start testing at the second value
    if index = 2; // On the second value, we determine if the report is increasing or decreasing
      increasing = current > previous;
    else;         // After the second value, we check if the report keeps increasing or decreasing
      if increasing <> (current > previous); // If the current value doesn't conform to the report direction, the report is unsafe
        return *off;
      endif;
    endif;
    if not (%abs(current - previous) in %range(1:3)); // If the difference from the previous value is not between 1 and 3, the report is unsafe
      return *off;
    endif;
  endif;

  index += 1;          // Increase the index
  previous = current;  // The current value is stored as the previous value for the next iteration
endfor;

return *on; // If we get here, the report isn't unsafe, therefore it is safe

end-proc;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

We still need to calculate how many reports are safe but this time, we tolerate, at most, one incorrect level in a report (meaning that by removing one level, we get a report with only correct levels).

We modify the isSafe procedure by adding a second parameter that is the index of the level we want to skip. If this index is 0, it means that we don't skip any level.

When we encounter an incorrect level, instead of declaring the report as unsafe, we call the isSafe procedure again but skipping the current index (and the n-1 and n-2 in some cases) if we are not already skipping a level. If by skipping a level, the report is safe then we consider the report safe.

<pre>**free
ctl-opt dftactgrp(*no);

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr isSafe ind;
  data varchar(255);
  skip int(10) value;
end-pr;

dcl-s fileName varchar(50);

dcl-s data varchar(255);
dcl-s safeReports int(10) inz(0);

fileName = %trim(input);

exec sql declare c1 cursor for select cast(line as varchar(255)) from table(qsys2.ifs_read_utf8(path_name => :fileName));

exec sql open c1;

exec sql fetch from c1 into :data;

dow sqlcode = 0;

  if isSafe(data:0);
    safeReports += 1;
  endif;

  exec sql fetch from c1 into :data;
enddo;

exec sql close c1;

snd-msg *info 'Safe reports: ' + %char(safeReports) %target(*pgmbdy:1);

*inlr = *on;
return;

dcl-proc isSafe;

dcl-pi *n ind;
  data varchar(255);
  skip int(10) value;  // if skip is 0, there is no skip. If skip > 0, this is the index to skip. Passed by value for ease of calling
end-pi;

dcl-s value char(5);
dcl-s previous int(10);
dcl-s current int(10);
dcl-s index int(10) inz(1);
dcl-s realIndex int(10) inz(1); // This is the real index that doesn't take skipping into account
dcl-s increasing ind;

for-each value in %split(data);
  if realIndex <> skip;             // If the current index is the one to be skipped, then we skip
  current = %int(value);

  if index > 1;
    if index = 2;
      increasing = current > previous;
    else;
      if increasing <> (current > previous);
        if skip > 0;                 // If we are already skipping and still have a bad level, the report is unsafe since the Problem Dampener can only tolerate one bad level
          return *off;
        else;
          if index = 3;              // The 3rd value is a special case, the bad level can be fixed by removing the first, second or third value
            return isSafe(data:3) or isSafe(data:2) or isSafe(data:1);
          else;                      // After the third value, the direction can only be fixed by removing the current value
            return isSafe(data:index);
          endif;
        endif;
      endif;
    endif;
    if not (%abs(current - previous) in %range(1:3));
      if skip >0;         // If we are already skipping and still have a bad level, the report is unsafe since the Problem Dampener can only tolerate one bad level
        return *off;
      else;               // We try to fix the report by removing the current or the previous value
        return isSafe(data:index) or isSafe(data:index-1);
      endif;
    endif;
  endif;

  index += 1;
  previous = current;
  endif;
  realIndex += 1;   // realIndex is incremented even if we are skipping this value
endfor;

return *on;

end-proc;</pre>