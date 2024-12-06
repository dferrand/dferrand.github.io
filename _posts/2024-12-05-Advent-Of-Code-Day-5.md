---
layout: post
title:  "Advent of Code 2024 day 5, RPG edition"
date:   2024-12-05 20:55:00 -0400
categories: adventofcode rpg
---

This is the fifth day of [Advent of Code](https://adventofcode.com/2024/about), let's solve today's puzzles with RPG.

The puzzle as well as the input data are available [here](https://adventofcode.com/2024/day/5).

The code is available [here](https://github.com/dferrand/aoc2024).

# Part 1

We are provided with a (stream) file containing two data sets:
* page ordering rules: each line contains two page numbers separated by a |. The rule is the first page must appear before the second one in updates.
* updates: each line represents an update, it consists of a variable odd number of pages separated by comas.

We need to identify all the correctly-ordered updates (updates that observe all the rules) and add the page number of the middle page of those orders.

We load the rules using [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function. We store them in a dynamic array of ```CHAR(4)```, each rule is two 2-byte numbers (the predecessor and the successor page numbers) rather than in a data structure array because we can't lookup a full data scructure in a data structure array. We sort the array (with SQL) so that RPG can use a binary search which is faster than a full scan search.

We then read the updates with [QSYS2.IFS_READ_UTF8](https://www.ibm.com/docs/en/i/7.5?topic=is-ifs-read-ifs-read-binary-ifs-read-utf8-table-functions) table function again. We call the processLine procedure with the coma separated page list. processLine returns 0 if the update is incorrectly-ordered, the middle page number otherwise. So we can safely add the return value of processLine to the result.

We turn the coma separated list into an array with the ```%SPLIT``` built in function. We perform a first loop over each page in the list except the last one. For each page, we perform a second loop over all the following pages of the update. For each pair, we look for a rule that would be broken, since we're not interesting in locating the rule, we use the ```IN``` operator instead of the the ```%LOOKUP``` built in function. If a rule is found to be broken, we return 0 immediately.

If we didn't find any broken rule, we return the page number of the middle page.

Here is the RPG code for part 1:
<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

dcl-pr processLine int(10);
  line varchar(1000);
end-pr;

dcl-s fileName varchar(50);

dcl-s line varchar(1000);
dcl-s result int(10) inz(0);

dcl-ds rule qualified;     // A rule is a pair of page numbers, the predecessor and the successor
  predecessor int(5);
  successor int(5);
end-ds;

// We store rules in a dynamic array
// We define the array elements as 4 bytes instead of a data structure because RPG 
// can only search over one field of a data structure, not the whole DS.
// We sort the array to use binary search which is faster
dcl-s rules char(4) ascend dim(*auto:2000);     

fileName = %trim(input);

// Read the rules from the IFS
exec sql declare c1 cursor for select int(left(line, position('|', line)-1)) a, 
int(substr(line, position('|', line)+1)) b from table(qsys2.ifs_read_utf8(path_name => :fileName)) 
where line like '%|%' order by a, b;

exec sql open c1;

exec sql fetch c1 into :rule;
dow sqlcode = 0;

  rules(*next) = rule;

exec sql fetch c1 into :rule;
enddo;

exec sql close c1;

// Read updates from the IFS
exec sql declare c2 cursor for select cast(line as varchar(1000)) from table(qsys2.ifs_read_utf8(path_name => :fileName)) 
where line like '%,%';

exec sql open c2;

exec sql fetch from c2 into :line;

dow sqlcode = 0;

  result += processLine(line);     // processLine returns 0 for incorrectly ordered updates so we can safely add it to the result

exec sql fetch from c2 into :line;
enddo;

exec sql close c2;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc processLine;

dcl-pi *n int(10);
  line varchar(1000);
end-pi;

dcl-s pages char(2) dim(*auto: 1000);
dcl-s i int(5);
dcl-s j int(5);

pages = %split(line:',');

for i = 1 to %elem(pages) - 1;         // i loops over all pages until the second to last
  rule.successor = %int(pages(i));
  for j = i+1 to %elem(pages);         // j loops from page i to the last one
    rule.predecessor = %int(pages(j));
    if %char(rule) in rules;           // If page order breaks a rule, we return 0
      return 0;
    endif;
  endfor;
endfor;

return %int(pages(%int(%elem(pages)/2) + 1));  // If no rule was broken, we return the middle page number

end-proc;</pre>

# Part 2

In part 2, we'll use the same input as in part 1.

In this part, we want to find all the incorrectly-ordered updates, fix them by swaping the incorrectly ordered pages and return the sum of the middle page numbers.

We can acheive this by slightly modifying the processLine procedure: instead of returning 0 if we find a broken rule, we swap the incorrectly-ordered pages and recursively call processLine on the updated page list. Once the update is correctly ordered, we return the middle page number. To know if the update was correctly ordered in the begining and return 0, we add a second indicator parameter to processList, it has the value *off on the first call and *on on a recursive call.

When recursively calling processLine, we need to transform back an array into a coma separated list, for that, we use the ```%concatarr``` built-in function which is the opposite of ```%split```.

<pre>**free
ctl-opt dftactgrp(*no);  // We're using procedure so we can't be in the default activation group

dcl-pi *n;
  input char(50);
end-pi;

// In part 2, process line will be recursive but we don't want to return the middle
// page number if the update is correctly ordered. Therefore, we add the swapped
// parameter that will be *off on the initial call and *on in the recursive calls
// it will know wether to return 0 or the middle page
dcl-pr processLine int(10);   
  line varchar(1000) value;
  swapped ind value;        
end-pr;

dcl-s fileName varchar(50);

dcl-s line varchar(1000);
dcl-s result int(10) inz(0);

dcl-ds rule qualified;
  predecessor int(5);
  successor int(5);
end-ds;

dcl-s rules char(4) ascend dim(*auto:2000);

fileName = %trim(input);

// Read the rules from the IFS
exec sql declare c1 cursor for select int(left(line, position('|', line)-1)) a, 
int(substr(line, position('|', line)+1)) b from table(qsys2.ifs_read_utf8(path_name => :fileName)) 
where line like '%|%' order by a, b;

exec sql open c1;

exec sql fetch c1 into :rule;
dow sqlcode = 0;

  rules(*next) = rule;

exec sql fetch c1 into :rule;
enddo;

exec sql close c1;

// Read updates from the IFS
exec sql declare c2 cursor for select cast(line as varchar(1000)) from table(qsys2.ifs_read_utf8(path_name => :fileName)) 
where line like '%,%';

exec sql open c2;

exec sql fetch from c2 into :line;

dow sqlcode = 0;

  result += processLine(line:*off);

exec sql fetch from c2 into :line;
enddo;

exec sql close c2;

snd-msg *info 'Result: ' + %char(result) %target(*pgmbdy:1); // Send message with answer

*inlr = *on;
return;

dcl-proc processLine;

dcl-pi *n int(10);
  line varchar(1000) value;
  swapped ind value;
end-pi;

dcl-s pages char(2) dim(*auto: 1000);
dcl-s i int(5);
dcl-s j int(5);
dcl-s swap char(2);

pages = %split(line:',');

for i = 1 to %elem(pages) - 1;
  rule.successor = %int(pages(i));
  for j = i+1 to %elem(pages);
    rule.predecessor = %int(pages(j));
    if %char(rule) in rules;              // If pages(i) and pages(j) break a rule, we swap them and recursively call processLine until the update is correctly ordered
      swap = pages(i);
      pages(i) = pages(j);
      pages(j) = swap;
      return processLine(%concatarr(',' : pages):*on);
    endif;
  endfor;
endfor;
if swapped;
  return %int(pages(%int(%elem(pages)/2) + 1));
else;
  return 0;
endif;

end-proc;</pre>