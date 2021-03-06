# asp

Interpreter for a tiny functional programming language.

I wrote this back in 1986, I found the code kicking around, I thought it would
be fun to upload somewhere as a curiosity. It compiles, no idea if it works.

It was an experiment with an implementation strategy for string reduction. The
language I mostly used back then was Miranda. It used Turner's Combinators
for evaulation and I wondered if direct string reduction would be faster. 

asp evaluates lambdas like this:

	lambda x: x + 12 * x

compiles as:

	lambda A: A->B + 12 * B

ie. when instantiating the lambda, we write the pointer into A, then follow
the old value of the pointer to write into B. This way you can set all
instances of a bound variable in a single step, rather than using combinators
to percolate values slowly downwards through the tree. 

This will only work if the expression is unshared. Therefore asp keeps
reference counts of objects, and reinstantiates afresh if a scrap of graph is
shared.

To make instantiation quick, asp has a heap with variable-sized objects and
does an instatiation with a straight memcpy(). Actually, there are two heaps
and it does GC by simmple compaction.

asp uses relative pointers to make memcpy() of tree fragments poossible. There
are absolute pointers too. 

It ended up being about 5x faster than Miranda, which pleased me. Though I
actually went with plain combinators for the next "compiler" I did. 

# Original notes


## Asp - simple applicative language.

Notes for version 1.0

```
Usage: 
	asp [-w - -c -h<n> -p<n> -a<n> -g<n> -f<n>] [filename filename ...]

Flags:	
	-w	Supresses warnings.
	-	Supress 'Starting evaluation' etc. messages.
	-c	Turn on counting - statistics are given for various things. 
		Only useful for benchmarking.
	-O	Turn on an optimiser ... this attempts to detect and remove 
		common subexpressions, for a 10-20% speed improvement. Compile 
		time goes up a bit.
	-h<n>	Set heap size. Default 60000 nodes.
	-p<n>	Set parse heap size. Default 20000.
	-a<n>	Set depth of application stack. Default 500.
	-g<n>	Set depth of C pointer stack. Default 1000.
	-f<n>	Set depth of fence stack. Default 100.
```

If no filenames are given in the command line, asp reads from stdin.  After
parsing all the files it generates a graph and starts reduction from 'main'.

Run with (eg):

	asp -w prelude.a myprog.a

prelude.a contains an Asp version of the Miranda standard prelude.

### Asp primitives

```
#include "pathname"	
		- Add file to list of files to be parsed.
[], ""		- The empty list.
:		- Infix cons.
hd, tl		- Head and tail of lists.
[a,b,c]		- List shorthand; same as a:b:c:[].
[1..]		- The infinite list [1,2,3,4,5,..].
[1,3..]		- The infinite list [1,3,5,7,9,..]. Can go down too.
[1..10]		- The list [1,2,3,4,5,6,7,8,9,10].
[1,3..11]	- The list [1,3,5,7,9,11]. As with [1,3..] can count up or 
		  down.
'a'		- char.
code, decode	- char -> num and num -> char.
"Hello world\n"	- List of char. Most of the C escape sequences are allowed.
error <expr>	- Stop with an error message on stderr. Look out for loops in 
		  the error expression!
.		- Function composition.
+,-,*,/		- Infix arithmetic ops. '-' can also be prefix.
true,false	- Boolean truth values.
and,or		- Infix logical ops.
~		- Not.
<>,=,<,>,>=,<=	- Infix relational ops.
read "pathname"	- Return the file as a list. Eg. read "/dev/tty" returns the 
		  list of chars typed at the terminal.
"name" write <expr>
		- The result of evaluating the expr is written to the file. 
		  Always returns true. Sorry it's infix.
```

All the usual precedences and associativities are followed.

### Asp syntax

Function definitions look like:

	fac n	= 1, n<2;
		= n * fac (n-1);

Write the function name, names for arguments and a list of possible right hand
sides qualified by guard expressions. During evaluation, each of the guards is
evaluated in turn (from the top). The result of the function is the expression
corresponding to the first guard to evaluate to true. The last rhs can have no
guard and is the default.

Everything following a % up to end of line is ignored.  Local definitions are
enclosed in curly braces following the function. For example:

	% Insert sort a list of num or char.
	sort list 
		= [], list=[];
		= insert (hd list) (sort (tl list));
		{	insert x list 
				= [x], list = [];
				= x:list, x <= a;
				= a:(insert x rest);
				{	a = hd list;
					rest = tl list;
				}
		}

Local definitions are slightly slower that global definitions.

### Bugs and problems

It would be nice if it were not necessary to type in all those )%!"#$ semi-
colons. Debugging is difficult without an interactive 'suck it and see' mode.
It would be nice if there was a type system! Local recursion is broken .. 
it makes the GC leak. It would be nice if hd/tl/+ etc. were functions rather
than built in operators.
