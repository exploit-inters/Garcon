This document explains the structure of the source code, how to use this program and key ideas behind the program.

## dependencies
- [php-parser](https://github.com/nikic/PHP-Parser)

To install PHP-Parser:
```bash
$ php composer.phar require nikic/php-parser
```

## usage
```bash
$ ./main.php < input/1.php
```
to suppress debug information:
```bash
$ ./traverse.php < input/1.php 2> /dev/null
```

## demo
input
```php
<?php
if(input_from_params()) {
    $a = $_GET[0];
} else {
    $a = mysql_fetch_row();
}
print_($a);

```
output
```
There is a Persisted XSS vulnerability at line 7
When the taint(source) conditions is satisfied:
input_from_params() == false
There is a Command line injection vulnerability at line 8
When the taint(source) conditions is satisfied:
input_from_params() == true
```

## structure
- main.php: traverses the AST and perform all the analysis on it
- symbolTable.php: implements SymbolTable class, ArrayTable class and ClassTable class
- taintInfo.php: implements TaintInfo class, SingleTaint class and SingleSanitize class
- condition.php: implements the Condition class and CompoundCondition class, which are the cornerstone of the analyzer

Besides, test files are in input/, demo files are in demo/. All other files are either for test or composer's file.

## classes
#### SymbolTable
SymbolTable class is a collection of three symbol tables, namely string symbol table, array symbol table and object symbol table. String symbol table is implemented as a simple associate array. Because array can fetch index and object can fetch property so they should be implemented as a two dimensional array. In this implementation, the second level array is implemented as an object which contains an array and other relevant methods. There is no integer array or boolean array because they cannot be tainted and thus are never in any symbol table. All the tables use the same algorithm to calculate the taint condition.

#### ArrayTable
The tricky thing that array table has to implement is situation as blow
```php
$a[0] = "tainted";
$a[foo()] = "tainted";
execute($a[0]);
execute($a[1]);
```
The analyzer should be able to point out `foo()==1` is the taint condition for $a[1] and $a[0] must be tainted. The way this is implemented is that there is an invisible index in the array. If the array uses an expression to index, suppose it is working on this invisible index. For instance, after `$a[foo()] = "tainted";`, the invisible index will be appended a new condition `old condition AND (foo() == "pending")`, suppose the invisible index is called "pending". Also, when an uncertain index was untainted, still operate on the invisible index.

#### ClassTable
This class should've be called ObjectTable, but somehow when I wrote the initial code I used word class whenever I wanted to say object. I think it is because I was working with hotspot klass at that point, so I was too used to the word class. ClassTable is basically the same as a simple array and also implements the condition calculation algorithm.

#### TaintInfo
TaintInfo is an extension to support of more than one type of taint. And it also keeps track of sanitize information, which is used when a vulnerability check happens. A lot of vital operations are also implemented in this class, such as merging two taints.

#### SingleTaint
A TaintInfo object contains a list of SingleTaint object and a list of SingleSanitize object. A SingleTaint object represent one taint type, its taint condition and other manipulation of taint condition. A condition can either be a Condition object or a CompoundCondition object.

#### SingleSanitize
Same as SingleTaint.

#### Condition
A implementation of one single contion such as `$a==true`. Include basic operations on a condition such as "setAlwaysTrue", "setNot" and printing.

#### CompoundCondition
Represent a condition logic expression such as `$b==true AND $a!=true`. A CompoundCondition object has a left condition, right condition and operator. Both left condition and right condition can be either a Condition obejct or another CompoundCondition object. In this way, conditions are concatenated.

### Defect
One problem I was struggling with was that I wanted to use PHP's obejct assignment feature to implemented the object assignment problem. The basic idea is that when copy an array table to the inner symbol table, do a deep copy, and do a shallow copy for class table. And shallow copy is default for object assignment.
```php
<?php
function modify0($a) {
    $a = "safe";
}

function modify1($a) {
    $a[0] = "safe";
}

function modify2($a) {
    // if this is uncommented, $c should not be tainted after going through the function call 
    // $a = new A(); 
    $a->foo = "foo";
}

$a = $_GET['u'];
$b[0] = $_GET['u'];
$c->foo = $_GET['u'];
modify0($a);
modify1($b);
modify2($c); // should be tainted
system($a);
system($b[0]);
```
My thoery should be right, but what happened was even if `// $a = new A();` is uncommented, `$c ->foo` will still be changed.

Besides this serious problem, other trivial things are that only core functionality was implemented and the program is not complete. Taint propagation in simple situations, such as string concatenation, is left unfinished.