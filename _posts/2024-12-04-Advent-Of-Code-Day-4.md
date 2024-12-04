---
layout: post
title:  "Advent of Code 2024 day 4, RPG edition"
date:   2024-12-03 12:00:00 -0400
categories: adventofcode rpg
---

This is the fourth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/4).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a word search. The only word to find is XMAS, it can be written in any direction (horizontal, vertical, diagonal, forware or backward).

We need to find how many XMAS are in the input data.

The IFS file can easily be read using [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function. We concatenate each line into a single variable with the ```LISTAGG``` SQL function.

We then scan characters sequentially looking for an X.

If the character is an X, we look in 8 directions unless we're to close to a border, for that, we keep track of the line and column we're currently in. If we find the word XMAS, we increment the counter.

We could use ```%SUBST``` to access any character in the input, instead we're using pointers for a (maybe?) slightly lighter syntax.

The procedure ```isXmas``` is responsible for looking for the word XMAS. Since we 'flattened' the input grid into a linear string, travelling in any direction, is just a matter of adding a given offset each time:
* to go to the right, the offset is +1
* to go to the left, the offset is -1
* to go down, the offset is +WIDTH
* to go up, the offset is -WIDTH
* to go top down, the offset is WIDTH+1
* etc...

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr isXmas ind;
  offset int(10) value;
end-pr;

dcl-s fileName varchar(50);

dcl-c WIDTH 140;
dcl-c HEIGHT 140;
dcl-c LENGTH const(19600);

dcl-s data char(LENGTH);
dcl-s result int(10) inz(0);
dcl-s line int(10) inz(1);
dcl-s col int(10) inz(1);
dcl-s index int(10);
dcl-s pCurChar pointer;
dcl-s curChar char(1) based(pCurChar);
dcl-s pScanChar pointer;
dcl-s scanChar char(1) based(pScanChar);

fileName = %trim(input);

// Read the input data from the IFS concatenating all lines
exec sql select listagg(line) into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

pCurChar = %addr(data);
for index = 1 to length;
  if curChar = 'X';
    if col <= WIDTH-3 and isXmas(1);  // Checking to the right
      result += 1;
    endif;
    if col <= WIDTH-3 and line <= HEIGHT-3 and isXmas(WIDTH+1); // Checking bottom right
      result += 1;
    endif;
    if line <= HEIGHT-3 and isXmas(WIDTH); // checking down
      result += 1;
    endif;
    if line <= HEIGHT-3 and col >= 4 and isXmas(WIDTH-1); // Checking bottom left
      result += 1;
    endif;
    if col >= 4 and isXmas(-1); // Checking left
      result += 1;
    endif;
    if col >= 4 and line >=4 and isXmas(-WIDTH-1); // Checking top left
      result += 1;
    endif;
    if line >= 4 and isXmas(-WIDTH); // Checking up
      result += 1;
    endif;
    if line >= 4 and col <= WIDTH-3 and isXmas(-WIDTH+1); // Checking top right
      result += 1;
    endif;
  endif;
  pCurChar += 1;
  if col < WIDTH;
    col += 1;
  else;
    col = 1;
    line += 1;
  endif;
endfor;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc isXmas;

dcl-pi *n ind;
  offset int(10) value;
end-pi;
pScanChar = pCurChar + offset;
if scanChar <> 'M';
  return *off;
endif;
pScanChar += offset;
if scanChar <> 'A';
  return *off;
endif;
pScanChar += offset;
if scanChar <> 'S';
  return *off;
endif;

return *on;

end-proc;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

There was a misunderstanding, it is an X-MAS puzzle, not an XMAS puzzle so we need to find two ```MAS``` in the shape of an X.

For example:
<pre>M.S
.A.
M.S</pre>
or
<pre>M.M
.A.
S.S</pre>

This time, we will look for an A and check if it is the center of an X made with MAS.

Since A is the center of an X, we don't need to look at the first and last lines and columns.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-s fileName varchar(50);

dcl-c WIDTH 140;
dcl-c HEIGHT 140;
dcl-c LENGTH const(19600);

dcl-s data char(LENGTH);
dcl-s result int(10) inz(0);
dcl-s line int(10) inz(2);
dcl-s col int(10) inz(2);
dcl-s index int(10);
dcl-s pCurChar pointer;
dcl-s curChar char(1) based(pCurChar);
dcl-s pScanChar1 pointer;
dcl-s scanChar1 char(1) based(pScanChar1);
dcl-s pScanChar2 pointer;
dcl-s scanChar2 char(1) based(pScanChar2);
dcl-s pScanChar3 pointer;
dcl-s scanChar3 char(1) based(pScanChar3);
dcl-s pScanChar4 pointer;
dcl-s scanChar4 char(1) based(pScanChar4);

fileName = %trim(input);

// Read the input data from the IFS concatenating all lines
exec sql select listagg(line) into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

pCurChar = %addr(data)+WIDTH+1;        // We start on the second column of the second line
for index = 1 to (WIDTH-2)*(HEIGHT-2); // We skip the first and last lines and columns
  if curChar = 'A';
    pScanChar1 = pCurChar - WIDTH - 1; // top left
    pScanChar2 = pCurChar - WIDTH + 1; // top right
    pScanChar3 = pCurChar + WIDTH - 1; // bottom left
    pScanChar4 = pCurChar + WIDTH + 1; // bottom right

    if (scanChar1 = 'M' and scanChar4 = 'S') or (scanChar1 = 'S' and scanChar4='M');
      if (scanChar2 = 'M' and scanChar3 = 'S') or (scanChar2 = 'S' and scanChar3='M');
        result += 1;
      endif;
    endif;
  endif;
  if col < WIDTH-1;  // If we are not on the second to last column, we advance to the next column
    col += 1;
    pCurChar += 1;
  else;              // If we are on the second to last column, we skip to the second column of next line
    col = 2;
    line += 1;
    pCurChar += 3;
  endif;
endfor;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>