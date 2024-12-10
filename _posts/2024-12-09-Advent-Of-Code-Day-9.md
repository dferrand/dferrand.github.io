---
layout: post
title:  "Advent of Code 2024 day 9, RPG edition"
date:   2024-12-09 21:20:00 -0400
categories: adventofcode rpg
---

This is the nineth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/9).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a disk map. It is a single line containing alternatively files and free space sizes (one digit for each).

We need to defragment the disk by moving blocks from files at the end of the disk to the free space at the begining of the disk. 

We then need to calculate the filesystem checksum (see the puzzle for the exact definition).

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

We defragment and compute the checksum in one pass by processing the disk map from both ends: we process alternatively the files from the lelft and fill the free space with blocks from the right.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c LENGTH 19999;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s position int(10) inz(0);
dcl-s leftID int(10) inz(0);
dcl-s rightID int(10) inz(9999);
dcl-s leftIndex  int(10) inz(1);
dcl-s rightIndex int(10) inz(19999);
dcl-s leftCount int(3);
dcl-s rightCount int(3);
dcl-s i int(3);

dcl-s result int(20) inz(0);

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select line into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

leftCount = %int(%subst(data:1:1));
rightCount = %int(%subst(data:19999:1));

dow leftIndex < rightIndex;
  for i = 1 to leftcount;
    result += position*leftID;
    position += 1;
  endfor;

  leftIndex += 1;
  leftCount = %int(%subst(data:leftindex:1));
  for i = 1 to leftCount;
    if rightCount = 0;
      rightIndex -=  2;
      rightCount = %int(%subst(data:rightIndex:1));
      rightID -= 1;
    endif;
    result += position*rightID;
    rightCount -= 1;
    position += 1;
  endfor;
  leftIndex += 1;
  leftCount = %int(%subst(data:leftIndex:1));
  leftID += 1;
enddo;

for i = 1 to rightCount;
  result += position*rightID;
  position += 1;
endfor;


snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

In part 2, we change the defragmentation rules:
* we can only move a whole file from the end of the disk to a free space
* we start moving files from the end of the disk
* we move a file to the leftmost free space big enough for the file
* if there is no free space for a file, it is not moved at all

We solve this puzzle in 3 passes:
* we compute the initial begining position of each file
* we perform the actual defragmentation be changing the begining position of each file
* we compute the checksum

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c LENGTH 19999;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s position int(10) inz(0);
dcl-s leftID int(10) inz(0);
dcl-s rightID int(10) inz(9999);
dcl-s leftIndex  int(10) inz(1);
dcl-s rightIndex int(10) inz(19999);
dcl-s leftCount int(3);
dcl-s rightCount int(3);
dcl-s i int(5);
dcl-s j int(5);
dcl-s positions int(10) dim(10000);
dcl-s fillings int(10) dim(9999);

dcl-s result int(20) inz(0);

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select line into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Calculate initial position of each File
for i = 1 to 10000;
  positions(i) = position;
  if i < 10000;
    position += %int(%subst(data:2*(i-1)+1:1))+%int(%subst(data:2*(i-1)+2:1));
  endif;
endfor;

// Defragment
for i = 19999 downto 3 by 2;
  rightCount = %int(%subst(data:i:1));
  j = 2;
  leftCount = %int(%subst(data:j:1));
  dow j<=i and rightCount > leftCount;
    j += 2;
    leftCount = %int(%subst(data:j:1));
  enddo;
  if leftCount >= rightCount;
    positions(%div(i+1:2)) = positions(%div(j:2))+%int(%subst(data:j-1:1))+fillings(%div(j:2));
    fillings(%div(j:2)) += rightCount;
    leftCount -= rightCount;
    %subst(data:j:1) = %char(leftCount);
  endif;
endfor;

// Compute checksum
for i = 1 to 10000;
  position = positions(i);
  for j = 1 to %int(%subst(data:2*(i-1)+1:1));
    result += (i-1)*position;
    position += 1;
  endfor;
endfor;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>