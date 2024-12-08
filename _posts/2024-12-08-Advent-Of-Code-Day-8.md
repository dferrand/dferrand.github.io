---
layout: post
title:  "Advent of Code 2024 day 8, RPG edition"
date:   2024-12-08 17:00:00 -0400
categories: adventofcode rpg
---

This is the eighth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/8).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a map of antennas and their frequencies. Each pair of antennas with the same frequency creates two antinode positions (see the puzzle definition for their exact definition). Some antinode positions might be out of the mapped area, we ignore those. The goal is to count the antinode positions.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function, we concatenate all the lines into a single variable with ```LISTAGG``` function.

We store the antinode positions in a separate CHAR variable that is initially all blanks.

We process every position of the map from the begining. If the position contains ```.```, it means it is empty, we ignore it. If it contains a frequency, we search forward for all identical frequencies (by only looking forward, we ensure that we process each pair only once). For each pair, we verify if each antinode position ("before" and "after") is within the map, if yes, we mark it with ```X``` in the antinode map.

We then count ```X```s in the antinode map with the ```REGEXP_COUNT```  SQL function.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 50;
dcl-c HEIGHT 50;
dcl-c LENGTH 2500;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s antinodes char(LENGTH) inz(*all' ');
dcl-s index int(5);
dcl-s searchIndex int(5);
dcl-s row int(5);
dcl-s col int(5);
dcl-s searchRow int(5);
dcl-s searchCol int(5);
dcl-s antinodeRow int(5);
dcl-s antinodeCol int(5);
dcl-s deltaRow int(5);
dcl-s deltaCol int(5);
dcl-s frequency char(1);

dcl-s result int(20) inz(0);

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select listagg(line) into : data from table(qsys2.ifs_read_utf8(path_name => :fileName));

for index = 1 to LENGTH-1;
  frequency = %subst(data:index:1);
  if frequency <> '.';
    searchIndex = %scan(frequency:data:index+1);
    row = %div(index-1:WIDTH)+1;
    col = %rem(index-1:WIDTH)+1;
    dow searchIndex > 0;
      searchCol = %rem(searchIndex-1:WIDTH)+1;
      searchRow = %div(searchIndex-1:WIDTH)+1;

      // Try the antinode "before"
      antinodeRow = row - (searchRow - row);
      antinodeCol = col - (searchCol - col);
      if antinodeRow >= 1 and antinodeRow <= WIDTH and antinodeCol >= 1 and antinodeCol <= HEIGHT;
        %subst(antinodes:(antinodeRow-1)*WIDTH+antinodeCol-1:1) = 'X';
      endif;

      // Try the antinode "after"
      antinodeRow = searchRow + (searchRow - row);
      antinodeCol = searchCol + (searchCol - col);
      if antinodeRow >= 1 and antinodeRow <= WIDTH and antinodeCol >= 1 and antinodeCol <= HEIGHT;
        %subst(antinodes:(antinodeRow-1)*WIDTH+antinodeCol-1:1) = 'X';
      endif;

      if searchIndex = LENGTH;
        searchIndex = 0;
      else;
        searchIndex = %scan(frequency:data:searchIndex+1);
      endif;
    enddo;
  endif;
endfor;

// Count the X in antinodes
exec sql select regexp_count(:antinodes, 'X') into :result from sysibm.sysdummy1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

In part 2, antinodes are at every position aligned with a pair of identical frequency antennas (that includes the antennas positions).

We slightly modify the program from part 1 to "walk in a straight line" from each antenna until we get out of the map.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 50;
dcl-c HEIGHT 50;
dcl-c LENGTH 2500;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s antinodes char(LENGTH) inz(*all' ');
dcl-s index int(5);
dcl-s searchIndex int(5);
dcl-s row int(5);
dcl-s col int(5);
dcl-s searchRow int(5);
dcl-s searchCol int(5);
dcl-s antinodeRow int(5);
dcl-s antinodeCol int(5);
dcl-s deltaRow int(5);
dcl-s deltaCol int(5);
dcl-s frequency char(1);

dcl-s result int(20) inz(0);

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select listagg(line) into : data from table(qsys2.ifs_read_utf8(path_name => :fileName));

for index = 1 to LENGTH-1;
  frequency = %subst(data:index:1);
  if frequency <> '.';
    searchIndex = %scan(frequency:data:index+1);
    row = %div(index-1:WIDTH)+1;
    col = %rem(index-1:WIDTH)+1;
    dow searchIndex > 0;
      searchCol = %rem(searchIndex-1:WIDTH)+1;
      searchRow = %div(searchIndex-1:WIDTH)+1;

      // Try the antinodes "before"
      antinodeRow = row;
      antinodeCol = col;
      dow antinodeRow >= 1 and antinodeRow <= WIDTH and antinodeCol >= 1 and antinodeCol <= HEIGHT;
        %subst(antinodes:(antinodeRow-1)*WIDTH+antinodeCol-1:1) = 'X';
        antinodeRow = antinodeRow - (searchRow - row);
        antinodeCol = antinodeCol - (searchCol - col);
      enddo;

      // Try the antinode "after"
      antinodeRow = searchRow;
      antinodeCol = searchCol;
      dow antinodeRow >= 1 and antinodeRow <= WIDTH and antinodeCol >= 1 and antinodeCol <= HEIGHT;
        %subst(antinodes:(antinodeRow-1)*WIDTH+antinodeCol-1:1) = 'X';
        antinodeRow = antinodeRow + (searchRow - row);
        antinodeCol = antinodeCol + (searchCol - col);
      enddo;

      if searchIndex = LENGTH;
        searchIndex = 0;
      else;
        searchIndex = %scan(frequency:data:searchIndex+1);
      endif;
    enddo;
  endif;
endfor;

// Count the X in antinodes
exec sql select regexp_count(:antinodes, 'X') into :result from sysibm.sysdummy1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>