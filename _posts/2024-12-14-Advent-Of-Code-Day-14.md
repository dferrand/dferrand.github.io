---
layout: post
title:  "Advent of Code 2024 day 14, RPG edition"
date:   2024-12-14 22:15:00 -0400
categories: adventofcode rpg
---

This is the fourteenth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/14).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing robots initial positions and speeds.

The robots teleport at the other en of the map when they reach the border.

We need to calculate how many robots are in the four quadrants of the map after 100 seconds and multiply the quantity of robots in each quadrant.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

Rather than calculating the position of the robots each second, we can directly calculate the position of a robot after 100 seconds by multiplying the speed by 100 and adding the initial position, then calculating the reminder of the division by the width or they heighth of the map (depending on the axis) with the ```%REM``` built-in function.

We also don't need to calculate the number of robots in each position, only in each quadrant.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 101;
dcl-c HEIGHT 103;
dcl-c MID_X 50;
dcl-c MID_Y 51;
dcl-c LENGTH 10403;
dcl-c TIME 100;

dcl-s fileName varchar(50);

dcl-s data varchar(255);
dcl-s x int(3);
dcl-s y int(3);
dcl-s vx int(3);
dcl-s vy int(3);

dcl-s result int(20) inz(0);

dcl-s map uns(5) dim(4); // We store the number of robots per quarter
dcl-s quadrant uns(3);

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select line from table(qsys2.ifs_read_utf8(path_name => :fileName));
exec sql open c1;

// Read first line
exec sql fetch c1 into :data;
  dow sqlcode = 0;
  // Read initial position and speed
  exec sql set :x = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 1));
  exec sql set :y = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 2));
  exec sql set :vx = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 3));
  exec sql set :vy = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 4));

  // Compute final position
  x = %rem(x+vx*TIME:WIDTH);
  y = %rem(y+vy*TIME:HEIGHT);

  // Adjust x and y if they are negative
  if x < 0;
    x += WIDTH;
  endif;
  if y < 0;
    y += HEIGHT;
  endif;

  // We ignore robots in the middle
  if x<>MID_X and y<>MID_Y;
    // determine the quadrant
    if x<MID_X;
      quadrant = 1;
    else;
      quadrant = 2;
    endif;
    if y>MID_Y;
      quadrant += 2;
    endif;
    map(quadrant) += 1;
  endif;

  // Read next line
  exec sql fetch c1 into :data;
enddo;

exec sql close c1;

result = map(1)*map(2)*map(3)*map(4);

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

Contrary to the usual precision of the puzzle, part 2 was extremely vague and the puzzle quite difficult to code because we don't know exactly what we're looking for.

The solution I ended up using is drawing each line of the map at every second from second 1 to 10000 and look for an horizontal line of 17 robots, it happened to work.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-c WIDTH 101;
dcl-c HEIGHT 103;
dcl-c MID_X 50;
dcl-c MID_Y 51;
dcl-c LENGTH 10403;
dcl-c MIN_TIME 0;
dcl-c MAX_TIME 10000;

dcl-s fileName varchar(100);

dcl-s data varchar(255);
dcl-s x int(3);
dcl-s y int(3);
dcl-s vx int(3);
dcl-s vy int(3);
dcl-s time int(5);

dcl-s result int(20) inz(0);

dcl-s map uns(5) dim(LENGTH); // We store the number of robots per position

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select line from table(qsys2.ifs_read_utf8(path_name => :fileName));

for time = MIN_TIME to MAX_TIME;
reset map;
exec sql open c1;

// Read first line
exec sql fetch c1 into :data;
  dow sqlcode = 0;
  // Read initial position and speed
  exec sql set :x = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 1));
  exec sql set :y = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 2));
  exec sql set :vx = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 3));
  exec sql set :vy = int(regexp_substr(:data, 'p=(\d+),(\d+) v=(-?\d+),(-?\d+)', 1, 1, '', 4));

  // Compute final position
  x = %rem(x+vx*TIME:WIDTH);
  y = %rem(y+vy*TIME:HEIGHT);

  // Adjust x and y if they are negative
  if x < 0;
    x += WIDTH;
  endif;
  if y < 0;
    y += HEIGHT;
  endif;

  map(x+y*WIDTH+1) += 1;
  
  // Read next line
  exec sql fetch c1 into :data;
  enddo;

exec sql close c1;

for y = 0 to HEIGHT-1;
  data = '';
  for x = 0 to WIDTH -1;
    if map(x+y*WIDTH+1) > 0;
      data += '#';
    else;
      data += '.';
    endif;
  endfor;
  exec sql set :result = regexp_count(:data, '#################');
  if result > 0;
    snd-msg *info 'Time: ' + %char(time) + 's' %target(*pgmbdy:1); // Send message with answer
  endif;
endfor;
endfor;

*inlr = *on;
return;</pre>