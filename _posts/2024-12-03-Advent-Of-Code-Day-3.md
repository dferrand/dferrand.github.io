---
layout: post
title:  "Advent of Code 2024 day 3, RPG edition"
date:   2024-12-03 08:35:00 -0400
categories: adventofcode rpg
---

This is the third day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/3).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing a corrupted program. We need to find valid instructions in the form mul(X,Y) where X and Y are 1 to 3 digit numbers.

We need to find all valid instructions, calculate X*Y and sum the results.

The IFS file can easily be read using [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function.

Finding valid instructions can easily be done using [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) which can be used in RPG programs via SQL.

The regular expression to identify valid instructions is ```mul\((\d{1,3}),(\d{1,3})\)```. See [here](https://regex101.com/r/LFlIzS/2) for an explenation of this regular expression.

Each report can be devided into levels with the %SPLIT RPG built-in function. We can iterate over the levels extracted by %SPLIT with the for-each instruction.

We use the ```REGEXP_COUNT``` SQL function to check how many matches are found.

We then do a for loop to process each match. For each match, we use ```REGEXP_SUBSTR``` twice to extract the parameters of the mul() function. We the multiply those two values and add it to the result.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr processLine;
  line varchar(32000);
end-pr;

dcl-s fileName varchar(50);

dcl-s line varchar(32000);
dcl-s result int(10) inz(0);

fileName = %trim(input);

// Read the input data from the IFS one line at a time
exec sql declare c1 cursor for select cast(line as varchar(32000)) from table(qsys2.ifs_read_utf8(path_name => :fileName));

exec sql open c1;

// Read first line
exec sql fetch from c1 into :line;

// Loop until the end of the file
dow sqlcode = 0;

  processLine(line);

  // Read next line
  exec sql fetch from c1 into :line;
enddo;

exec sql close c1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc processLine;

dcl-pi *n;
  line varchar(32000);
end-pi;

dcl-s count int(10);        // Number of correct instructions found
dcl-s val1 int(10);         // First operand
dcl-s val2 int(10);         // Second operand
dcl-s index int(10);        // Index of the occurence

exec sql select regexp_count(:line, 'mul\((\d{1,3}),(\d{1,3})\)') into :count from sysibm.sysdummy1;

for index = 1 to count;

  exec sql select int(regexp_substr(:line, 'mul\((\d{1,3}),(\d{1,3})\)', 1, :index, 'c', 1)) into :val1 from sysibm.sysdummy1;
  exec sql select int(regexp_substr(:line, 'mul\((\d{1,3}),(\d{1,3})\)', 1, :index, 'c', 2)) into :val2 from sysibm.sysdummy1;

  result += val1 * val2;

endfor;

end-proc;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

We still need to calculate the sum of the mul() instructions but 2 new instructions must be handled:
* do() that enables future mul instructions
* don't() that disables future mul instructions

Only the lastest do() or don't() instruction applies. We consider that mul instructions are enabled at the begining of the program.

We introduce a global indicator variable inDo that contains *on if mul instructions are enabled and *off otherwise.

We use a slightly more complex regular expression to find the next valid instruction: ```mul\((\d{1,3}),(\d{1,3})\)|do\(\)|don't\(\)``` see [here](https://regex101.com/r/2kFkoY/1) for an explanation of this regular expression.

We use the ```REGEXP_INSTR``` SQL to return the position of the first valid instruction in the string. If no valid instruction is found, it returns 0.

We analyse the 3 characters at the returned position to determine which instruction was found with RPG select/when-is structure:
* if we have 'do(', it means it's a do() instruction, so we set inDo to *on
* if we have 'don', it means it's a don't() instruction, so we set inDo to *off
* if we have 'mul', it means it's a mul() instruction. If inDo is *off, we do nothing. If inDo is *on, we use the same regular expression as in part 1 to extract the parameters and multiply the values.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr processLine;
  line varchar(32000);
end-pr;

dcl-s fileName varchar(50);

dcl-s line varchar(32000);
dcl-s result int(10) inz(0);
dcl-s inDo ind inz(*on);

fileName = %trim(input);

// Read the input data from the IFS one line at a time
exec sql declare c1 cursor for select cast(line as varchar(32000)) from table(qsys2.ifs_read_utf8(path_name => :fileName));

exec sql open c1;

// Read first line
exec sql fetch from c1 into :line;

// Loop until the end of the file
dow sqlcode = 0;

  processLine(line);

  // Read next line
  exec sql fetch from c1 into :line;
enddo;

exec sql close c1;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc processLine;

dcl-pi *n;
  line varchar(32000);
end-pi;

dcl-s val1 int(10);         // First operand
dcl-s val2 int(10);         // Second operand
dcl-s pos int(10) inz(1);   // currentPosition
dcl-s nextPosition int(10); // next position

// search for the first do(), don't() or mul(xxx,xxx)
exec sql select regexp_instr(:line, 'mul\((\d{1,3}),(\d{1,3})\)|do\(\)|don''t\(\)', :pos) into :nextPosition from sysibm.sysdummy1;

dow nextPosition > 0;

  select %subst(line: nextPosition: 3);
    when-is 'do('; // If we found a do(), we set inDo to *on
      inDo = *on;
    when-is 'don'; // If we found a don't(), we set inDo to *off
      inDo = *off;
    when-is 'mul'; // If we found a mul() we process the multiplication only if inDo is set to *on
      if inDo;
        exec sql select int(regexp_substr(:line, 'mul\((\d{1,3}),(\d{1,3})\)', :nextPosition, 1, 'c', 1)) into :val1 from sysibm.sysdummy1;
        exec sql select int(regexp_substr(:line, 'mul\((\d{1,3}),(\d{1,3})\)', :nextPosition, 1, 'c', 2)) into :val2 from sysibm.sysdummy1;

        result += val1 * val2;
      endif;
  endsl;
  pos = nextPosition + 1;
// search for the next do(), don't() or mul(xxx,xxx)
exec sql select regexp_instr(:line, 'mul\((\d{1,3}),(\d{1,3})\)|do\(\)|don''t\(\)', :pos) into :nextPosition from sysibm.sysdummy1;
enddo;

end-proc;</pre>