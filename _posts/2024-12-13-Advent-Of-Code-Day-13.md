---
layout: post
title:  "Advent of Code 2024 day 13, RPG edition"
date:   2024-12-13 19:30:00 -0400
categories: adventofcode rpg
---

This is the thirteenth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/13).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing configurations for a claw machine.

The claw machine has 2 buttons: A and B. Each button moves a given quantity (defined in the input) on X and Y axis. To win, we need to press A and B buttons a given number of times each to reach a given X and Y position (defined in the input).

It's not possible to win all configurations.

Pressing A button costs 3 tokens, pressing B button costs 1 token.

The goal is to calculate how many token are need to win all winnable configurations.

The puzzle description seem to encourage a brute force method but let's just math it out!

A solution can be represented as 2 equations with two unknown:
Ax + By = C
Dx + Ey = F

Where:
* x is the number of A button press
* y is the number of B button press
* A is the X offset of A button
* B is the X offset of B button
* C is the target X coordinate
* D is the Y offset of A button
* E is the Y offset of B button
* F is the target Y coordinate

If you don't feel like solving those equations, you can ask [Wolfram](https://www.wolframalpha.com/input?i=solve+x*A%2By*B%3DC%3Bx*D%2By*E%3DF).

The solution is:
x=(CE-BF)/(AE-BD) and y=(CD-AF)/(BD-AE)

We simply calculate the value of x and y. To be a solution, x and y must be positive (or null) whole number. To avoid a false solution because of a rounding error, we verify the x and y values actually work.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-s fileName varchar(50);

dcl-s data varchar(255);
dcl-s val varchar(10);

dcl-s result int(20) inz(0);

dcl-s ax int(5);
dcl-s ay int(5);
dcl-s bx int(5);
dcl-s by int(5);
dcl-s x int(20);
dcl-s y int(20);
dcl-s a packed(20:5);
dcl-s b packed(20:5);

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select line from table(qsys2.ifs_read_utf8(path_name => :fileName));
exec sql open c1;

// Read first line
exec sql fetch c1 into :data;

dow sqlcode = 0;
  // extract ax
  exec sql set :val = regexp_substr(:data, 'Button A: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 1);
  ax = %int(val);
  // extract ay
  exec sql set :val = regexp_substr(:data, 'Button A: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 2);
  ay = %int(val);

  // Read second line
  exec sql fetch c1 into :data;
  // extract bx
  exec sql set :val = regexp_substr(:data, 'Button B: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 1);
  bx = %int(val);
  // extract by
  exec sql set :val = regexp_substr(:data, 'Button B: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 2);
  by = %int(val);
 
  // Read third line
  exec sql fetch c1 into :data;
  // extract x
  exec sql set :val = regexp_substr(:data, 'Prize: X=(\d+), Y=(\d+)', 1, 1, 'c', 1);
  x = %int(val);
  // extract y
  exec sql set :val = regexp_substr(:data, 'Prize: X=(\d+), Y=(\d+)', 1, 1, 'c', 2);
  y = %int(val);

  a = (x*by-bx*y)/(ax*by-bx*ay);
  b = (x*ay-ax*y)/(-ax*by+bx*ay);

  if a>=0 and b>=0 and %int(a) = a and %int(b) = b;
    if a*ax+b*bx = x and a*ay+b*by = y;
      result += %int(a)*3 + %int(B);
    endif;
  endif;
 
  // Read next 2 lines
  exec sql fetch c1 into :data;
  exec sql fetch c1 into :data;
enddo;

exec sql close c1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>

# Part 2

In part 2, we add a big number to the target coordinates.

If had used a brute force method in part 1, we would be in trouble. Since we went the math way, we just have to change the target coordinates.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-s fileName varchar(50);

dcl-s data varchar(255);
dcl-s val varchar(10);

dcl-s result int(20) inz(0);

dcl-s ax int(5);
dcl-s ay int(5);
dcl-s bx int(5);
dcl-s by int(5);
dcl-s x int(20);
dcl-s y int(20);
dcl-s a packed(20:5);
dcl-s b packed(20:5);

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select line from table(qsys2.ifs_read_utf8(path_name => :fileName));
exec sql open c1;

// Read first line
exec sql fetch c1 into :data;

dow sqlcode = 0;
  // extract ax
  exec sql set :val = regexp_substr(:data, 'Button A: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 1);
  ax = %int(val);
  // extract ay
  exec sql set :val = regexp_substr(:data, 'Button A: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 2);
  ay = %int(val);

  // Read second line
  exec sql fetch c1 into :data;
  // extract bx
  exec sql set :val = regexp_substr(:data, 'Button B: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 1);
  bx = %int(val);
  // extract by
  exec sql set :val = regexp_substr(:data, 'Button B: X\+(\d+), Y\+(\d+)', 1, 1, 'c', 2);
  by = %int(val);
 
  // Read third line
  exec sql fetch c1 into :data;
  // extract x
  exec sql set :val = regexp_substr(:data, 'Prize: X=(\d+), Y=(\d+)', 1, 1, 'c', 1);
  x = %int(val)+10000000000000;
  // extract y
  exec sql set :val = regexp_substr(:data, 'Prize: X=(\d+), Y=(\d+)', 1, 1, 'c', 2);
  y = %int(val)+10000000000000;

  a = (x*by-bx*y)/(ax*by-bx*ay);
  b = (x*ay-ax*y)/(-ax*by+bx*ay);

  if a>=0 and b>=0 and %int(a) = a and %int(b) = b;
    if a*ax+b*bx = x and a*ay+b*by = y;
      result += %int(a)*3 + %int(B);
    endif;
  endif;
 
  // Read next 2 lines
  exec sql fetch c1 into :data;
  exec sql fetch c1 into :data;
enddo;

exec sql close c1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;</pre>