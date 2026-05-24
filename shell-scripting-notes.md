# Shell Scripting — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Your First Script](#2-your-first-script)
3. [Comments](#3-comments)
4. [Variables](#4-variables)
5. [Variable Types & Declare](#5-variable-types--declare)
6. [Special Variables](#6-special-variables)
7. [Quoting](#7-quoting)
8. [User Input — read](#8-user-input--read)
9. [Echo & Printf](#9-echo--printf)
10. [Arithmetic](#10-arithmetic)
11. [Conditionals — if / else](#11-conditionals--if--else)
12. [Case Statement](#12-case-statement)
13. [Loops](#13-loops)
14. [Functions](#14-functions)
15. [Arrays](#15-arrays)
16. [Strings](#16-strings)
17. [Command Substitution](#17-command-substitution)
18. [Process Substitution](#18-process-substitution)
19. [Redirection](#19-redirection)
20. [Pipes](#20-pipes)
21. [Exit Status & Error Handling](#21-exit-status--error-handling)
22. [Subshells & Command Grouping](#22-subshells--command-grouping)
23. [Here Documents & Here Strings](#23-here-documents--here-strings)
24. [Regular Expressions in Bash](#24-regular-expressions-in-bash)
25. [Globbing (Filename Expansion)](#25-globbing-filename-expansion)
26. [Signal Handling & Trap](#26-signal-handling--trap)
27. [Debugging](#27-debugging)
28. [IFS — Internal Field Separator](#28-ifs--internal-field-separator)
29. [Getting Options — getopts & getopt](#29-getting-options--getopts--getopt)
30. [Sed — Stream Editor](#30-sed--stream-editor)
31. [Awk — Text Processing](#31-awk--text-processing)
32. [Grep — Pattern Searching](#32-grep--pattern-searching)
33. [Find Command](#33-find-command)
34. [Xargs](#34-xargs)
35. [Crontab — Scheduling](#35-crontab--scheduling)
36. [Working with Files & Data](#36-working-with-files--data)
37. [Networking in Scripts](#37-networking-in-scripts)
38. [Logging in Scripts](#38-logging-in-scripts)
39. [Best Practices & Script Template](#39-best-practices--script-template)
40. [Advanced Topics](#40-advanced-topics)
41. [Common Patterns & Recipes](#41-common-patterns--recipes)
42. [Quick Reference Cheat Sheet](#42-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Shell?
- Shell is a **command-line interpreter** that takes commands from user and gives them to the OS to execute.
- It acts as a bridge between the **user** and the **kernel**.

### What is Shell Scripting?
- Writing a series of commands in a **file** (script) that the shell can execute automatically.
- Extension: `.sh` (convention, not required)

### Types of Shells
| Shell | Path | Description |
|-------|------|-------------|
| `bash` | `/bin/bash` | Bourne Again Shell — most common on Linux |
| `sh` | `/bin/sh` | Original Bourne Shell — POSIX compliant |
| `zsh` | `/bin/zsh` | Z Shell — modern with extra features |
| `ksh` | `/bin/ksh` | Korn Shell — scripting-focused |
| `csh` | `/bin/csh` | C Shell — C-like syntax |
| `tcsh` | `/bin/tcsh` | Enhanced C Shell |
| `fish` | `/usr/bin/fish` | Friendly Interactive Shell |
| `dash` | `/bin/dash` | Debian Almquist Shell — fast, minimal |

### Check Your Current Shell
```bash
echo $SHELL        # default login shell
echo $0            # currently running shell
cat /etc/shells    # all available shells
```

---

## 2. YOUR FIRST SCRIPT

### Create and Run
```bash
# Step 1: Create file
nano hello.sh

# Step 2: Write script
#!/bin/bash
echo "Hello, World!"

# Step 3: Make executable
chmod +x hello.sh

# Step 4: Run it
./hello.sh
# OR
bash hello.sh
# OR
sh hello.sh
```

### The Shebang Line (`#!`)
```bash
#!/bin/bash       # Use bash
#!/bin/sh         # Use POSIX sh
#!/usr/bin/env bash   # Find bash in PATH (portable)
#!/usr/bin/env python3  # Can run Python too
```
- **Must be the very first line** — no blank lines or spaces before it.
- Tells the OS which interpreter to use.
- Without it, the script runs in the current shell.

---

## 3. COMMENTS

```bash
# This is a single-line comment

echo "Hello"  # Inline comment

# Multi-line comments (hack using heredoc)
: '
This is a
multi-line
comment block
'

# Another way
<<'COMMENT'
This is also
a multi-line comment
COMMENT
```

---

## 4. VARIABLES

### Declaring Variables
```bash
# No spaces around = sign!
name="Ritesh"          # String
age=25                 # Number (still stored as string)
is_active=true         # Boolean-like (just a string)

# WRONG — these will fail:
# name = "Ritesh"      ← spaces cause error
# name ="Ritesh"       ← spaces cause error
```

### Using Variables
```bash
echo $name             # Simple usage
echo ${name}           # With braces (recommended)
echo "Hello, $name"    # Inside double quotes — EXPANDS
echo 'Hello, $name'    # Inside single quotes — LITERAL (no expansion)
echo "Age is ${age}years"  # Braces needed to avoid ambiguity
```

### Variable Naming Rules
- Can contain: `a-z`, `A-Z`, `0-9`, `_`
- **Cannot start** with a number
- **Case-sensitive**: `Name` ≠ `name` ≠ `NAME`
- Convention: lowercase for local, UPPERCASE for environment/constants

### Read-Only Variables
```bash
readonly PI=3.14
PI=3.15   # ERROR: PI: readonly variable

declare -r MAX=100   # Another way
```

### Unsetting Variables
```bash
name="Ritesh"
unset name
echo $name   # (empty)

# Cannot unset readonly variables
```

### Variable Scope
```bash
# By default, variables are LOCAL to the current shell

# Export to make available in child processes
export MY_VAR="hello"
# OR
MY_VAR="hello"
export MY_VAR

# declare -x also exports
declare -x MY_VAR="hello"
```

---

## 5. VARIABLE TYPES & DECLARE

```bash
declare -i num=10      # Integer — arithmetic evaluated automatically
declare -r const="ABC" # Read-only
declare -a arr=(1 2 3) # Indexed array
declare -A map         # Associative array (dictionary)
declare -l lower="HI"  # Convert to lowercase → "hi"
declare -u upper="hi"  # Convert to uppercase → "HI"
declare -x exported="val"  # Export to environment
declare -p name        # Print variable attributes and value
```

---

## 6. SPECIAL VARIABLES

| Variable | Description |
|----------|-------------|
| `$0` | Name of the script |
| `$1` to `$9` | Positional parameters (arguments) |
| `${10}` | 10th argument (braces required for 10+) |
| `$#` | Number of arguments passed |
| `$@` | All arguments as separate words |
| `$*` | All arguments as single string |
| `$$` | PID of current shell/script |
| `$!` | PID of last background process |
| `$?` | Exit status of last command (0=success) |
| `$-` | Current shell option flags |
| `$_` | Last argument of previous command |
| `$LINENO` | Current line number in script |
| `$FUNCNAME` | Current function name |
| `$RANDOM` | Random number (0–32767) |
| `$SECONDS` | Seconds since shell started |
| `$BASH_VERSION` | Bash version string |
| `$HOME` | Home directory |
| `$USER` | Current username |
| `$HOSTNAME` | System hostname |
| `$PWD` | Current working directory |
| `$OLDPWD` | Previous working directory |
| `$PATH` | Command search path |
| `$IFS` | Internal Field Separator (default: space, tab, newline) |
| `$SHELL` | Default login shell path |

### `$@` vs `$*`
```bash
#!/bin/bash
# Called as: ./script.sh "hello world" foo bar

# $@ — each argument stays separate
for arg in "$@"; do
    echo "$arg"
done
# Output:
# hello world
# foo
# bar

# $* — all arguments joined by first char of IFS
for arg in "$*"; do
    echo "$arg"
done
# Output:
# hello world foo bar
```

---

## 7. QUOTING

### Three Types
```bash
# 1. Double Quotes " " — allows variable expansion & command substitution
name="World"
echo "Hello $name"         # Hello World
echo "Today is $(date)"   # Today is Mon Apr 14...
echo "Path: $PATH"        # expands PATH

# 2. Single Quotes ' ' — everything is LITERAL
echo 'Hello $name'        # Hello $name
echo 'Today is $(date)'   # Today is $(date)

# 3. Backticks ` ` — command substitution (old style, avoid)
echo "Today is `date`"    # works but $() is preferred

# 4. Escape Character \
echo "Price is \$5"       # Price is $5
echo "She said \"Hi\""    # She said "Hi"
echo 'It'\''s fine'       # It's fine (break out of single quotes)
```

### ANSI-C Quoting `$'...'`
```bash
echo $'Hello\tWorld'       # Hello    World (tab)
echo $'Line1\nLine2'       # two lines
echo $'\e[31mRed\e[0m'     # colored text
```

---

## 8. USER INPUT — `read`

```bash
# Basic
read name
echo "Hello, $name"

# With prompt
read -p "Enter name: " name

# Silent input (passwords)
read -sp "Password: " pass
echo   # newline after hidden input

# With timeout
read -t 5 -p "Quick! Enter name: " name

# Read N characters
read -n 1 -p "Press any key..."

# Read into array
read -a fruits <<< "apple banana cherry"
echo ${fruits[1]}   # banana

# Read with default value
read -p "Enter name [default]: " name
name=${name:-default}

# Read from file line by line
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

### `read` Options Summary
| Option | Description |
|--------|-------------|
| `-p` | Display prompt |
| `-s` | Silent mode (hide input) |
| `-t` | Timeout in seconds |
| `-n` | Read N characters |
| `-N` | Read exactly N characters (ignore delimiter) |
| `-a` | Read into array |
| `-r` | Raw mode (don't interpret backslashes) |
| `-d` | Custom delimiter (instead of newline) |
| `-e` | Use readline (allows editing) |
| `-i` | Default text (with -e) |
| `-u` | Read from file descriptor |

---

## 9. ECHO & PRINTF

### `echo`
```bash
echo "Hello World"          # with newline
echo -n "No newline"        # suppress trailing newline
echo -e "Tab\there"         # enable escape sequences
echo -e "Line1\nLine2"      # newline
echo -e "\e[32mGreen\e[0m"  # colored text
echo -E "No \n escapes"     # explicitly disable escapes
```

### `printf` (more powerful & portable)
```bash
printf "Hello %s, you are %d years old\n" "Ritesh" 25
printf "%-20s %5d\n" "Item" 100       # left-align string, right-align number
printf "%05d\n" 42                      # 00042 — zero-padded
printf "%.2f\n" 3.14159                 # 3.14 — decimal places
printf "%x\n" 255                       # ff — hexadecimal
printf "%o\n" 8                         # 10 — octal
printf "%b\n" "Hello\tWorld"            # interpret backslash escapes
printf "%s\n" "${arr[@]}"               # print array, one per line
```

### Format Specifiers
| Specifier | Description |
|-----------|-------------|
| `%s` | String |
| `%d` / `%i` | Integer (decimal) |
| `%f` | Float |
| `%e` | Scientific notation |
| `%x` / `%X` | Hexadecimal |
| `%o` | Octal |
| `%b` | String with escape interpretation |
| `%q` | Shell-quoted string |
| `%%` | Literal percent sign |
| `%10s` | Right-align in 10 chars |
| `%-10s` | Left-align in 10 chars |
| `%.5s` | Max 5 characters |
| `%05d` | Zero-pad to 5 digits |

---

## 10. ARITHMETIC

### Methods

```bash
# 1. Double Parentheses (( )) — most common
a=10; b=3
result=$(( a + b ))        # 13
(( result = a * b ))       # 30
(( a++ ))                  # increment
(( a-- ))                  # decrement
(( a += 5 ))               # compound assignment
echo $(( 2 ** 10 ))        # 1024 — exponentiation

# 2. let command
let "sum = 5 + 3"
let "sum++"
let "a = 10 % 3"          # modulus → 1

# 3. expr command (old, external)
expr 5 + 3                 # spaces required!
expr 10 \* 3               # must escape *
result=$(expr 5 + 3)

# 4. bc — for floating point
echo "scale=2; 10/3" | bc          # 3.33
result=$(echo "scale=4; 22/7" | bc) # 3.1428
echo "sqrt(144)" | bc               # 12
echo "3.5 + 2.1" | bc               # 5.6

# 5. awk — for floating point
awk "BEGIN {print 10/3}"            # 3.33333
awk "BEGIN {printf \"%.2f\n\", 10/3}"  # 3.33

# 6. $[] — deprecated (avoid)
echo $[5+3]   # works but don't use
```

### Arithmetic Operators
| Operator | Description |
|----------|-------------|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Integer division |
| `%` | Modulus (remainder) |
| `**` | Exponentiation |
| `++` | Increment |
| `--` | Decrement |
| `+=`, `-=`, `*=`, `/=`, `%=` | Compound assignment |

### Bases
```bash
echo $(( 2#1010 ))    # Binary to decimal → 10
echo $(( 8#17 ))      # Octal to decimal → 15
echo $(( 16#FF ))     # Hex to decimal → 255
printf "%d\n" 0xFF    # 255
printf "%x\n" 255     # ff
printf "%o\n" 8       # 10
```

---

## 11. CONDITIONALS — `if / else`

### Basic Syntax
```bash
if [ condition ]; then
    # commands
elif [ condition ]; then
    # commands
else
    # commands
fi
```

### Test Operators: `[ ]` vs `[[ ]]` vs `(( ))`

```bash
# [ ] — POSIX test (works in all shells)
if [ "$a" = "$b" ]; then echo "equal"; fi

# [[ ]] — Bash enhanced test (recommended)
if [[ "$a" == "$b" ]]; then echo "equal"; fi

# (( )) — Arithmetic test
if (( a > b )); then echo "a is greater"; fi
```

### String Comparisons
```bash
[[ "$str1" == "$str2" ]]   # Equal
[[ "$str1" != "$str2" ]]   # Not equal
[[ "$str1" < "$str2" ]]    # Less than (alphabetical)
[[ "$str1" > "$str2" ]]    # Greater than (alphabetical)
[[ -z "$str" ]]            # String is empty (zero length)
[[ -n "$str" ]]            # String is NOT empty
[[ "$str" == pattern* ]]   # Glob pattern matching
[[ "$str" =~ ^[0-9]+$ ]]   # Regex matching
```

### Integer Comparisons
```bash
# Inside [ ] or [[ ]]
[[ $a -eq $b ]]    # Equal
[[ $a -ne $b ]]    # Not equal
[[ $a -gt $b ]]    # Greater than
[[ $a -lt $b ]]    # Less than
[[ $a -ge $b ]]    # Greater or equal
[[ $a -le $b ]]    # Less or equal

# Inside (( )) — use normal operators
(( a == b ))
(( a != b ))
(( a > b ))
(( a < b ))
(( a >= b ))
(( a <= b ))
```

### File Test Operators
```bash
[[ -e file ]]     # File exists (any type)
[[ -f file ]]     # Regular file exists
[[ -d dir ]]      # Directory exists
[[ -L file ]]     # Symbolic link exists
[[ -r file ]]     # Readable
[[ -w file ]]     # Writable
[[ -x file ]]     # Executable
[[ -s file ]]     # File exists & size > 0
[[ -b file ]]     # Block device
[[ -c file ]]     # Character device
[[ -p file ]]     # Named pipe
[[ -S file ]]     # Socket
[[ -t fd ]]       # File descriptor is open & is terminal
[[ -O file ]]     # Owned by current user
[[ -G file ]]     # Owned by current group
[[ -N file ]]     # Modified since last read
[[ -u file ]]     # SUID bit set
[[ -g file ]]     # SGID bit set
[[ -k file ]]     # Sticky bit set

[[ file1 -nt file2 ]]   # file1 is Newer Than file2
[[ file1 -ot file2 ]]   # file1 is Older Than file2
[[ file1 -ef file2 ]]   # Same inode (hard links)
```

### Logical Operators
```bash
# Inside [[ ]]
[[ cond1 && cond2 ]]     # AND
[[ cond1 || cond2 ]]     # OR
[[ ! condition ]]         # NOT

# Inside [ ] (POSIX)
[ cond1 ] && [ cond2 ]   # AND
[ cond1 ] || [ cond2 ]   # OR
[ ! condition ]            # NOT
# OR use -a and -o (deprecated):
[ cond1 -a cond2 ]        # AND
[ cond1 -o cond2 ]        # OR
```

### One-Liners with && and ||
```bash
# Run if true
[[ -f file.txt ]] && echo "File exists"

# Run if false
[[ -f file.txt ]] || echo "File NOT found"

# Ternary-like
[[ $age -ge 18 ]] && echo "Adult" || echo "Minor"
```

---

## 12. CASE STATEMENT

```bash
case "$variable" in
    pattern1)
        commands
        ;;
    pattern2 | pattern3)
        commands
        ;;
    *)
        default commands
        ;;
esac
```

### Examples
```bash
# Example 1: Menu
read -p "Enter choice [y/n]: " choice
case "$choice" in
    y|Y|yes|YES)
        echo "You said yes"
        ;;
    n|N|no|NO)
        echo "You said no"
        ;;
    *)
        echo "Invalid choice"
        ;;
esac

# Example 2: File extension
file="photo.jpg"
case "$file" in
    *.jpg | *.jpeg | *.png | *.gif)
        echo "Image file"
        ;;
    *.mp4 | *.avi | *.mkv)
        echo "Video file"
        ;;
    *.sh)
        echo "Shell script"
        ;;
    *)
        echo "Unknown type"
        ;;
esac

# Example 3: With character classes
read -p "Enter a character: " ch
case "$ch" in
    [a-z])   echo "Lowercase letter" ;;
    [A-Z])   echo "Uppercase letter" ;;
    [0-9])   echo "Digit" ;;
    ?)       echo "Special character" ;;
    *)       echo "String or empty" ;;
esac
```

### Bash 4.0+ Case Modifiers
```bash
case "$answer" in
    y)  echo "yes" ;;&         # ;;&  → continue testing next patterns
    ye) echo "yes" ;;&
    yes) echo "Confirmed" ;;
esac

case "$val" in
    a) echo "matched a" ;&     # ;&  → fall through to next action (no test)
    b) echo "matched b" ;;
esac
```

---

## 13. LOOPS

### `for` Loop

```bash
# Style 1: List-based
for item in apple banana cherry; do
    echo "$item"
done

# Style 2: Range
for i in {1..10}; do
    echo "$i"
done

# Range with step
for i in {0..100..5}; do
    echo "$i"    # 0, 5, 10, 15...
done

# Style 3: C-style
for (( i=0; i<10; i++ )); do
    echo "$i"
done

# Style 4: Over array
fruits=("apple" "banana" "cherry")
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Style 5: Over command output
for file in $(ls *.txt); do     # fragile, avoid
    echo "$file"
done

# Better: Use glob
for file in *.txt; do
    [[ -f "$file" ]] || continue
    echo "$file"
done

# Style 6: Over lines of a file
for line in $(cat file.txt); do     # splits on whitespace — AVOID
    echo "$line"
done

# Style 7: Over arguments
for arg in "$@"; do
    echo "$arg"
done

# Infinite for loop
for (( ;; )); do
    echo "forever"
done
```

### `while` Loop

```bash
# Basic
count=1
while [[ $count -le 5 ]]; do
    echo "Count: $count"
    (( count++ ))
done

# Read file line by line (best method)
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Read from command output
while IFS= read -r line; do
    echo "$line"
done < <(ls -la)

# Infinite while loop
while true; do
    echo "Running..."
    sleep 1
done

# While with counter
while (( count < 10 )); do
    echo $count
    (( count++ ))
done

# Read from pipe (runs in subshell — variables won't persist!)
cat file.txt | while read -r line; do
    echo "$line"
done
```

### `until` Loop
```bash
# Runs UNTIL condition becomes TRUE (opposite of while)
count=1
until [[ $count -gt 5 ]]; do
    echo "Count: $count"
    (( count++ ))
done
```

### `select` Loop (Menu)
```bash
PS3="Choose a fruit: "    # custom prompt
select fruit in apple banana cherry quit; do
    case $fruit in
        apple)  echo "You chose apple" ;;
        banana) echo "You chose banana" ;;
        cherry) echo "You chose cherry" ;;
        quit)   break ;;
        *)      echo "Invalid option" ;;
    esac
done
```

### Loop Control
```bash
# break — exit the loop
for i in {1..10}; do
    [[ $i -eq 5 ]] && break
    echo $i    # prints 1 2 3 4
done

# break N — break out of N nested loops
for i in {1..3}; do
    for j in {1..3}; do
        [[ $j -eq 2 ]] && break 2   # break both loops
    done
done

# continue — skip to next iteration
for i in {1..10}; do
    [[ $((i % 2)) -eq 0 ]] && continue
    echo $i    # prints only odd: 1 3 5 7 9
done

# continue N — skip to Nth enclosing loop
```

---

## 14. FUNCTIONS

### Defining Functions
```bash
# Style 1 (recommended)
greet() {
    echo "Hello, $1"
}

# Style 2 (Bash-specific)
function greet {
    echo "Hello, $1"
}

# Style 3 (both keyword and parens)
function greet() {
    echo "Hello, $1"
}
```

### Calling Functions
```bash
greet "Ritesh"         # Hello, Ritesh
greet "World"          # Hello, World
```

### Function Arguments
```bash
# Arguments are accessed via $1, $2, $3, ... (NOT script's args)
add() {
    echo $(( $1 + $2 ))
}
add 5 3    # 8

info() {
    echo "Function: $FUNCNAME"
    echo "Args count: $#"
    echo "All args: $@"
    echo "Script name: $0"    # still the script name, not function
}
```

### Return Values
```bash
# return sets exit status (0-255 only)
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0   # success = true
    else
        return 1   # failure = false
    fi
}

is_even 4 && echo "Even" || echo "Odd"

# Capture exit status
is_even 5
echo $?    # 1

# To return actual data — use echo + command substitution
get_sum() {
    echo $(( $1 + $2 ))
}
result=$(get_sum 10 20)
echo $result    # 30
```

### Local Variables
```bash
myfunc() {
    local name="inside"    # local to this function
    echo $name
}
name="outside"
myfunc          # inside
echo $name      # outside — unchanged
```

### Recursive Functions
```bash
factorial() {
    if (( $1 <= 1 )); then
        echo 1
    else
        local prev=$(factorial $(( $1 - 1 )))
        echo $(( $1 * prev ))
    fi
}
factorial 5    # 120
```

### Function with Nameref (Bash 4.3+)
```bash
# Pass variable by reference
swap() {
    local -n ref1=$1 ref2=$2
    local temp=$ref1
    ref1=$ref2
    ref2=$temp
}
a=10 b=20
swap a b
echo "$a $b"   # 20 10
```

### Exporting Functions
```bash
greet() { echo "Hello $1"; }
export -f greet    # available in child shells/scripts
```

---

## 15. ARRAYS

### Indexed Arrays
```bash
# Declare
fruits=("apple" "banana" "cherry")
declare -a colors

# Assign
colors[0]="red"
colors[1]="green"
colors[2]="blue"

# Access
echo ${fruits[0]}       # apple
echo ${fruits[1]}       # banana
echo ${fruits[-1]}      # cherry (last element, Bash 4.2+)

# All elements
echo ${fruits[@]}       # apple banana cherry
echo ${fruits[*]}       # apple banana cherry

# Number of elements
echo ${#fruits[@]}      # 3

# Length of specific element
echo ${#fruits[0]}      # 5 (length of "apple")

# All indices
echo ${!fruits[@]}      # 0 1 2

# Slice
echo ${fruits[@]:1:2}   # banana cherry (from index 1, count 2)
echo ${fruits[@]:1}     # banana cherry (from index 1 to end)

# Append
fruits+=("date")
fruits+=("elderberry" "fig")

# Delete element
unset fruits[1]         # removes banana (leaves gap!)
echo ${fruits[@]}       # apple cherry date elderberry fig

# Delete entire array
unset fruits

# Iterate
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Iterate with index
for i in "${!fruits[@]}"; do
    echo "$i: ${fruits[$i]}"
done
```

### Associative Arrays (Bash 4.0+)
```bash
# Must declare first
declare -A user
user[name]="Ritesh"
user[age]=25
user[city]="Delhi"

# OR declare with values
declare -A colors=(
    [red]="#FF0000"
    [green]="#00FF00"
    [blue]="#0000FF"
)

# Access
echo ${user[name]}       # Ritesh

# All values
echo ${user[@]}          # Ritesh 25 Delhi

# All keys
echo ${!user[@]}         # name age city

# Number of elements
echo ${#user[@]}         # 3

# Check if key exists
[[ -v user[name] ]] && echo "exists"

# Delete key
unset user[age]

# Iterate
for key in "${!user[@]}"; do
    echo "$key = ${user[$key]}"
done
```

### Array Operations
```bash
arr=(5 3 1 4 2)

# Sort array (using external command)
sorted=($(printf '%s\n' "${arr[@]}" | sort -n))

# Unique values
unique=($(printf '%s\n' "${arr[@]}" | sort -u))

# Join array with delimiter
joined=$(IFS=,; echo "${arr[*]}")   # 5,3,1,4,2

# Split string into array
IFS=',' read -ra parts <<< "a,b,c,d"
echo ${parts[1]}   # b

# Copy array
copy=("${arr[@]}")

# Merge arrays
merged=("${arr1[@]}" "${arr2[@]}")

# Search in array
if [[ " ${arr[*]} " == *" 3 "* ]]; then
    echo "Found 3"
fi

# Array as function argument
print_arr() {
    local -n ref=$1
    for elem in "${ref[@]}"; do
        echo "$elem"
    done
}
print_arr arr
```

---

## 16. STRINGS

### String Operations
```bash
str="Hello, World!"

# Length
echo ${#str}              # 13

# Substring (offset, length)
echo ${str:0:5}           # Hello
echo ${str:7}             # World!
echo ${str: -6}           # orld!  (space before minus!)

# Uppercase / Lowercase (Bash 4.0+)
echo ${str^^}             # HELLO, WORLD!    (all upper)
echo ${str,,}             # hello, world!    (all lower)
echo ${str^}              # Hello, World!    (first char upper)
echo ${str,}              # hello, World!    (first char lower)
echo ${str^^[aeiou]}      # HEllO, WOrld!   (upper specific chars)

# Search and Replace
str="foo bar foo baz foo"
echo ${str/foo/FOO}       # FOO bar foo baz foo  (first match)
echo ${str//foo/FOO}      # FOO bar FOO baz FOO  (all matches)
echo ${str/#foo/FOO}      # FOO bar foo baz foo  (match at start)
echo ${str/%foo/FOO}      # foo bar foo baz FOO  (match at end)
echo ${str//foo/}         # bar baz       (delete all matches)

# Remove from beginning (prefix)
file="/home/user/docs/file.tar.gz"
echo ${file#*/}           # home/user/docs/file.tar.gz   (shortest match)
echo ${file##*/}          # file.tar.gz                   (longest match → basename)

# Remove from end (suffix)
echo ${file%.*}           # /home/user/docs/file.tar     (shortest match)
echo ${file%%.*}          # /home/user/docs/file         (longest match)
echo ${file%/*}           # /home/user/docs              (dirname)

# Concatenation
first="Hello"
last="World"
full="$first $last"
full+=" !!!"
```

### Default Values
```bash
# ${var:-default}  → use default if var is unset or empty
echo ${name:-"Guest"}      # Guest (if name is unset/empty)

# ${var:=default}  → set AND use default if unset or empty
echo ${name:="Guest"}      # sets name="Guest" too

# ${var:+value}    → use value ONLY if var IS set
echo ${name:+"has name"}   # prints "has name" only if name is set

# ${var:?message}  → error if var is unset or empty
echo ${name:?"name is required!"}   # exits with error if unset
```

### Indirect Reference
```bash
var_name="greeting"
greeting="Hello World"
echo ${!var_name}          # Hello World

# List variables matching prefix
echo ${!BASH*}             # BASH BASHOPTS BASHPID BASH_VERSION ...
```

---

## 17. COMMAND SUBSTITUTION

```bash
# Modern syntax (recommended)
today=$(date)
files=$(ls -l)
count=$(wc -l < file.txt)

# Old syntax (backticks — avoid)
today=`date`

# Nesting (easy with $(), hard with backticks)
result=$(echo $(date +%Y)-$(hostname))

# Arithmetic inside
echo "Result: $(( $(wc -l < file.txt) * 2 ))"
```

---

## 18. PROCESS SUBSTITUTION

```bash
# <(command) — treat command output as a file
diff <(ls dir1) <(ls dir2)

# Compare sorted files
diff <(sort file1.txt) <(sort file2.txt)

# Read from process
while read -r line; do
    echo "$line"
done < <(find / -name "*.conf" 2>/dev/null)

# >(command) — treat as a writable file
tee >(grep ERROR > errors.log) >(grep WARN > warnings.log) < app.log > /dev/null
```

---

## 19. REDIRECTION

### Standard File Descriptors
| FD | Name | Default |
|----|------|---------|
| 0 | stdin | Keyboard |
| 1 | stdout | Screen |
| 2 | stderr | Screen |

### Redirection Operators
```bash
# stdout to file
command > file.txt          # overwrite
command >> file.txt         # append

# stderr to file
command 2> errors.txt       # overwrite
command 2>> errors.txt      # append

# stdout AND stderr to same file
command > output.txt 2>&1   # redirect both (order matters!)
command &> output.txt       # shorthand (Bash)
command &>> output.txt      # append both (Bash)

# stdout to one file, stderr to another
command > out.txt 2> err.txt

# Discard output
command > /dev/null          # discard stdout
command 2> /dev/null         # discard stderr
command &> /dev/null         # discard both

# stdin from file
command < input.txt

# stdin from string (here-string)
grep "hello" <<< "hello world"

# Here document
cat << EOF
Hello, $USER
Today is $(date)
EOF

# Here document (no variable expansion)
cat << 'EOF'
This $variable is NOT expanded
EOF

# Here document (remove leading tabs)
cat <<- EOF
	This text is indented with tabs
	Tabs are stripped from output
EOF

# File descriptor manipulation
exec 3> custom.txt      # open fd 3 for writing
echo "Hello" >&3         # write to fd 3
exec 3>&-                # close fd 3

exec 4< input.txt       # open fd 4 for reading
read line <&4            # read from fd 4
exec 4<&-                # close fd 4

exec 5<> file.txt       # open fd 5 for read/write
```

### Redirect Within Script
```bash
# Redirect all script output
exec > logfile.txt 2>&1    # all output goes to logfile

# Save and restore stdout
exec 3>&1                   # save stdout to fd 3
exec > logfile.txt          # redirect stdout to file
echo "This goes to file"
exec 1>&3                   # restore stdout
exec 3>&-
echo "This goes to screen"
```

---

## 20. PIPES

```bash
# Basic pipe
ls -la | grep ".txt"

# Chain multiple
cat access.log | grep "404" | sort | uniq -c | sort -rn | head

# Pipe to while
find . -name "*.sh" | while read -r file; do
    echo "Script: $file"
done

# Named pipe (FIFO)
mkfifo mypipe
echo "Hello" > mypipe &     # writer (blocks until reader)
cat mypipe                    # reader

# PIPESTATUS — exit status of each command in pipe
false | true | false
echo ${PIPESTATUS[@]}        # 1 0 1

# pipefail — fail if ANY command in pipe fails
set -o pipefail
false | true
echo $?   # 1 (without pipefail, would be 0)
```

---

## 21. EXIT STATUS & ERROR HANDLING

### Exit Codes
```bash
# Convention:
# 0       = success
# 1       = general error
# 2       = misuse of shell command
# 126     = command cannot execute (permission)
# 127     = command not found
# 128+n   = killed by signal n
# 130     = Ctrl+C (SIGINT = 128+2)
# 255     = exit status out of range

# Set exit status
exit 0      # success
exit 1      # failure

# Check last exit status
command
echo $?     # 0 if success
```

### Error Handling Options
```bash
# Exit immediately on error
set -e
# OR
set -o errexit

# Treat unset variables as error
set -u
# OR
set -o nounset

# Fail on pipe errors
set -o pipefail

# Debug mode — print each command before execution
set -x
# OR
set -o xtrace

# Common combo at top of scripts:
set -euo pipefail

# Turn off options with +
set +e    # disable exit-on-error
```

### Trap — Error/Signal Handling
```bash
# Syntax: trap 'commands' SIGNAL

# Cleanup on exit
cleanup() {
    rm -f /tmp/mytemp.*
    echo "Cleaned up"
}
trap cleanup EXIT

# Catch Ctrl+C
trap 'echo "Interrupted!"; exit 1' INT

# Catch errors
trap 'echo "Error on line $LINENO"; exit 1' ERR

# Ignore signal
trap '' TERM

# Reset to default
trap - INT

# Multiple signals
trap 'cleanup' EXIT INT TERM HUP

# Debug trap — runs before every command
trap 'echo "Running: $BASH_COMMAND"' DEBUG
```

### Error Handling Patterns
```bash
# Pattern 1: || with exit
cd /some/dir || { echo "Failed to cd"; exit 1; }

# Pattern 2: Custom die function
die() {
    echo "ERROR: $*" >&2
    exit 1
}
[[ -f config.txt ]] || die "config.txt not found"

# Pattern 3: try-catch simulation
{
    command1
    command2
    command3
} || {
    echo "Something failed"
}

# Pattern 4: Comprehensive error handler
error_handler() {
    local line=$1
    local cmd=$2
    local code=$3
    echo "Error on line $line: command '$cmd' exited with status $code" >&2
}
trap 'error_handler $LINENO "$BASH_COMMAND" $?' ERR
```

---

## 22. SUBSHELLS & COMMAND GROUPING

### Subshell `( )`
```bash
# Runs in a NEW child shell — changes don't affect parent
(cd /tmp && pwd)     # /tmp
pwd                  # still original directory

x=10
(x=20; echo $x)     # 20
echo $x              # 10 — unchanged!

# Useful for temporary environment changes
(export PATH="/custom:$PATH"; some_command)
```

### Command Group `{ }`
```bash
# Runs in CURRENT shell — changes persist
{ cd /tmp; pwd; }   # /tmp
pwd                  # NOW in /tmp!

# Must have space after { and ; before }
{ echo "one"; echo "two"; } > output.txt

# Both outputs go to one file
{
    echo "stdout message"
    echo "stderr message" >&2
} > out.txt 2> err.txt
```

### Differences Summary
| Feature | `( )` Subshell | `{ }` Group |
|---------|----------------|-------------|
| New process | Yes | No |
| Variable changes persist | No | Yes |
| cd affects parent | No | Yes |
| Syntax | `( cmd1; cmd2 )` | `{ cmd1; cmd2; }` |
| Trailing semicolon | Optional | Required |
| Spaces | Optional | Required after `{` |

---

## 23. HERE DOCUMENTS & HERE STRINGS

### Here Document
```bash
# Variable expansion happens
cat << EOF
Hello, $USER
Home: $HOME
Date: $(date)
EOF

# No variable expansion (quoted delimiter)
cat << 'EOF'
This $USER is literal
No $(expansion) here
EOF

# Strip leading TABS (not spaces) with <<-
cat <<- EOF
	Indented content
	Tabs are stripped
EOF

# Redirect into command
mysql -u root << EOF
SELECT * FROM users;
SHOW TABLES;
EOF

# Save to variable
message=$(cat << EOF
Line 1
Line 2
Line 3
EOF
)
```

### Here Strings
```bash
# Feed string as stdin
grep "hello" <<< "hello world"

# With variable
name="Ritesh"
read first last <<< "$name Singh"

# With command substitution
bc <<< "scale=2; 22/7"
```

---

## 24. REGULAR EXPRESSIONS IN BASH

### Using `=~` in `[[ ]]`
```bash
string="Hello123"

# Basic regex match
if [[ $string =~ ^[A-Za-z]+[0-9]+$ ]]; then
    echo "Match!"
fi

# Capture groups via BASH_REMATCH
string="2026-04-14"
if [[ $string =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "Full match: ${BASH_REMATCH[0]}"   # 2026-04-14
    echo "Year: ${BASH_REMATCH[1]}"          # 2026
    echo "Month: ${BASH_REMATCH[2]}"         # 04
    echo "Day: ${BASH_REMATCH[3]}"           # 14
fi

# Store regex in variable (avoids quoting issues)
pattern='^[0-9]+$'
if [[ "12345" =~ $pattern ]]; then
    echo "All digits"
fi
```

### Common Regex Patterns
| Pattern | Matches |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of previous |
| `+` | One or more of previous |
| `?` | Zero or one of previous |
| `^` | Start of string |
| `$` | End of string |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `[a-z]` | Range |
| `\b` | Word boundary |
| `\d` / `[0-9]` | Digit |
| `\w` / `[a-zA-Z0-9_]` | Word character |
| `\s` / `[[:space:]]` | Whitespace |
| `{n}` | Exactly n occurrences |
| `{n,m}` | Between n and m occurrences |
| `(group)` | Capture group |
| `a\|b` | Alternation (a or b) |

### POSIX Character Classes (use in `[ ]` and `grep`)
| Class | Equivalent | Description |
|-------|-----------|-------------|
| `[[:alpha:]]` | `[a-zA-Z]` | Letters |
| `[[:digit:]]` | `[0-9]` | Digits |
| `[[:alnum:]]` | `[a-zA-Z0-9]` | Letters + Digits |
| `[[:upper:]]` | `[A-Z]` | Uppercase |
| `[[:lower:]]` | `[a-z]` | Lowercase |
| `[[:space:]]` | `[\t\n\r\f\v ]` | Whitespace |
| `[[:blank:]]` | `[ \t]` | Space + Tab |
| `[[:punct:]]` | | Punctuation |
| `[[:print:]]` | | Printable chars |
| `[[:cntrl:]]` | | Control chars |
| `[[:xdigit:]]` | `[0-9a-fA-F]` | Hex digits |

---

## 25. GLOBBING (FILENAME EXPANSION)

### Basic Globs
```bash
*           # Match everything
*.txt       # All .txt files
file?.log   # file1.log, fileA.log (single char)
file[0-9].log   # file0.log through file9.log
file[!0-9].log  # NOT a digit
file[abc].log   # filea.log, fileb.log, filec.log
```

### Extended Globbing (enable with `shopt -s extglob`)
```bash
shopt -s extglob

?(pattern)    # Zero or one occurrence
*(pattern)    # Zero or more
+(pattern)    # One or more
@(pattern)    # Exactly one
!(pattern)    # Anything except pattern

# Examples
ls !(*.txt)           # everything except .txt files
ls @(*.jpg|*.png)     # only .jpg and .png
rm !(important.*)     # delete all except important.*
```

### Glob Options
```bash
shopt -s nullglob     # no match → empty (instead of literal pattern)
shopt -s dotglob      # * includes hidden files (dotfiles)
shopt -s globstar     # ** matches recursively: **/*.txt
shopt -s nocaseglob   # case-insensitive globbing
shopt -s failglob     # no match → error
```

---

## 26. SIGNAL HANDLING & TRAP

### Common Signals
| Signal | Number | Description | Default |
|--------|--------|-------------|---------|
| `SIGHUP` | 1 | Terminal hangup | Terminate |
| `SIGINT` | 2 | Ctrl+C interrupt | Terminate |
| `SIGQUIT` | 3 | Ctrl+\\ quit | Core dump |
| `SIGKILL` | 9 | Force kill (uncatchable) | Terminate |
| `SIGTERM` | 15 | Graceful termination | Terminate |
| `SIGSTOP` | 19 | Stop process (uncatchable) | Stop |
| `SIGTSTP` | 20 | Ctrl+Z suspend | Stop |
| `SIGCONT` | 18 | Continue stopped process | Continue |
| `SIGUSR1` | 10 | User-defined 1 | Terminate |
| `SIGUSR2` | 12 | User-defined 2 | Terminate |
| `EXIT` | - | Shell exit (pseudo-signal) | - |
| `ERR` | - | Command error (pseudo-signal) | - |
| `DEBUG` | - | Before each command (pseudo) | - |
| `RETURN` | - | After function/source returns | - |

### Trap Examples
```bash
# Cleanup temporary files on exit
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT

# Handle Ctrl+C gracefully
trap 'echo "Use quit command to exit"; ' INT

# Logging on exit
trap 'echo "Script ended at $(date)" >> script.log' EXIT

# Ignore signals
trap '' INT TERM    # now Ctrl+C won't work

# Reset to default behavior
trap - INT TERM

# List all traps currently set
trap -p
```

---

## 27. DEBUGGING

### Debug Options
```bash
# Method 1: Set at top of script
set -x          # Print each command before executing
set -v          # Print each line as read
set -e          # Exit on first error
set -u          # Error on undefined variables

# Method 2: On shebang line
#!/bin/bash -x

# Method 3: From command line
bash -x script.sh

# Debug specific section only
set -x
# ... problematic code ...
set +x    # turn off

# Method 4: Using PS4 for better debug output
export PS4='+${BASH_SOURCE}:${LINENO}: ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

### Debug Tools
```bash
# Print variable values
echo "DEBUG: var=$var" >&2

# Print function call stack
caller        # line number and source file
caller 0      # current
caller 1      # parent

# Full stack trace
print_stack() {
    local i=0
    while caller $i; do
        (( i++ ))
    done
}

# Check syntax without executing
bash -n script.sh

# Verbose mode — print lines as read
bash -v script.sh

# Using trap for debugging
trap '(read -p "[$BASH_SOURCE:$LINENO] $BASH_COMMAND? ")' DEBUG
```

### Shellcheck — Static Analysis
```bash
# Install
apt install shellcheck    # Debian/Ubuntu
yum install shellcheck    # RHEL/CentOS

# Usage
shellcheck script.sh
shellcheck -s bash script.sh    # specify shell
shellcheck -e SC2034 script.sh  # exclude specific check

# Common issues it catches:
# SC2086: Double quote to prevent globbing and word splitting
# SC2046: Quote this to prevent word splitting
# SC2006: Use $(...) instead of `...`
# SC2034: Variable appears unused
```

---

## 28. IFS — Internal Field Separator

```bash
# Default IFS = space, tab, newline
echo "$IFS" | cat -A    # shows: $\t\n

# Change IFS temporarily
old_IFS=$IFS

# Split on comma
IFS=',' read -ra fields <<< "a,b,c,d"
echo ${fields[1]}   # b

# Split on colon (like PATH)
IFS=':' read -ra dirs <<< "$PATH"
for dir in "${dirs[@]}"; do
    echo "$dir"
done

# IFS affects $* (but not $@)
set -- "one" "two" "three"
IFS=','
echo "$*"    # one,two,three
echo "$@"    # one two three

# Restore
IFS=$old_IFS

# Read CSV
while IFS=',' read -r name age city; do
    echo "Name: $name, Age: $age, City: $city"
done < data.csv

# Unset IFS (uses default behavior)
unset IFS

# Set IFS for a single command only
IFS=: command    # only affects that one command
```

---

## 29. GETTING OPTIONS — `getopts` & `getopt`

### `getopts` (Built-in, POSIX)
```bash
#!/bin/bash

usage() {
    echo "Usage: $0 [-v] [-f file] [-n num] [-h]"
    exit 1
}

verbose=false
file=""
num=0

while getopts "vf:n:h" opt; do
    case $opt in
        v) verbose=true ;;
        f) file="$OPTARG" ;;      # : means requires argument
        n) num="$OPTARG" ;;
        h) usage ;;
        \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
        :) echo "Option -$OPTARG requires argument" >&2; exit 1 ;;
    esac
done

# Shift processed options
shift $((OPTIND - 1))

# Remaining args
echo "Remaining args: $@"
echo "Verbose: $verbose"
echo "File: $file"
echo "Number: $num"
```

### `getopt` (External, supports long options)
```bash
#!/bin/bash

OPTS=$(getopt -o vf:n:h --long verbose,file:,number:,help -n "$0" -- "$@")
eval set -- "$OPTS"

verbose=false
file=""
num=0

while true; do
    case "$1" in
        -v|--verbose) verbose=true; shift ;;
        -f|--file)    file="$2"; shift 2 ;;
        -n|--number)  num="$2"; shift 2 ;;
        -h|--help)    usage; shift ;;
        --)           shift; break ;;
        *)            echo "Error"; exit 1 ;;
    esac
done

echo "Remaining: $@"
```

### Manual Parsing
```bash
while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose) verbose=true; shift ;;
        -f|--file)    file="$2"; shift 2 ;;
        --)           shift; break ;;
        -*)           echo "Unknown option: $1"; exit 1 ;;
        *)            args+=("$1"); shift ;;
    esac
done
```

---

## 30. SED — Stream Editor

### Syntax
```bash
sed [options] 'command' file
sed [options] -e 'cmd1' -e 'cmd2' file
sed [options] -f script.sed file
```

### Substitution
```bash
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences (global)
sed 's/old/new/g' file.txt

# Replace Nth occurrence
sed 's/old/new/3' file.txt

# Case-insensitive
sed 's/old/new/gi' file.txt

# In-place editing (modify file)
sed -i 's/old/new/g' file.txt

# In-place with backup
sed -i.bak 's/old/new/g' file.txt

# Different delimiters
sed 's|/usr/local|/opt|g' file.txt
sed 's#old#new#g' file.txt

# Capture groups
sed 's/\(Hello\) \(World\)/\2 \1/' file.txt     # World Hello
sed -E 's/(Hello) (World)/\2 \1/' file.txt       # extended regex

# & — entire matched string
sed 's/[0-9]*/(&)/' file.txt    # wraps number in parentheses
```

### Line Operations
```bash
# Print specific line
sed -n '5p' file.txt             # line 5
sed -n '3,7p' file.txt           # lines 3 to 7
sed -n '$p' file.txt             # last line

# Delete lines
sed '3d' file.txt                # delete line 3
sed '3,5d' file.txt              # delete lines 3-5
sed '/pattern/d' file.txt        # delete matching lines
sed '/^$/d' file.txt             # delete empty lines
sed '/^#/d' file.txt             # delete comment lines

# Insert text
sed '3i\New line before 3' file  # insert BEFORE line 3
sed '3a\New line after 3' file   # append AFTER line 3

# Replace entire line
sed '3c\Replacement line' file   # replace line 3

# Line addressing
sed '1,5s/old/new/g' file        # only in lines 1-5
sed '/start/,/end/s/old/new/g' file  # between patterns
sed '0~2s/old/new/g' file        # every 2nd line starting from 0

# Print line numbers
sed -n '/pattern/=' file.txt
```

### Multi-line & Advanced
```bash
# Multiple commands
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
sed 's/foo/bar/g; s/baz/qux/g' file.txt

# Read from file
sed '/MARKER/r insert.txt' file.txt

# Write to file
sed -n '/pattern/w output.txt' file.txt

# Transform (like tr)
sed 'y/abc/ABC/' file.txt    # a→A, b→B, c→C

# Print only changed lines
sed -n 's/old/new/gp' file.txt
```

---

## 31. AWK — Text Processing

### Basic Syntax
```bash
awk 'pattern { action }' file
awk -F'delimiter' '{ action }' file
```

### Built-in Variables
| Variable | Description |
|----------|-------------|
| `$0` | Entire current line |
| `$1, $2, ...` | Fields (columns) |
| `NR` | Current line number (total) |
| `NF` | Number of fields in current line |
| `FS` | Input field separator (default: space) |
| `OFS` | Output field separator (default: space) |
| `RS` | Input record separator (default: newline) |
| `ORS` | Output record separator (default: newline) |
| `FILENAME` | Current filename |
| `FNR` | Line number in current file |
| `$NF` | Last field |
| `$(NF-1)` | Second to last field |

### Basic Operations
```bash
# Print entire file
awk '{ print }' file.txt

# Print specific columns
awk '{ print $1, $3 }' file.txt

# Custom delimiter
awk -F',' '{ print $1, $2 }' data.csv
awk -F':' '{ print $1 }' /etc/passwd

# Set output separator
awk -F',' 'BEGIN { OFS="\t" } { print $1, $2 }' file.csv

# Print with format
awk '{ printf "%-20s %5d\n", $1, $2 }' file.txt
```

### Patterns & Conditions
```bash
# Match pattern
awk '/error/' file.txt               # lines containing "error"
awk '!/error/' file.txt              # lines NOT containing "error"
awk '/start/,/end/' file.txt         # range between patterns

# Condition
awk '$3 > 100' file.txt              # 3rd field > 100
awk '$1 == "admin"' file.txt         # 1st field equals "admin"
awk 'NR >= 5 && NR <= 10' file.txt   # lines 5 to 10
awk 'NR % 2 == 0' file.txt           # even lines
awk 'NF > 3' file.txt                # lines with more than 3 fields
awk 'length > 80' file.txt           # lines longer than 80 chars
```

### BEGIN and END
```bash
awk 'BEGIN { print "Header" } { print $0 } END { print "Footer" }' file.txt

# Sum of column
awk '{ sum += $2 } END { print "Total:", sum }' file.txt

# Average
awk '{ sum += $2; count++ } END { print "Avg:", sum/count }' file.txt

# Count lines
awk 'END { print NR }' file.txt

# Find max
awk 'BEGIN { max=0 } $2 > max { max=$2 } END { print "Max:", max }' file.txt
```

### String Functions
```bash
awk '{ print length($0) }' file        # string length
awk '{ print toupper($1) }' file       # uppercase
awk '{ print tolower($1) }' file       # lowercase
awk '{ print substr($1, 2, 3) }' file  # substring
awk '{ gsub(/old/, "new"); print }' file  # global replace
awk '{ sub(/old/, "new"); print }' file   # first replace
awk '{ split($0, arr, ":"); print arr[1] }' file  # split
awk 'match($0, /[0-9]+/) { print substr($0, RSTART, RLENGTH) }' file  # regex match
awk '{ idx = index($0, "find"); print idx }' file  # find position
awk 'BEGIN { printf "%s\n", sprintf("%.2f", 3.14159) }'  # sprintf
```

### Arithmetic
```bash
awk '{ print $1 + $2 }' file        # add columns
awk '{ print $1 * $2 }' file        # multiply
awk '{ print int($1 / $2) }' file   # integer division
awk '{ print $1 % $2 }' file        # modulus
awk '{ print sqrt($1) }' file       # square root
awk '{ print log($1) }' file        # natural log
awk '{ print sin($1) }' file        # sine
awk '{ print int(rand() * 100) }'   # random number
```

### Associative Arrays
```bash
# Count occurrences
awk '{ count[$1]++ } END { for (key in count) print key, count[key] }' file

# Delete element
awk '{ arr[$1]=$2 } END { delete arr["key"]; for (k in arr) print k, arr[k] }' file

# Check existence
awk '{ if ($1 in seen) print "dup:", $0; seen[$1]=1 }' file
```

### Multi-line Programs
```bash
awk -f program.awk file.txt

# In a file (program.awk):
BEGIN {
    FS = ","
    OFS = "\t"
    print "Name", "Age", "City"
}
NR > 1 {
    print $1, $2, $3
}
END {
    print "Total records:", NR - 1
}
```

### Awk Control Flow
```bash
# if-else
awk '{ if ($2 > 50) print "High:", $0; else print "Low:", $0 }' file

# for loop
awk '{ for (i=1; i<=NF; i++) print $i }' file

# while loop
awk '{ i=1; while (i<=NF) { print $i; i++ } }' file

# Ternary
awk '{ print ($2 > 50 ? "High" : "Low"), $0 }' file

# next (skip to next record)
awk '/^#/ { next } { print }' file    # skip comments

# exit
awk '{ if (NR > 100) exit; print }' file
```

---

## 32. GREP — PATTERN SEARCHING

### Basic Usage
```bash
grep "pattern" file.txt
grep "pattern" file1.txt file2.txt    # multiple files
command | grep "pattern"               # from stdin
```

### Common Options
```bash
grep -i "pattern" file        # ignore case
grep -v "pattern" file        # invert match (non-matching lines)
grep -n "pattern" file        # show line numbers
grep -c "pattern" file        # count matches
grep -l "pattern" *.txt       # files containing match
grep -L "pattern" *.txt       # files NOT containing match
grep -w "word" file           # whole word match
grep -x "exact line" file     # exact line match
grep -r "pattern" /dir        # recursive search
grep -R "pattern" /dir        # recursive, follow symlinks
grep -o "pattern" file        # only matching part
grep -m 5 "pattern" file      # stop after 5 matches
grep -q "pattern" file        # quiet (just set exit code)
grep --color=auto "pattern" file  # colorize
```

### Context
```bash
grep -A 3 "pattern" file      # 3 lines After
grep -B 3 "pattern" file      # 3 lines Before
grep -C 3 "pattern" file      # 3 lines Context (before+after)
```

### Regex with grep
```bash
# Basic Regex (BRE — default)
grep "^start" file             # starts with
grep "end$" file               # ends with
grep "^$" file                 # empty lines
grep "a.b" file                # a, any char, b
grep "ab*c" file               # a, zero+ b's, c
grep "a\+b" file               # one or more (BRE needs \)
grep "a\?b" file               # zero or one
grep "[0-9]" file              # digit
grep "[^0-9]" file             # not digit

# Extended Regex (ERE)
grep -E "ab+c" file            # one or more (no escape)
grep -E "ab?c" file            # zero or one
grep -E "cat|dog" file         # alternation
grep -E "(ab){3}" file         # exactly 3 of "ab"
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" file   # IP-like

# Perl-Compatible Regex (PCRE)
grep -P "\d{3}-\d{3}-\d{4}" file      # phone number
grep -P "(?<=@)\w+" file               # lookbehind
grep -P "\b\w+(?=ing\b)" file         # lookahead
```

### Useful Grep Recipes
```bash
# Find files containing string recursively
grep -rl "TODO" src/

# Count occurrences (not lines)
grep -o "pattern" file | wc -l

# Search only specific file types
grep -r --include="*.py" "import" .

# Exclude directories
grep -r --exclude-dir=node_modules "pattern" .

# Multiple patterns
grep -e "pat1" -e "pat2" file
grep -E "pat1|pat2" file

# Fixed strings (no regex)
grep -F "literal.string*" file
```

---

## 33. FIND COMMAND

### Basic Syntax
```bash
find [path] [conditions] [actions]
```

### By Name
```bash
find . -name "*.txt"              # case-sensitive
find . -iname "*.txt"             # case-insensitive
find / -name "config"             # find from root
find . -not -name "*.log"         # exclude pattern
find . -name "*.js" -o -name "*.ts"  # OR
```

### By Type
```bash
find . -type f       # regular files
find . -type d       # directories
find . -type l       # symbolic links
find . -type b       # block devices
find . -type c       # character devices
find . -type p       # named pipes
find . -type s       # sockets
```

### By Size
```bash
find . -size +100M      # larger than 100MB
find . -size -1k        # smaller than 1KB
find . -size 50c        # exactly 50 bytes
# Units: c(bytes), k(KB), M(MB), G(GB)
find . -empty           # empty files and directories
```

### By Time
```bash
# Modified time (days)
find . -mtime 0         # modified today (last 24h)
find . -mtime -7        # modified within 7 days
find . -mtime +30       # modified more than 30 days ago

# Access time
find . -atime -7        # accessed within 7 days

# Changed time (metadata)
find . -ctime -7        # status changed within 7 days

# Minutes instead of days
find . -mmin -60        # modified within 60 minutes

# Newer than file
find . -newer ref.txt   # modified after ref.txt
```

### By Permissions & Ownership
```bash
find . -perm 644        # exact permissions
find . -perm -644       # at least these permissions
find . -perm /644       # any of these permissions
find . -user ritesh     # owned by user
find . -group dev       # owned by group
find . -nouser          # no owner
find . -nogroup         # no group
```

### Depth Control
```bash
find . -maxdepth 2 -name "*.txt"   # max 2 levels deep
find . -mindepth 1 -maxdepth 3     # between 1 and 3
```

### Actions
```bash
# Print (default)
find . -name "*.txt" -print

# Print with null separator (for xargs)
find . -name "*.txt" -print0

# Delete
find . -name "*.tmp" -delete

# Execute command for each
find . -name "*.log" -exec rm {} \;          # one by one
find . -name "*.txt" -exec grep "TODO" {} \; # search each
find . -name "*.sh" -exec chmod +x {} \;     # make executable

# Execute with multiple files at once (faster)
find . -name "*.txt" -exec grep "TODO" {} +

# Confirmation before action
find . -name "*.tmp" -ok rm {} \;   # asks y/n each time

# Combine with xargs
find . -name "*.txt" -print0 | xargs -0 grep "pattern"
find . -type f -print0 | xargs -0 chmod 644

# Execute multiple commands
find . -name "*.log" -exec echo "Found: {}" \; -exec wc -l {} \;
```

### Combining Conditions
```bash
# AND (implicit)
find . -name "*.txt" -size +1M

# OR
find . -name "*.jpg" -o -name "*.png"
find . \( -name "*.jpg" -o -name "*.png" \) -size +1M

# NOT
find . -not -name "*.txt"
find . ! -name "*.txt"
```

---

## 34. XARGS

```bash
# Basic — pass stdin as arguments
echo "file1 file2 file3" | xargs rm

# Specify max args per command
echo {1..100} | xargs -n 10 echo    # 10 per line

# Custom placeholder
find . -name "*.txt" | xargs -I{} cp {} /backup/

# Null-delimited (handle spaces in filenames)
find . -name "*.txt" -print0 | xargs -0 ls -l

# Parallel execution
find . -name "*.png" -print0 | xargs -0 -P 4 -I{} convert {} {}.jpg

# Prompt before execution
echo "file1 file2" | xargs -p rm

# Verbose
echo "file1 file2" | xargs -t rm    # prints command before running

# Max command length
echo {1..1000} | xargs -s 200 echo

# With delimiter
echo "a,b,c,d" | xargs -d',' -n1 echo
```

---

## 35. CRONTAB — SCHEDULING

### Crontab Syntax
```
┌───────── minute (0–59)
│ ┌─────── hour (0–23)
│ │ ┌───── day of month (1–31)
│ │ │ ┌─── month (1–12 or JAN–DEC)
│ │ │ │ ┌─ day of week (0–7, 0 and 7 = Sunday, or SUN–SAT)
│ │ │ │ │
* * * * * command
```

### Examples
```bash
# Edit crontab
crontab -e

# List crontab
crontab -l

# Remove all crontabs
crontab -r

# Common schedules
* * * * *     command          # every minute
*/5 * * * *   command          # every 5 minutes
0 * * * *     command          # every hour
0 0 * * *     command          # every midnight
0 6 * * 1-5  command           # 6 AM, Mon-Fri
0 0 1 * *    command           # 1st of every month
0 0 * * 0    command           # every Sunday midnight
30 4 1,15 * * command          # 4:30 AM on 1st and 15th
0 9-17 * * 1-5 command         # every hour 9-5 weekdays

# Special strings
@reboot       command          # on startup
@yearly       command          # once a year (0 0 1 1 *)
@monthly      command          # once a month (0 0 1 * *)
@weekly       command          # once a week (0 0 * * 0)
@daily        command          # once a day (0 0 * * *)
@hourly       command          # once an hour (0 * * * *)

# Environment in crontab
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=user@example.com

# Output handling
* * * * * command > /dev/null 2>&1          # discard output
* * * * * command >> /var/log/job.log 2>&1  # log output
```

---

## 36. WORKING WITH FILES & DATA

### Temporary Files
```bash
# Create safe temp file
tmpfile=$(mktemp)                    # e.g., /tmp/tmp.Xk3j2h
tmpdir=$(mktemp -d)                  # temp directory
tmpfile=$(mktemp /tmp/myapp.XXXXXX)  # custom template

# Cleanup
trap 'rm -f "$tmpfile"' EXIT

# Write and use
echo "data" > "$tmpfile"
process < "$tmpfile"
```

### Locking (Prevent Multiple Instances)
```bash
LOCKFILE="/var/lock/myscript.lock"

# Method 1: Using flock
(
    flock -n 200 || { echo "Already running"; exit 1; }
    # ... script body ...
) 200>"$LOCKFILE"

# Method 2: Using mkdir (atomic)
if ! mkdir "$LOCKFILE" 2>/dev/null; then
    echo "Already running"
    exit 1
fi
trap 'rmdir "$LOCKFILE"' EXIT
```

### Config File Parsing
```bash
# Source a config file (key=value format)
source config.sh    # DANGEROUS if untrusted!

# Safe parsing
while IFS='=' read -r key value; do
    key=$(echo "$key" | tr -d '[:space:]')
    value=$(echo "$value" | tr -d '[:space:]')
    case "$key" in
        host)     HOST=$value ;;
        port)     PORT=$value ;;
        timeout)  TIMEOUT=$value ;;
    esac
done < config.txt

# Parse INI-style
parse_ini() {
    local section=""
    while IFS= read -r line; do
        line=$(echo "$line" | sed 's/#.*//' | xargs)
        [[ -z "$line" ]] && continue
        if [[ "$line" =~ ^\[(.+)\]$ ]]; then
            section="${BASH_REMATCH[1]}"
        elif [[ "$line" =~ ^([^=]+)=(.*)$ ]]; then
            echo "${section}_${BASH_REMATCH[1]}=${BASH_REMATCH[2]}"
        fi
    done < "$1"
}
```

### CSV Processing
```bash
# Read CSV
while IFS=',' read -r name age city; do
    echo "Name=$name Age=$age City=$city"
done < data.csv

# Skip header
tail -n +2 data.csv | while IFS=',' read -r name age city; do
    echo "$name is $age years old"
done

# Using awk for CSV
awk -F',' 'NR>1 { print $1, $2 }' data.csv
```

---

## 37. NETWORKING IN SCRIPTS

```bash
# Check if host is reachable
ping -c 1 -W 2 google.com > /dev/null 2>&1 && echo "UP" || echo "DOWN"

# Check if port is open
(echo > /dev/tcp/hostname/80) 2>/dev/null && echo "Open" || echo "Closed"

# Download file
curl -sO https://example.com/file.txt
wget -q https://example.com/file.txt

# API call
response=$(curl -s https://api.example.com/data)

# POST request
curl -s -X POST -H "Content-Type: application/json" \
    -d '{"key":"value"}' https://api.example.com/endpoint

# Check HTTP status
status=$(curl -s -o /dev/null -w "%{http_code}" https://example.com)

# TCP client with bash
exec 3<>/dev/tcp/hostname/port
echo "GET / HTTP/1.0" >&3
cat <&3
exec 3<&-

# Send email
echo "Body" | mail -s "Subject" user@example.com

# SSH remote execution
ssh user@host 'command'
ssh user@host 'bash -s' < local_script.sh
```

---

## 38. LOGGING IN SCRIPTS

```bash
# Simple logging function
LOG_FILE="/var/log/myscript.log"

log() {
    local level=$1
    shift
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$level] $*" | tee -a "$LOG_FILE"
}

log INFO "Script started"
log WARN "Disk space low"
log ERROR "Connection failed"

# Log levels
log_info()  { log INFO  "$@"; }
log_warn()  { log WARN  "$@"; }
log_error() { log ERROR "$@" >&2; }
log_debug() { [[ ${DEBUG:-0} -eq 1 ]] && log DEBUG "$@"; }

# Rotate log
rotate_log() {
    local max_size=$((10 * 1024 * 1024))  # 10MB
    if [[ -f "$LOG_FILE" ]] && (( $(stat -c%s "$LOG_FILE") > max_size )); then
        mv "$LOG_FILE" "${LOG_FILE}.old"
    fi
}

# Send to syslog
logger -t myscript "Something happened"
logger -p user.error -t myscript "Error occurred"
```

---

## 39. BEST PRACTICES & SCRIPT TEMPLATE

### Best Practices
```
1.  Always use #!/bin/bash (or #!/usr/bin/env bash)
2.  Use set -euo pipefail at the top
3.  Always double-quote variables: "$var" not $var
4.  Use [[ ]] instead of [ ] for tests
5.  Use $() instead of backticks
6.  Use (( )) for arithmetic
7.  Use local variables in functions
8.  Always check return values / exit codes
9.  Use trap for cleanup
10. Use meaningful variable names (UPPER for constants, lower for locals)
11. Use readonly for constants
12. Quote command substitution: "$(command)"
13. Use -- to end option processing: rm -- "$file"
14. Create usage/help function
15. Validate all inputs
16. Log errors to stderr: echo "Error" >&2
17. Use shellcheck for linting
18. Don't parse ls output — use globs or find
19. Use mktemp for temporary files
20. Handle signals with trap
```

### Script Template
```bash
#!/usr/bin/env bash
#
# Script: myscript.sh
# Description: Brief description
# Author: Your Name
# Date: 2026-04-14
# Version: 1.0
#

set -euo pipefail
IFS=$'\n\t'

# ─── Constants ────────────────────────────────────
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
readonly VERSION="1.0.0"

# ─── Defaults ─────────────────────────────────────
VERBOSE=false
DRY_RUN=false
LOG_FILE="/tmp/${SCRIPT_NAME}.log"

# ─── Functions ────────────────────────────────────
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <argument>

Description of what the script does.

Options:
    -v, --verbose     Enable verbose output
    -n, --dry-run     Show what would be done
    -h, --help        Show this help message
    -V, --version     Show version

Examples:
    $SCRIPT_NAME -v input.txt
    $SCRIPT_NAME --dry-run /path/to/dir
EOF
    exit "${1:-0}"
}

version() {
    echo "$SCRIPT_NAME v$VERSION"
    exit 0
}

log() {
    local level=$1; shift
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$level] $*" >> "$LOG_FILE"
    if [[ "$VERBOSE" == true ]] || [[ "$level" == "ERROR" ]]; then
        if [[ "$level" == "ERROR" ]]; then
            echo "[$level] $*" >&2
        else
            echo "[$level] $*"
        fi
    fi
}

die() {
    log ERROR "$@"
    exit 1
}

cleanup() {
    # Remove temp files, restore state, etc.
    log INFO "Cleanup complete"
}

# ─── Trap ─────────────────────────────────────────
trap cleanup EXIT
trap 'die "Received SIGINT"' INT
trap 'die "Received SIGTERM"' TERM

# ─── Parse Arguments ──────────────────────────────
while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)  VERBOSE=true; shift ;;
        -n|--dry-run)  DRY_RUN=true; shift ;;
        -h|--help)     usage 0 ;;
        -V|--version)  version ;;
        --)            shift; break ;;
        -*)            die "Unknown option: $1" ;;
        *)             break ;;
    esac
done

# ─── Validation ───────────────────────────────────
[[ $# -lt 1 ]] && { usage 1; }
[[ -f "$1" ]] || die "File not found: $1"

# ─── Main Logic ───────────────────────────────────
main() {
    log INFO "Script started with args: $*"

    # Your logic here

    log INFO "Script completed successfully"
}

main "$@"
```

---

## 40. ADVANCED TOPICS

### Coprocesses
```bash
# Start a background process with two-way communication
coproc MY_PROC { while read -r line; do echo "REPLY: $line"; done; }

echo "Hello" >&${MY_PROC[1]}     # send to coprocess stdin
read -r response <&${MY_PROC[0]}  # read from coprocess stdout
echo "$response"                    # REPLY: Hello

# Close
exec {MY_PROC[1]}>&-
```

### Process Substitution
```bash
# Compare two command outputs
diff <(sort file1) <(sort file2)

# Write to multiple pipelines
tee >(gzip > file.gz) >(wc -l) < input.txt > /dev/null

# Feed into while without subshell
while read -r line; do
    ((count++))
done < <(find . -name "*.txt")
echo $count    # variable persists!
```

### Eval & Indirect Expansion
```bash
# eval — execute string as command (USE WITH CAUTION)
cmd="ls -la"
eval "$cmd"

# Indirect variable reference
var_name="greeting"
greeting="Hello"
echo ${!var_name}    # Hello

# Dynamic variable names
for i in 1 2 3; do
    declare "var_$i=value_$i"
done
echo $var_1    # value_1
```

### FIFOs (Named Pipes)
```bash
# Create named pipe
mkfifo /tmp/mypipe

# Producer (will block until consumer reads)
echo "data" > /tmp/mypipe &

# Consumer
cat /tmp/mypipe

# Cleanup
rm /tmp/mypipe
```

### Parallel Execution
```bash
# Background processes
for i in {1..5}; do
    process_item "$i" &
done
wait    # wait for all background jobs

# With max parallel limit
max_jobs=4
for item in "${items[@]}"; do
    (( $(jobs -r | wc -l) >= max_jobs )) && wait -n
    process "$item" &
done
wait

# Using GNU parallel
parallel -j 4 process_item ::: {1..20}
parallel -j 4 convert {} {.}.png ::: *.jpg

# Using xargs parallel
find . -name "*.txt" -print0 | xargs -0 -P 4 -I{} process {}
```

### Source & Libraries
```bash
# Create a library file (lib.sh)
#!/bin/bash
log() { echo "[$(date)] $*"; }
die() { echo "FATAL: $*" >&2; exit 1; }

# Use it in your script
source ./lib.sh       # or . ./lib.sh
log "Starting"
```

### Associative Array Tricks
```bash
declare -A config

# Load from file
while IFS='=' read -r key value; do
    [[ "$key" =~ ^[[:space:]]*# ]] && continue  # skip comments
    [[ -z "$key" ]] && continue                   # skip empty
    config["$key"]="$value"
done < config.txt

# Check key exists
[[ -v config["key"] ]] && echo "exists"

# Default value
val="${config[key]:-default}"

# Serialize/dump
declare -p config
```

### Here Document as Function Input
```bash
create_html() {
    cat << EOF
<!DOCTYPE html>
<html>
<head><title>$1</title></head>
<body><h1>$2</h1></body>
</html>
EOF
}

create_html "My Page" "Welcome" > index.html
```

### Bash Strict Mode Explained
```bash
set -e           # Exit immediately if a command exits with non-zero
set -u           # Treat unset variables as error
set -o pipefail  # Pipe fails if ANY command fails (not just last)
IFS=$'\n\t'      # Safer word splitting (only newline and tab)

# Shorthand:
set -euo pipefail

# CAVEATS with set -e:
# These do NOT trigger exit:
command || true              # explicitly handled
if command; then ...; fi     # inside condition
command && other             # left side of &&
command1 || command2         # left side of ||
```

---

## 41. COMMON PATTERNS & RECIPES

### Check if Root
```bash
[[ $EUID -ne 0 ]] && { echo "Must run as root"; exit 1; }
```

### Check if Command Exists
```bash
command -v docker &>/dev/null || { echo "docker not found"; exit 1; }
# OR
type git &>/dev/null || die "git is required"
# OR
which python3 &>/dev/null || die "python3 not found"
```

### Check OS / Distribution
```bash
case "$(uname -s)" in
    Linux*)   OS=Linux ;;
    Darwin*)  OS=Mac ;;
    CYGWIN*)  OS=Cygwin ;;
    MINGW*)   OS=MinGW ;;
    *)        OS=Unknown ;;
esac

# Linux distribution
if [[ -f /etc/os-release ]]; then
    source /etc/os-release
    echo "$ID $VERSION_ID"    # ubuntu 22.04
fi
```

### Yes/No Confirmation
```bash
confirm() {
    read -rp "$1 [y/N]: " answer
    [[ "$answer" =~ ^[Yy]([Ee][Ss])?$ ]]
}

if confirm "Delete all files?"; then
    rm -rf *
fi
```

### Spinner / Progress
```bash
spinner() {
    local pid=$1
    local chars='|/-\'
    local i=0
    while kill -0 "$pid" 2>/dev/null; do
        printf "\r%c" "${chars:i++%4:1}"
        sleep 0.1
    done
    printf "\r"
}

long_task &
spinner $!
```

### Retry Pattern
```bash
retry() {
    local max=$1; shift
    local delay=$1; shift
    local attempt=1
    while (( attempt <= max )); do
        "$@" && return 0
        echo "Attempt $attempt/$max failed. Retrying in ${delay}s..."
        sleep "$delay"
        (( attempt++ ))
    done
    return 1
}

retry 3 5 curl -s https://example.com
```

### Menu Function
```bash
menu() {
    local prompt="$1"; shift
    local options=("$@")
    PS3="$prompt "
    select opt in "${options[@]}" "Quit"; do
        [[ "$opt" == "Quit" ]] && exit 0
        [[ -n "$opt" ]] && { echo "$opt"; return; }
        echo "Invalid option"
    done
}

choice=$(menu "Select OS:" "Ubuntu" "CentOS" "Arch")
echo "You chose: $choice"
```

### Color Output
```bash
# ANSI color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
WHITE='\033[1;37m'
NC='\033[0m'    # No Color (reset)
BOLD='\033[1m'
UNDERLINE='\033[4m'

echo -e "${RED}Error!${NC}"
echo -e "${GREEN}Success!${NC}"
echo -e "${YELLOW}Warning!${NC}"
echo -e "${BOLD}Bold text${NC}"

# Functions
red()    { echo -e "${RED}$*${NC}"; }
green()  { echo -e "${GREEN}$*${NC}"; }
yellow() { echo -e "${YELLOW}$*${NC}"; }

# Check if terminal supports colors
if [[ -t 1 ]] && tput colors &>/dev/null; then
    # use colors
    :
else
    # strip colors
    RED='' GREEN='' YELLOW='' NC=''
fi
```

### Progress Bar
```bash
progress_bar() {
    local current=$1
    local total=$2
    local width=50
    local pct=$(( current * 100 / total ))
    local filled=$(( current * width / total ))
    local empty=$(( width - filled ))
    printf "\r["
    printf "%${filled}s" | tr ' ' '#'
    printf "%${empty}s" | tr ' ' '-'
    printf "] %3d%%" "$pct"
}

for i in $(seq 1 100); do
    progress_bar $i 100
    sleep 0.05
done
echo
```

### Validate Input
```bash
# Check if number
is_number() {
    [[ "$1" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]
}

# Check if integer
is_integer() {
    [[ "$1" =~ ^-?[0-9]+$ ]]
}

# Check if IP address
is_ip() {
    [[ "$1" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]
}

# Check if email
is_email() {
    [[ "$1" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

# Check if empty
is_empty() {
    [[ -z "${1// /}" ]]   # also catches spaces-only
}
```

---

## 42. QUICK REFERENCE CHEAT SHEET

### Test Operators at a Glance
```
String:   -z  -n  ==  !=  <  >
Integer:  -eq  -ne  -lt  -gt  -le  -ge
File:     -e  -f  -d  -L  -r  -w  -x  -s  -nt  -ot
Logic:    &&  ||  !
```

### Parameter Expansion Cheat Sheet
```
${var}                Variable value
${var:-default}       Default if unset/empty
${var:=default}       Assign default if unset/empty
${var:+alternate}     Alternate if set
${var:?error}         Error if unset/empty
${#var}               String length
${var:offset:length}  Substring
${var#pattern}        Remove shortest prefix
${var##pattern}       Remove longest prefix
${var%pattern}        Remove shortest suffix
${var%%pattern}       Remove longest suffix
${var/old/new}        Replace first match
${var//old/new}       Replace all matches
${var/#old/new}       Replace prefix
${var/%old/new}       Replace suffix
${var^}               Uppercase first char
${var^^}              Uppercase all
${var,}               Lowercase first char
${var,,}              Lowercase all
${!prefix*}           Variable names matching prefix
${!name}              Indirect reference
${var@Q}              Quoted value
${var@a}              Attributes
```

### Brace Expansion
```bash
echo {1..10}           # 1 2 3 4 5 6 7 8 9 10
echo {a..z}            # a b c ... z
echo {01..10}          # 01 02 03 ... 10 (zero-padded)
echo {1..20..2}        # 1 3 5 7 9 11 13 15 17 19
echo file{A,B,C}.txt   # fileA.txt fileB.txt fileC.txt
mkdir -p project/{src,bin,lib,doc}
cp file.txt{,.bak}     # cp file.txt file.txt.bak
```

### Arithmetic Reference
```
$(( expression ))
(( expression ))       # returns true/false
let "expression"
expr expression
```

### Exit Code Meanings
```
0   — Success
1   — General error
2   — Misuse of command
126 — Cannot execute
127 — Command not found
128 — Invalid exit argument
130 — Ctrl+C
137 — SIGKILL (128+9)
143 — SIGTERM (128+15)
```

---

*Complete Shell Scripting Reference — All 42 Topics*
*Last updated: April 2026*
