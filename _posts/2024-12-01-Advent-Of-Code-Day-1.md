---
layout: post
title:  "Advent of Code 2024 day 1, RPG edition"
date:   2024-12-01 21:00:00 -0400
categories: adventofcode rpg
---

[Advent of Code](https://adventofcode.com/2024/about) is an advent calendar of small programming puzzles.

Each day has two parts. The second part is unlocked when you solve the first part.

I will solve the first day puzzle using RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/1).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing 2 lists of numbers side by side.

We need to pair the numbers from each list in ascending order. For each pair, we define the distance as the difference between both elements of the pair (as a positive value or 0).

We must calculate the total distance between the lists, which is the sum of the distances of each pair.

We can download the file from the AOC website and store it in the IFS.

The IFS file can easily be read using [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

Each line contains two five digits numbers starting on column 1 and 9. Each number can be extracted with the substring function.

We can create a first SQL cursor reading and sorting the first list and a second cursor doing the same for the second list.

We can the read from both cursors and calculate the distance for each pair and sum this distance to compute the total distance.

We can then send the information to the user with the snd-msg opcode.

Here is the RPG code for part 1:
<pre>**free

// One parameter: the path to the input file
dcl-pi *n;
  input char(50);
end-pi;  


dcl-s fileName varchar(50); // ifs_read_utf8 doesn't like trailing spaces in the file name, so we use a VARCHAR to get rid of them
dcl-s val1 int(10);         // will hold the current value from the first list
dcl-s val2 int(10);         // will hold the current value from the second list
dcl-s diff int(20) inz(0);  // will hole the distance between the lists

fileName = %trim(input);

// Cursor for the first list
exec sql declare c1 cursor for select integer(substring(line, 1, 5)) v
  from table(qsys2.ifs_read_utf8(path_name => :fileName))
  order by v;

// Cursor for the second list
exec sql declare c2 cursor for select integer(substring(line, 9, 5)) v
  from table(qsys2.ifs_read_utf8(path_name => :fileName))
  order by v;

exec sql open c1;
exec sql open c2;

// Read the first pair
exec sql fetch from c1 into :val1;
exec sql fetch from c2 into :val2;

dow sqlcode = 0;
  diff += %abs(val1-val2);   // distance is the absolute value of the difference

  // Read the next pair.
  exec sql fetch from c1 into :val1;
  exec sql fetch from c2 into :val2;
enddo;

exec sql close c1;
exec sql close c2;

// Give the answer to the user
snd-msg *info 'Distance is '+%char(diff) %target(*pgmbdy:1);

*inlr = *on;
return;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

This time, we'll calculate the similarity score: for each element of the first list, the similarity score is the value of the element multiplied by the number of times it appears in the second list.

This can be done with a single SQL instruction, here is the full RPG code:
<pre>**free

// One parameter: the path to the input file
dcl-pi *n;
  input char(50);
end-pi;  

dcl-s fileName varchar(50); // ifs_read_utf8 doesn't like trailing spaces in the file name, so we use a VARCHAR to get rid of them
dcl-s sim int(20) inz(0);   // Will contain the similarity score

fileName = %trim(input);

// We use SELECT INTO since we're reading a single value
exec sql select sum(integer(substring(a.line, 1, 5)) * (select
  count(*) from table(qsys2.ifs_read_utf8(path_name => :fileName)) b
  where substring(b.line, 9, 5) = substring(a.line, 1, 5))) into :sim 
  from table(qsys2.ifs_read_utf8(path_name => :fileName)) a;

// Give the answer to the user
snd-msg *info 'Similarity score is '+%char(sim) %target(*pgmbdy:1);

*inlr = *on;
return;</pre>