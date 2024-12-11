---
layout: post
title:  "Advent of Code 2024 day 10, RPG edition"
date:   2024-12-10 20:00:00 -0400
categories: adventofcode rpg
---

This is the eleventh day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/11).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing numbers engraved on stones.

Each time we blink, the number on the stones and the number of stones changes (see the rules in the puzzle definition).

We need to calculate how many stones there will be after blinking 25 times.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

We load the values in a dynamic array with ```%SPLIT``` built-in function.

We loop 25 times and apply the rules to each stone value to build the next iteration array.

The result is the number of elements in the array after the last iteration.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-s fileName varchar(50);
dcl-s data char(255);
dcl-s value char(255);
dcl-s i int(5);
dcl-s stone int(20);
dcl-s result int(10) inz(0);

dcl-s currentState int(20) dim(*auto:1677310);
dcl-s newState int(20) dim(*auto:1677310);

fileName = %trim(input);

// Read the input data from the IFS
exec sql select line into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Load current State from input data
for-each value in %split(data);
  currentState(*next) = %int(value);
endfor;

for i = 1 to 25;
  for-each stone in currentState;
    if stone = 0;
      newState(*next) = 1;
    else;
      if %rem(%len(%char(stone)):2) = 0;
        newState(*next) = %int(%subst(%char(stone):1:%div(%len(%char(stone)):2)));
        newState(*next) = %int(%subst(%char(stone):%div(%len(%char(stone)):2)+1));
      else;
        newState(*next) = stone*2024;
      endif;
    endif;
  endfor;
  currentState = newState;
  %elem(newState) = 0;
endfor;

result = %elem(currentState);

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

Part 2 definition is extremely simple: you blink 75 times instead of 25.

The obvious solution is to change the loop to iterate 75 times instead of 25. Problem is the method used in part 1 is exponential in memory and processing required so it's basically impossible to do 75 iterations (with my input data, the final array alone would have used more than 3 PB of memory).

Although the number of stones grows exponentially and gets really huge, the number of distinct values on the stones is significantly lower (3772 after 75 iterations with my data) and the order and position of the stones is totally irrelevant (on the value of a stone determines what happens to it during an iteration) therefore, we only need to keep track of the different values on the stones and the number of stones with each value.

Each iteration becomes very fast and the memory needed is very low (the final array uses 60 KB with my input data).

After the last iteration, we just need to sum the number of stones for each value.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr addState;
  stone uns(20) value;
  count uns(20) value;
end-pr;

dcl-s fileName varchar(50);
dcl-s data char(255);
dcl-s value char(255);
dcl-s result uns(20) inz(0);
dcl-s stone uns(20);

dcl-ds state qualified;
  stone uns(20);
  count uns(20);
end-ds;

dcl-ds currentState likeds(state) dim(*auto:1000000);
dcl-ds newState likeds(state) dim(*auto:1000000);

dcl-s i int(5);

fileName = %trim(input);

// Read the input data from the IFS
exec sql select line into :data from table(qsys2.ifs_read_utf8(path_name => :fileName));

// Load current State from input data
for-each value in %split(data);
  state.stone = %int(value);
  state.count = 1;
  currentState(*next) = state;
endfor;

for i = 1 to 75;
  for-each state in currentState;
    if state.stone = 0;
      addState(1:state.count);
      else;
      if %rem(%len(%char(state.stone)):2) = 0;
        addState(%int(%subst(%char(state.stone):1:%div(%len(%char(state.stone)):2))):state.count);
        addState(%int(%subst(%char(state.stone):%div(%len(%char(state.stone)):2)+1)):state.count);
      else;
        addState(state.stone*2024:state.count);
      endif;
    endif;
  endfor;
  %elem(currentState) = %elem(newState);
  currentState = newState;
  %elem(newState) = 0;
endfor;

for-each state in currentState;
  result += state.count;
endfor;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc addState;

dcl-pi *n;
  stone uns(20) value;
  count uns(20) value;
end-pi;

dcl-ds wState likeds(state);
dcl-s i int(10);

i = %lookup(stone:newState(*).stone);
if i = 0;
  wState.stone = stone;
  wState.count = count;
  newState(*next) = wState;
else;
  newState(i).count += count;
endif;

end-proc;</pre>