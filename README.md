# shell-scripting-guide

### [13 Best-practices I follow](#13-best-practices-i-follow)

### [Script Loader](#script-loader)

### [Argument Processing](#argument-processing)

### [Input Validation](#input-validation)

### ['\n' is a lie](#n-is-a-lie)

### [Safe use of Eval](#safe-use-of-eval-1)

### [POSIX shell-only string replacement function](#)

### [Working with filenames](#)

### [Shell Script Documentation](#)


# 13 Best-practices I follow
1) Whenever possible, **use dash instead of bash**
    * Dash is faster
    * Dash is less complicated
    * Dash is less error-prone
    * Dash encourages better use of shell scripts (see below)
    
2) **Always quote variables**, unless variable expansion is specifically desired.
    * If possible, break such use into it's own function and document quoting exception.
    * There are many, minor, exceptions to when variables do not need to be quoted, but in all cases unneeded quoting does not cause problems; and it's much easier to remember to always quote than it is to remember all the exceptions

3) **Avoid global variables**. 
    * Use `readonly` global constants with a prefix to avoid collisions
      * Example: `readonly global_blah="meow"`
      * Example: `readonly path_to_configuration="whatever"`
    * Once a global readonly variable is set, it cannot be unset or used in a local context

4) **Use "local" variables** in functions
    * Without the local specifier, any variables declared within a function are global variables.

5) **Validate application and function inputs**
   * My prefered validation is to use a series of || expressions.
   * With multiple ||'s:
      * Each test/action must fail to continue in the || sequence
```
	[ -d "$target_path" ] \
     		|| mkdir -p "$target_path" \
   		|| ERROR "Target path does not exist and cannot be created"

	[ ! -f "$output_filename" ] \
		|| [ "$__clobber" ] \
		|| ERROR "Output file already exists. Use --clobber to overwrite"

   	is_digits "$age" \
   		|| ERROR: "Age must be a whole number"
   ```
     
7) **Avoid using eval**
    * If you must use it, use with extreme caution
    * See section below on safe use of eval

8) **Use `set -o nounset`**

9) When using **boolean variables** (true/false):
    * True: Use the value "true" 
    * False: Use the null value (ie "")
    * `[ "$boolean_value" ]`
    
10) Avoid combining `&&` and `||` into a single expression
    * evaluation can be unintuitive
    * `some_command || X && Y` will execute X if some_command fails, Y if some_command succeeds, and Y if X succeeds
    * `some_command && X || Y` will execute X if some_command succeeds, Y if some_command fails, and Y if X fails
    
11) Using `set -o errexit` without additional coding restrictions produces undesirable results
    * it is better to catch all errors manually:
        * `some_command blah blah || ERROR "...message..."`
        * this reads as "some_command ... OR ERROR...."
    * see section below for additional restrictions
      
12) To completely shut down a script with backgrounded child processes, use `trap 'kill -- -$$' EXIT`

13) To check if variable exists (has value or is null, but is not unset)
    * `[ "${variable+exists}" ] || function_to_call_if_variable_does_not_exist`
    * will not trigger a nounset error

# Script Loader
* Loads all functions so you don't have to worry about load order
* Allows script entry code to appear at the top of the script
* Makes using a main() function easier to do, and it's use unambigious when looking at source
* NOTE: If using this script loader, do not execute any code in the global scope (except global constants)
* NOTE: This example uses a universal argument processor (found in ~/scripts/lib/process_arguments)
```
#!/bin/dash

[ "$loading" ] || {
   set -o nounset
   loading="true"
   . "$0"
   . ~/scripts/lib/process_arguments
   main "$@"
   exit
}

main(){
   #...
}

```

# Argument Processing


## Basic Argument Processing
```
  while [ "$#" -gt 0 ]; do case "$1" in
    (--a)
      readonly argument__a="true"
      ;;
    (--name=*)
      readonly argument__name="${1#--name=}"
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
```

## Automatic Argument Processing
* Arguments are automatically read into variables
  * `--argument` becomes `$__argument="true"`
  * `--argument=value` becomes `$__argument="value"`
  * `--argument-with-dashes=some-value` becomes `$__argument_with_dashes="some-value"`
* Valid arguments must be declared before calling `process_arguments "$@"`
  * ex: `__argument=""`
  * may also be given a default value
* Arguments will be validated using a `validate__argument` function (if defined)
  * validator is passed a single argument, the parsed value
  * validator responsible for printing error and halting script if invalid
  * validator may just print warning and use an alternate value, if desired
  * or, do anything you want (use it as a hook, for example)
* This function's body can also be copypasta'd into whatever scope and will:
  * remove all parsed --options from the positional paramaters
  * leaves all non-parsed positional paramaters
  * note: local statement will need to be removed if used in a global context
* Requires an `ERROR` function
```
process_arguments(){
   local argument left right
	for argument; do 
		shift
		left="$(echo "${argument%%=*}" | tr - _)"
		right="${argument#--*=}"
		[ "$right" = "$argument" ] && right="true"
		case "$left" in
			(__*[!0-9A-Za-z_]*)
				ERROR "INVALID OPTION: $argument"
				;;
			(__*)
				eval "[ \"\${$left+exists}\" ] || ERROR \"INVALID OPTION: $argument\""
            eval "$left=\"\$right\""
				type "validate$left" > /dev/null \
					&& "validate$left" "$right"
				;;
			(*)
				set -- "$@" "$argument"
				;;
		esac
	done
}
```

### Usage Example (w/ loader script)
```

	[ "$loading" ] || {
	   set -o nounset
	   loading="true"

	   . "$0"
	   . ~/scripts/lib/process_arguments
	   main "$@"
	   exit
	}
	
	main(){
	   [ "${1+exists}" ] || {
		echo "action is required"
                exit 1
           }
`          action="$1"
           shift
           case "$action" in
               create)
                  __name=""
                  __big=""
                  process_arguments "$@"
                  do_create
                  ;;
               destroy)
                  __name=""
                  process_arguments "$@"
                  do_destroy
                  ;;
	}

        do_create(){
           : ...
        }

```

# Input Validation

### is_digits()
```is_digits(){ case "$1" in (*[!0-9]*|"") return 1;; (*) return 0;; esac; }```
note: ```[ "${1##*[!0-9]*}" ]``` also works, but is slower
### is_variable_name()
```is_variable_name(){ [ case "$1" in ([0-9]*|*[!a-zA-Z0-9_]*|"") return 1;; (*) return 0;; esac; }```





# "\n" is a lie
* The shell language doesn't unescape escaped newlines.
* This is always handed by the command you are passing it to
* If you want to print a raw string/variable, use printf:
* ```printf "%s\n" "(\n)"```

# Safe use of Eval
* Eval is dangerous because it expands variables twice
* This is also the primary use of an eval statement
* Validate all unescaped variables before using them
* All other variables simply need to be escaped:
  ```eval "$validated=\"\$unvalidated\""```
* To avoid the need to escape quote characters, use a heredoc:
  ```
  read -r eval <<-END
	$validated="\$unvalidated"
  END
  eval "$eval"
  ```


# POSIX shell-only string replacement function
```
# FUNCTION: replace <in-string> <match-string> <replace-string>
# ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
# result is stored in the $return variable
replace(){
    if [ $# != 3 ] || [ ! "$2" ]; then
	return 1
    fi
    local rest="$1"
    local left=""
    return=""
    while [ "$rest" ]; do
	left="${rest%%"$2"*}"
	rest="${rest#"$left"}"
	return="$return$left${rest:+"$3"}"
	rest="${rest#"$2"}"
   done
   return 0
}
```
(see "working with filenames" for usage example)
(for a shell function, this thing smokes in dash -- only runnung 3x slower than sed)
(this function is most useful when working with filenames stored in a variable)







# Working with filenames
* Filenames can contain any character except null
* Filenames are easily mangled when sent through a stream/pipe or stored in a file
* The safest place to keep a filename is in a variable
### To escape a filename, use:
```
readonly newline="
"
escape(){
    replace "$1" '\' '\\ '
    replace "$return" "$newline" '\n'
}
```
### to unescape a filename use:
```
unescape(){
    replace "$1" '\n' "$newline"
    replace "$return" '\\ ' '\'
}
```
### Verification test:
```
mkdir /tmp/filetest
cd /tmp/filetest
touch "\n$newline"
# more filenames
for f in *; do
   escape "$f"
   printf "%s" "$return" | {
      read -r fn
      unescape "$fn"
      [ "$f" = "$return" ] || echo "fail: '$f' != '$return'"
   }
done
cd -
rm -rf /tmp/filetest
```



        


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
