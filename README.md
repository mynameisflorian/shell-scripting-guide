# shell-scripting-guide

1) Whenever possible, **use dash instead of bash**
    * Dash is faster
    * Dash is less complicated
    * Dash is less error-prone
    * Dash encourages "proper" use of shell scripts (see below)
    
2) **Always quote variables**, unless variable expansion is specifically desired.
    * If possible, break such use into it's own function and document quoting exception.
    * There are many, minor, exceptions to when variables do not need to be quoted, but in all cases unneeded quoting does not cause problems; and it's much easier to remember to always quote than it is to remember all the exceptions

3) **Avoid global variables**. 
    * Use `readonly` global constants with a prefix to avoid collisions
    * Once a global readonly variable is set, it cannot be unset or used in a local context

4) **Use "local" variables** in functions
    * Without the local specifier, any variables declared within a function are global variables.

5) **Validate application and function inputs**

6) **Avoid using eval**
    * If you must use it, use with extreme caution
    * See section below on safe use of eval

7) **Use `set -o nounset`**

8) When using **boolean variables** (true/false):
    * True: Use the value "true" 
    * False: Use the null value (ie "")
    * `[ "$boolean_value" ]`
    
9) Avoid combining `&&` and `||` into a single expression
    * evaluation can be unintuitive
    * `some_command || X && Y` will execute X if some_command fails, Y if some_command succeeds, and Y if X succeeds
    * `some_command && X || Y` will execute X if some_command succeeds, Y if some_command fails, and Y if X fails
    
10) Using `set -o errexit` without additional coding restrictions produces undesirable results
    * see section below for additional restrictions
    * otherwise, catch all errors manually:
        * `some_command blah blah || ERROR "...message..."`
      
11) To completely shut down a script with backgrounded child processes, use `trap 'kill -- -$$' EXIT`







## Processing Arguments
```
  while [ "$#" -gt 0 ]; do case "$1" in
    (--a)
      argument__a="true"
      ;;
    (--name=*)
      argument__name="${1#--name=}"
      ;;
    (""|--help)
      print_usage
      exit 0
      ;;
    (*)
      echo "Error: Unknown argument '$1'"
      exit 1
      ;;
  esac; shift; done
  readonly argument__a argument__name
```



# Input Validation

### is_digits()
is_digits(){ [ "${1##*[!0-9]*}" ]; }

### is_variable_name()
is_variable_name(){ [ "${1##[0-9]*|*[!a-zA-Z0-9_]*}" ]; }







# Safe(er) use of Eval
### ref(): Variable Reference Getter
```
# Usage: ref <variable-name>
# Example: some_command $(ref "$1")
ref(){
  [ "$#" = 1 ] || ERROR "ref(): requires exactly one argument"
  case "$1" in ([0-9]*|*[!a-zA-z0-9_]*) ERROR "ref(): invalid variable name";; esac
  eval echo "\$$1"
}
```
### set_ref(): Variable Reference Setter
```
# Usage: set_ref <variable-name> <value>
# Description: validates <variable-name> and sets variable by wrapping <value> in single quotes, replacing any single quotes in <value> with <'"'"'> 
set_ref(){
  [ "$#" = 2 ] || ERROR "set_ref(): requires exactly 2 arguments"
  case "$1" in ([0-9]*|*[!a-zA-z0-9_]*) ERROR "set_ref(): invalid variable name";; esac
  eval "$1='$(echo "$2"|sed "s/'/'\"'\"'/g")'"
}
```
note: 
* single-quotes strings are completely literal (ie no character has any special meaning)
* two strings directly adjacent to each other (w/o any whitespace) are concated (combined) into a single string
* the sed statement replaces any single quotes(`'`) with a single-quote-escaping-sequence(`'"'"'`)
* `'"'"'` evaluates as:
    * exit singly-quoted string
    * enter doubly-quoted string
    * supply a single-quote character to the string
    * exit doubly quoted string
    * re-enter singly-quoted string






# Working with filenames

Working with files

* Filenames can contain any character except null
* Filenames are easily mangled when sent through a stream/pipe or stored in a file
* The safest place to keep a filename is in a variable
* To escape a filename, use:
   * ```/usr/bin/printf "%q" "$filename"```
* to unescape a filename use:
   * ```somecommand "$(eval /usr/bin/printf "%s" "$escaped_filename")"```
   * DOES NOT WORK IN DASH -- TODO DOCUMENT WORKAROUND
* currently newlines do not correctly unescape
* Verification test:
```
for f in *; do
   fn="$(/usr/bin/printf "%q" "$f")"
   fnf="$(eval echo $fn)"
   [ "$f" = "$fnf" ] \
      || printf "Fail: %s != %s\n" "$f" "$fnf"
done
```
(tested in bash and dash, dash fails to unescape on filenames with embedded newlines


        


# Shell Script Documentation
* https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html
* man dash





# Notes on errexit

"set -o errexit" should be used with caution. 

Use of functions in these contexts "inhibits" errexit within the function:
* as conditionals in "if" statements
* in all but the last position of a short-circuit expression (ie && and ||)

Recommended rules if using errexit:
* do not use functions as conditionals in "if" statements (unless function meant specifically to be used this way)
* do not use "short circuit" expressions on functions (ex some_function || do_something)
* all functions should use explicit return statements (see gotchya)

### Gotchya #1
* functions will return the status of the last command, even if that command is inhibited
```
      f(){ false && true; }
      f                   #--> errexit
```

* "&& true" inhibits "false" from triggering errexit, but the function still uses the return status of false for it's retutn status, 
triggering an errexit





# Miscellaneous shell tricks

### Execute if variable null or unset
```        
    ${VAR:+:} ERROR "VAR IS UNSET"
    # ${...} evaluates to ":" when null or unset (:+)
   # ${VAR+:} would only exec when unset
```



### Multi-line comments

```
   alias '[NOTES]=:<<"[/NOTES]"'
   
   [NOTES]
      Blah blah blah
      .
      .
      .
      Blah!
   [/NOTES]
```
