# Extended Bash
_This Project aims to collate tips, tricks and little functions that make working with bash easier or extend its functionality._
_As this Project is closely related to [Functional Bash](https://github.com/Ozymandias42/Functional-Bash) it can be regarded as implementation example._

## First trick: Working with Arrays
One of the big weaknesses of Bash is its inability to return arrays from functions.  
Let's have a look at the three big problems.
1. Passing strings with quotes as singular argument to functions
2. returning arrays of strings with quotes from functions
3. passing multiple arrays to functions

First the short answers to these problems, explanations will follow.

1. Use Bash expansion-escape syntax `$""` or `"${param@Q}`
2. Use name-references via `declare -n` or declaration conversion sytax via `${param@A}`
3. Use of indirection via `func arr[@]`-call and `local -a arr=(${!1})` inside of functions

---
### Passing whitespace separated strings as singular argument
Let's look at the first problem.
```bash
array=("bob \"speaks\" to 'alice'" "eve listens in")
bob(){ echo $1 ; }
bob ${array[0]}   # => bob || expected: "bob "speaks" to 'alice'"
bob "${array[0]}" # => bob "speaks" to 'alice' || meets the expectation.
```
Now why does the quoted line work and the other does not?  
As arguments in Bash are whitespace separated, quoting an array expansion effectively treats it as singular argument instead of defaulting to the whitespace as separator.  
**But wait** what if I need the quotes in that output string explicitly escaped?  
Now this is where `${param@Q}` comes in. (Think of 'Q' as Quasiquote)

```bash
echo ${array[@]@Q} # => 'bob "speaks" to '\''alice'\''' 'eve listens in'
```
See: [GNU Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html#Shell-Parameter-Expansion)

_Okay!_ now when and why to use `$""` what does the dollar sign mean?
From `man bash`  
>_"A double-quoted string preceded by a dollar sign ($"string") will cause the string to be  translated  according  to the current locale.  If the current locale is C or POSIX, the dollar sign is ignored.  If the
string is translated and replaced, the replacement is double-quoted."_

When is this useful? If you try to do i18n in your shell script. 

---
### Returning arrays _from_ functions in general.

Returning arrays from function can be achieved through use of Brace Expansion syntax `${}`.  
To return an array from a function it needs to be expanded into it's elements and those need to be returned as string via `echo`  

```bash
array=("bob \"speaks\" to 'alice'" "eve listens in")
echo ${#array{@}} # 2 => number of elements in array.
echo ${array[@]}  # "bob "speaks" to 'alice'" "eve listens in"
```
Trying to put that back into an array by wrapping `()` around it results in an array with 7 elements instead of 2 like it originally was.
```bash
newarr=( $( echo ${array[@]} ) )
echo ${#newarr[@]} # 7 => expected 2
newarr=( $( echo "${array[@]}" ) ) #quoted
echo ${#newarr[@]} # 7 => expected 2, same result.
```

Now there is a way around this. If arguments with escapable characters like single- or doublequotes need to be passed to functions and you need to make sure they are treated as one argument instead of multiple this can be achieved via the `Q` operator in the brace expansion syntax.  

```bash
echo ${#newarr[@]@Q}
'alice '\''eve'\'''
```

Unfortunately this does not work for re-creating an array consisting of strings with such characters.
```bash
array=("bob \"speaks\" to 'alice'" "eve listens in")
echo ${#array{@}}
2 # => number of elements in array.
newarr=($(echo $"${array[@]@Q}"))
echo ${#newarr[@]}
7 # => expected 2
```

---
### Returning arrays with strings with whitespaces from functions
There are two ways around it. 
1. Using named references via the `-n` parameter of `declare` and `local`
2. Using Brace expansion to `declare -p var_name` form via `${array[@]@A}`

#### Named References
Let's say we have a function called bob and we want to have it manipulate strings in array fields.
As we have seen we cannot simply pass the array, do some transformations and return it. We can hower pass the _name_ of the array and re-define a `local` array inside of the function that acts as a name reference to the outside original array.
This means, that mutating the nameref's array's state inside of the function mutates it in the original outside too.
You can think of it as pass by reference rather than pass by value if you're coming from C.
Example:
```bash
bob(){
  local -n arr=$1
  #do sth on arr.
  #notice, no echo or return!
}
outside_array=("complex string with whitespace" "string with 'quotes'")
bob outside_array
```

#### (Re-)declaring the undying array!
To illustrate what `declare -p` or `${array[@]@A}` do respectively let's start with an example.
```bash
# First step  create a complex array
declare -a array=("bob \"speaks\" to 'alice'" "eve listens in")
# Second step run declare -p
declare -p array
# => declare -a array=([0]="bob \"speaks\" to 'alice'" [1]="eve listens in")
# Brace expansion with the @A parameter does the same
echo ${array[@]@A}
# => declare -a array=([0]="bob \"speaks\" to 'alice'" [1]="eve listens in")
```

Okay, cool but how do we use this?  
Well easy the answer is the infamous `eval` !  
As we can see `declare -p varname` prints out a (properly escaped) declaration statement for variable of the given name.  
Returning this via an echo statement and running it through eval redeclares that variable.

```bash
testDeclareP(){
  local -a array=("bob \"speaks\" to 'alice'" "eve listens in")
  echo "${array[@]@A}"
}
eval "$(testDeclareP)"
echo ${#array[@]} # => 2 || meets expectation!
```

### Passing multiple arrays to functions
How are arrays passed to functions at all?
```bash
declare -a array=("bob" "alice")
hello(){ echo "$@" ; } # $@ == $1 $2 ... $n
hello $array           # => bob       || $array == ${array[0]}
hello ${array[@]}      # => bob alice || each item passed as parameter.
```
Okay, now what happens if we pass two arrays via expansion?
```bash
declare -a array1=("bob" "alice")
declare -a array2=("eve" "steve")
hello ${array1[@]} ${array2[@]} # same as hello "bob" "alice" "eve" "steve"
```
We can already see, that unless we know the size of both arrays inside the function we will not be able to tell where the arguments of one array end and those of the other begin. What we need is a way to give the function to parameters and tell it that those are arrays.

**Wait!** one might shout, _didn't we already do that with `hello array1` and doing `local -n array=($1)` inside the function?_

Well yes, we did but there's a difference here. With `-n` we basically introduce a second named pointer to the values behind the original variable name behind `$1`

With indirection on the other hand we use the string we supplied to the function as parameter as name of a variable to whose to expand
this means: `funcname "arrayName[@]"` with `local -a arr=({!1})` expands `arrayName[@]` inside of the function into arr. This way we can supply multiple arrays into a single function without having to worry about their size.  
**Note** however, doing `local -a arr=(${!1})` and `local -a arr=("${!1}")` yields diefferent results!  
Example:
```bash
declare -a array=("bob \"speaks\" to 'alice'" "eve listens in")
testFunc(){ local -a arr=(${!1}) ; echo ${arr[@]@A} ; }
testFunc array[@]
# => declare -a arr=([0]="bob" [1]="\"speaks\"" [2]="to" [3]="'alice'" [4]="eve" [5]="listens" [6]="in")
# vs.
testFunc(){ local -a arr=("${!1}") ; echo ${arr[@]@A} ; }
testFunc array[@]
# => declare -a arr=([0]="bob \"speaks\" to 'alice'" [1]="eve listens in")

```