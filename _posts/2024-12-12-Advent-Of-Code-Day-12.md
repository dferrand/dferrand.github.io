---
layout: post
title:  "Advent of Code 2024 day 12, RPG edition"
date:   2024-12-12 14:30:00 -0400
categories: adventofcode rpg
---

This is the twelfth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/12).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a map of the garden plots.

Each plot grows only one type of plant. A region is a set of contiguous plots that grow the same type of plants.

We need to calculate the area and perimeter of each region multiply them and sum this value.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function with the ```LISTAGG``` function  to concatenate all lines.

We start at the top left plot and explore in the four directions:
* if we've already explored that plot, we do nothing
* if it's the same type of plant, we recursively explore that neighbour
* if it's another type of plant or there is no plot because we are at the edge of the map, we add one to the perimeter.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 140;
dcl-c HEIGHT 140;
dcl-c LENGTH 19600;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s i int(5);
dcl-s row int(5);
dcl-s col int(5);
dcl-s result int(10) inz(0);
dcl-s currentChar char(1);

dcl-ds dimension qualified;
  area int(5);
  perimeter int(5);
end-ds;

dcl-pr explore likeds(dimension);
  position int(5) value;
end-pr;

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select listagg(line) into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Find first char
exec sql select regexp_instr(:data, '[^ ]') into :i from sysibm.sysdummy1;

dow i <> 0;
  currentChar = %subst(data:i:1);
  
  dimension = explore(i);

  result += dimension.area * dimension.perimeter;

  data = %scanrpl('_':' ':data); // Replace _ with spaces

  // Find first char
  exec sql select regexp_instr(:data, '[^ ]') into :i from sysibm.sysdummy1;
enddo;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc explore;

dcl-pi *n likeds(dimension);
  position int(5) value;
end-pi;
dcl-s row int(5);
dcl-s col int(5);
dcl-ds retVal likeds(dimension);
dcl-ds exploreResult likeds(dimension);


if %subst(data:position:1) = '_'; // IF this is an already visited position for this region
  retVal.area = 0;
  retVal.perimeter = 0;
  return retVal;
endif;

if %subst(data:position:1) <> currentChar; // If this is a position from another region
  retVal.area = 0;
  retVal.perimeter = 1;
  return retVal;
endif;

%subst(data:position:1) = '_';

row = %div(position-1:WIDTH)+1;
col = %rem(position-1:WIDTH)+1;

retVal.area = 1;
retVal.perimeter = 0;

// Explore up
if row = 1;
  retVal.perimeter += 1;
else;
  exploreResult = explore(position-WIDTH);
  retVal.area += exploreResult.area;
  retVal.perimeter += exploreResult.perimeter;
endif;
 
// Explore right
if col = WIDTH;
  retVal.perimeter += 1;
else;
  exploreResult = explore(position+1);
  retVal.area += exploreResult.area;
  retVal.perimeter += exploreResult.perimeter;
endif;
  
// Explore down
if row = HEIGHT;
  retVal.perimeter += 1;
else;
  exploreResult = explore(position+WIDTH);
  retVal.area += exploreResult.area;
  retVal.perimeter += exploreResult.perimeter;
endif;
   
// Explore left
if col = 1;
  retVal.perimeter += 1;
else;
  exploreResult = explore(position-1);
  retVal.area += exploreResult.area;
  retVal.perimeter += exploreResult.perimeter;
endif;

return retVal;

end-proc;</pre>

# Part 2

In part 2, we need to calculate the number of sides of the regions instead of the perimeter.

For this part, we'll proceed in two passes:
* we first mark all the plots of a region
* we then scan the marked plots line by line and left to right, for each plot, we look in the four directions for an out of region plot, if this is the case, we increment the number of sides if this is the first out of region plot we encounter on that side.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 140;
dcl-c HEIGHT 140;
dcl-c LENGTH 19600;

dcl-s fileName varchar(50);
dcl-s data char(LENGTH);
dcl-s i int(5);
dcl-s j int(5);
dcl-s row int(5);
dcl-s col int(5);
dcl-s result int(10) inz(0);
dcl-s currentChar char(1);
dcl-s area int(5);
dcl-s sides int(5);

dcl-pr explore;
  position int(5) value;
end-pr;

dcl-pr measure;
  position int(5) value;
end-pr;

fileName = %trim(input);

// Read the input data from the IFS, concatenating all lines
exec sql select listagg(line) into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Find first char
exec sql select regexp_instr(:data, '[^ ]') into :i from sysibm.sysdummy1;

dow i <> 0;
  currentChar = %subst(data:i:1);
  
  explore(i);

  j = i;
  dow j <> 0;
  measure(j);

  if j = LENGTH;
    j = 0;
  else;
    j = %scan('_':data:j+1);
  endif;
  enddo;

  result += area * sides;
  area = 0;
  sides = 0;

  data = %scanrpl('_':' ':data); // Replace _ with spaces

  // Find first char
  exec sql select regexp_instr(:data, '[^ ]') into :i from sysibm.sysdummy1;
enddo;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc explore;

dcl-pi *n;
  position int(5) value;
end-pi;
dcl-s row int(5);
dcl-s col int(5);

if %subst(data:position:1) <> currentChar; // If this is a position from another region or already visited
  return;
endif;

%subst(data:position:1) = '_';

row = %div(position-1:WIDTH)+1;
col = %rem(position-1:WIDTH)+1;

// Explore up
if row > 1;
  explore(position-WIDTH);
endif;
 
// Explore right
if col < WIDTH;
  explore(position+1);
endif;
  
// Explore down
if row < HEIGHT;
  explore(position+WIDTH);
endif;
   
// Explore left
if col > 1;
  explore(position-1);
endif;

end-proc;

dcl-proc measure;

dcl-pi *n;
  position int(5) value;
end-pi;
dcl-s row int(5);
dcl-s col int(5);

row = %div(position-1:WIDTH)+1;
col = %rem(position-1:WIDTH)+1;

// Check up
if (row = 1 or %subst(data:position-WIDTH:1) <> '_') and (
  col = 1 or not(%subst(data:position-1:1) = '_' and (row = 1 or %subst(data:position-WIDTH-1:1) <> '_')));
  sides += 1;
endif;
 
// Check right
if (col = WIDTH or %subst(data:position+1:1) <> '_') and (
  row = 1 or not(%subst(data:position-WIDTH:1) = '_' and (col = WIDTH or %subst(data:position-WIDTH+1:1) <> '_')));
  sides += 1;
endif;
 
// Check down
if (row = HEIGHT or %subst(data:position+WIDTH:1) <> '_') and (
  col = 1 or not(%subst(data:position-1:1) = '_' and (row = HEIGHT or %subst(data:position+WIDTH-1:1) <> '_')));
  sides += 1;
endif;
  
// Check left
if (col = 1 or %subst(data:position-1:1) <> '_') and (
  row = 1 or not(%subst(data:position-WIDTH:1) = '_' and (col = 1 or %subst(data:position-WIDTH-1:1) <> '_')));
  sides += 1;
endif;
    
area += 1;

end-proc;</pre>