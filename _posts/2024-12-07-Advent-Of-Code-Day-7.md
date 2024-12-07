---
layout: post
title:  "Advent of Code 2024 day 7, RPG edition"
date:   2024-12-07 09:30:00 -0400
categories: adventofcode rpg
---

This is the seventh day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/7).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a serie of equations with the operators missing. The operators can be plus or multiply, operations are done from left to right, there is no operator precedence.

The goal of part one is to calculate the sum of the equations that can be made true by adding operators.

As usual, we read the input file with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

We create the solve procedure that try to solve the equation by using either add or multiply for the two leftmost values and recursively calls itself with the rest of the values until no more values are left. It returns 0 if the current combination of operators doesn't solve the equation or the result if it does.

We use dynamic arrays to store the values. Problem is you can't pass dynamic (or even varying length) arrays to procedures so we do it the old fashion way by passing a fixed length array and a number of elements.

Since solve returns 0 if the equation can't be solved, we can simply sum the result of solve.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr solve int(20);
  target int(20) value;
  left int(20) value;
  right int(5) dim(30) value;
  rightElems int(5) value;
end-pr;

dcl-s fileName varchar(50);

dcl-s result int(20) inz(0);
dcl-s line varchar(1000);
dcl-s sum int(20);
dcl-s values int(5) dim(*auto:30);
dcl-s value varchar(5);

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select cast(line as varchar(1000)) from table(qsys2.ifs_read_utf8(path_name => :fileName));
exec sql open c1;

// Read the first line
exec sql fetch from c1 into :line;

dow sqlcode = 0;
// read the sum
sum = %int(%left(line:%scan(':':line)-1));

// read the terms
%elem(values) = 0;
for-each value in %split(%subst(line:%scan(':':line)+1));
  values(*next) = %int(value);
endfor;

result += solve(sum:values(1):%subarr(values:2):%elem(values)-1);

// Read the next line
exec sql fetch from c1 into :line;
enddo;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc solve;
dcl-pi *n int(20);
  target int(20) value;
  left int(20) value;
  right int(5) dim(30) value;
  rightElems int(5) value;
end-pi;

if rightElems = 1;
  if left+right(1) = target or left*right(1) = target;
    return target;
  else;
    return 0;
  endif;
else;
  if solve(target:left+right(1):%subarr(right:2:rightElems-1):rightElems-1) > 0 or solve(target:left*right(1):%subarr(right:2:rightElems-1):rightElems-1) > 0;
    return target;
  else;
    return 0;
  endif;
endif;

end-proc;</pre>

# Part 2

Part 2 is exactly the same as part 1 with an added operator: concatenation (||) which, as the name implies, concatenates numbers (12 || 95 = 1295).

We simply add a test for the concatenate operator to the program from part 1.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr solve int(20);
  target int(20) value;
  left int(20) value;
  right int(5) dim(30) value;
  rightElems int(5) value;
end-pr;

dcl-pr concatenate int(20);
  left int(20) value;
  right int(20) value;
end-pr;

dcl-s fileName varchar(50);

dcl-s result int(20) inz(0);
dcl-s line varchar(1000);
dcl-s sum int(20);
dcl-s values int(5) dim(*auto:30);
dcl-s value varchar(5);

fileName = %trim(input);

// Read the input data from the IFS
exec sql declare c1 cursor for select cast(line as varchar(1000)) from table(qsys2.ifs_read_utf8(path_name => :fileName));
exec sql open c1;

// Read the first line
exec sql fetch from c1 into :line;

dow sqlcode = 0;
// read the sum
sum = %int(%left(line:%scan(':':line)-1));

// read the terms
%elem(values) = 0;
for-each value in %split(%subst(line:%scan(':':line)+1));
  values(*next) = %int(value);
endfor;

result += solve(sum:values(1):%subarr(values:2):%elem(values)-1);

// Read the next line
exec sql fetch from c1 into :line;
enddo;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc solve;
dcl-pi *n int(20);
  target int(20) value;
  left int(20) value;
  right int(5) dim(30) value;
  rightElems int(5) value;
end-pi;

if rightElems = 1;
  if left+right(1) = target or left*right(1) = target or concatenate(left:right(1)) = target;
    return target;
  else;
    return 0;
  endif;
else;
  if solve(target:left+right(1):%subarr(right:2:rightElems-1):rightElems-1) > 0 or solve(target:left*right(1):%subarr(right:2:rightElems-1):rightElems-1) > 0
     or solve(target:concatenate(left:right(1)):%subarr(right:2:rightElems-1):rightElems-1) > 0;
    return target;
  else;
    return 0;
  endif;
endif;

end-proc;


dcl-proc concatenate;
dcl-pi concatenate int(20);
  left int(20) value;
  right int(20) value;
end-pi;

return %int(%char(left)+%char(right));

end-proc;</pre>