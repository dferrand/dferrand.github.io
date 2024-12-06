---
layout: post
title:  "Advent of Code 2024 day 6, RPG edition"
date:   2024-12-06 18:25:00 -0400
categories: adventofcode rpg
---

This is the sixth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/6).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a map of an area containing obstacles that is patroled by a guard. The guard walks straight until he encounters an obstacle. When that happens, he turns 90° right and walks straight. This repeats until he walk outs of the mapped area.

The goal of part one is to calculate how many positions the guard visits until he leaves the area.

Like on day 4, we read and aggregate all the lines of the map with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function and ```LISTAGG``` SQL function.

We store the current guard direction in an integer variable: 1 means up, 2 means right, 3 means down and 4 means left. Turning right means calculating the reminder of the division by 4 and adding 1.

When moving, we check if the next position is an obstacle. If this is the case, we turn right. Otherwise, we move to the next position and mark is with 'X'.

We loop until the guard gets out of the area.

We count the visited positions using a regular expression and the REGEXP_COUNT SQL function.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-s fileName varchar(50);

dcl-c WIDTH 130;
dcl-c HEIGHT 130;
dcl-c LENGTH const(16900);

dcl-s data char(LENGTH);
dcl-s result int(10) inz(0);
dcl-s line int(10);
dcl-s col int(10);
dcl-s index int(10);
dcl-s pCurChar pointer;
dcl-s curChar char(1) based(pCurChar);
dcl-s pNextChar pointer;
dcl-s nextChar char(1) based(pNextChar);
dcl-s direction int(5) inz(1);             // 1-up, 2-right, 3-down, 4-left

fileName = %trim(input);

// Read the input data from the IFS concatenating all lines
exec sql select listagg(line) into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Find the initial position
index = %scan('^':data);
line = %div(index-1:WIDTH) + 1;
col = %rem(index-1:WIDTH) + 1;

pCurChar = %addr(data) + index - 1; // Mark initial position as used.
curChar = 'X';

// We loop until we exit the area
dou (line = 1 and direction = 1) or (col = WIDTH and direction = 2) or (line = HEIGHT and direction = 3) or (col = 1 and direction = 4); 
  select direction;                    // Move pNextChar to the next position depending on the direction
    when-is 1;                         // right
      pNextChar = pCurChar - WIDTH;
    when-is 2;                         // down
      pNextChar = pCurChar + 1;
    when-is 3;                         // left
      pNextChar = pCurChar + WIDTH;
    when-is 4;                         // up
      pNextChar = pCurChar - 1;
  endsl;

  if nextChar <> '#';                  // If nextChar is not an obstacle, we proceed
    pCurChar = pNextChar;
    line = %div(pCurChar-%addr(data):WIDTH) + 1;
    col = %rem(pCurChar-%addr(data):WIDTH) + 1;
    curChar = 'X';                     // Mark the position as visited
  else;                                // If next is an obstacle, we don't move and rotate right
    direction = %rem(direction:4) + 1;
  endif;
enddo;

// Use regular expression to count the visited positions
exec sql select REGEXP_COUNT(:data, 'X') into :result from sysibm.sysdummy1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

In this part, we want to send the guard into a loop by adding one obstacle.

The goal is to calculate in how many different positions we can add one obstacle to send the guard into a loop.

We loop over all the positions that are not already an obstacle or the initial guard position and check wether adding an obstacle at this position sends the guard into a loop or not.

The trickiest part is to detect the loop. For that, we change the way we mark a visited position: instead of marking it with an X, we mark it with the direction of the guard (1, 2, 3 or 4). If we visit a position with the same direction as previously, it means that the guard is in a loop. If the guard gets out of the area, it means it's not in a loop.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr isLoop ind end-pr;

dcl-s fileName varchar(50);

dcl-c WIDTH 130;
dcl-c HEIGHT 130;
dcl-c LENGTH const(16900);

dcl-s data char(LENGTH);
dcl-s dataOrig char(LENGTH);
dcl-s result int(10) inz(0);
dcl-s line int(10);
dcl-s col int(10);
dcl-s index int(10);
dcl-s i int(10);
dcl-s pCurChar pointer;
dcl-s curChar char(1) based(pCurChar);
dcl-s pNextChar pointer;
dcl-s nextChar char(1) based(pNextChar);
dcl-s direction int(5) inz(1);             // 1-up, 2-right, 3-down, 4-left

fileName = %trim(input);

// Read the input data from the IFS concatenating all lines
exec sql select listagg(line) into :dataOrig from table(qsys2.ifs_read_utf8(path_name => :fileName));

for i = 1 to LENGTH;                       // Loop over all the positions of the map
  pNextChar = %addr(dataOrig) + i -1;
  if nextChar = '.';                       // We only add an obstacle if the position is empty
    direction = 1;                         // Reset direction to up
    data = dataOrig;                       // Reset the map to the original map
    pNextChar = %addr(data) + i -1;
    nextChar = '#';                        // Add the obstacle
    if isLoop();                           // If the guard enters a loop, we add 1 to the result
      result += 1;
    endif;
  endif;
endfor;


snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc isLoop;

dcl-pi *n ind;
end-pi;

// Find the initial position
index = %scan('^':dataOrig);
line = %div(index-1:WIDTH) + 1;
col = %rem(index-1:WIDTH) + 1;

// Mark the initial position with the original direction (up)
pCurChar = %addr(data) + index - 1;
curChar = '1';

// Loop until we exit the area
dou (line = 1 and direction = 1) or (col = WIDTH and direction = 2) or (line = HEIGHT and direction = 3) or (col = 1 and direction = 4); 
  select direction;
    when-is 1;
      pNextChar = pCurChar - WIDTH;
    when-is 2;
      pNextChar = pCurChar + 1;
    when-is 3;
      pNextChar = pCurChar + WIDTH;
    when-is 4;
      pNextChar = pCurChar - 1;
  endsl;

  if nextChar = %char(direction);          // If the next position was already visited with the same direction, then we're in a loop, we exit and return *on
    return *on;
  endif;
  if nextChar <> '#';
    pCurChar = pNextChar;
    line = %div(pCurChar-%addr(data):WIDTH) + 1;
    col = %rem(pCurChar-%addr(data):WIDTH) + 1;
    curChar = %char(direction);
  else;
    direction = %rem(direction:4) + 1;
  endif;
enddo;

return *off;                               // If the guard exits the area, that means it's not a loop
end-proc;</pre>