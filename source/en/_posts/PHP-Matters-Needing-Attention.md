---
title: PHP Matters Needing Attention
date: 2016-10-20 11:08:57
tags:
  -  PHP
  -  Tips
categories:
  - PHP
description: A collection of things to keep in mind during everyday PHP development. They can, to a certain extent, make your programs run faster and more reliably.
lang: en
---

## Tips
1. #### Prefer static when possible
    If a function can be made static, prefer to make it static.
  The difference mainly shows up in memory handling: static methods allocate memory when the program starts, whereas instance methods allocate memory only when the object is instantiated.
  Static methods can be called directly, while instance methods require you to first create an instance and then call through it.

 > Too many static methods will consume memory.

1. #### echo VS print
   echo is faster than print, because echo has no return value while print returns an integer.
 > When echoing large strings, the corresponding configuration needs to be done on the server.

1. #### echo multiple strings
   When echo-ing multiple strings, use `,` instead of `.` to join them.

1.  #### @ error suppression
    Using @ to suppress errors slows down the script. In particular, avoid using @ inside loops.

1. #### $row['id'] & $row[id] & $row[1]
    `'id'` looks up the key `'id'` directly. Without quotes, a variable or constant is parsed by first determining its type and then resolving the value.

1. #### isset() & empty()
   isset() tests whether a variable has been assigned.
   empty() tests whether an already-assigned variable is empty. Referencing a variable that hasn't been assigned is allowed but produces a notice.
 > If a variable is assigned an empty value `$t = ""; $t = 0; $t = false;`, both `empty($t)` and `isset($t)` return true.
 > To destroy a variable, use `unset($t)` or `$t = NULL`.

1. #### Confirm the maximum count before looping
    Determine the maximum count before entering a for loop; don't recompute the maximum on every iteration.
    ```php
    // Don't do this
    for ($i=0;$i<=count($array);$i++){
    }

    // Do this instead
    $len = count($array);
    for ($i=0;$i<=$len;$i++){
    }
    ```
1. #### include() & include_once() & require() & require_once()
   All four load a file. The `_once` variants first check whether the file has already been loaded, to avoid loading it twice.
   The main differences are:
   * Error handling
      * `require()` throws a `fatal error` and halts the script if the file doesn't exist.
      * `include()` throws a `warning` but the script continues if the file doesn't exist.
   * Performance
     * `require()` processes the file only once (in effect, the file's content replaces the require() statement).
     * `include()` reads and evaluates the file on every execution.
     If code containing one of these directives may run multiple times, `require()` is more efficient.
   * Flexibility
     *  `require()` is usually placed at the very top of a script; the specified file is loaded before execution, becoming part of the script.
     *  `include()` is usually placed inside control-flow blocks. The file is loaded only when execution reaches it. This can simplify the runtime flow.

 > Use full paths when including files, to reduce the time spent resolving paths.

1. #### ' ' & " "
    PHP allows both single and double quotes to delimit string variables.
    `" "` first reads the string contents, then looks for variables inside it and interpolates their values.

1. #### Don't copy variables carelessly
    Copying one variable into another doubles the memory consumption.

1. #### if else & switch case
    A switch/case is better than a long chain of if/else if statements, and the code is easier to read and maintain.

1. #### Not everything has to be object-oriented
    Object orientation often carries significant overhead; every method call and object instantiation consumes memory.

1. #### Don't split methods too finely
    Every method call consumes memory.

1. #### Prefer PHP's built-in functions where possible.

1. #### Don't declare variables inside loops, especially large ones: objects.

1. #### Destroy variables to release memory, especially large arrays.

1. #### Use string functions instead of regular expressions.
1. #### split is faster than explode.
