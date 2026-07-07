# mysh — a tiny shell, built to understand how shells really work

This is my own imagination-driven implementation of a Unix shell, called **mysh**.

I didn't write this to replace `bash` or `zsh` — I wrote it to actually *understand* what a shell does under the hood: how it reads your command, how it decides whether to run a built-in or hand things off to the operating system, how variables and history work, and so on. Reading about shells is one thing; writing a small one yourself is what makes it click.

---

## What it does

When you run `mysh`, you get a prompt like:

```
mysh:~/projects$ 
```

From there you can type commands, just like in a normal shell. Two kinds of commands are supported:

- **Built-in commands** — handled directly inside mysh itself (`cd`, `pwd`, `exit`, `set`, `unset`, `echo`, `help`, `history`)
- **External commands** — anything else (like `ls`, `cat`, `whoami`) gets handed off to the actual operating system to run

---

## Building it

You need `gcc` (or any C compiler) on a Linux/Unix system.

```bash
gcc my_shell.c -o mysh
```

## Running it

```bash
./mysh
```

You'll land in the mysh prompt. Type `exit` when you're done.

---

## The built-in commands

### `pwd`
Prints your current working directory.

**Under the hood:** it calls the `getcwd()` system call, which asks the operating system directly "what folder am I standing in right now?" and copies the answer into a buffer.

### `cd [dir]`
Changes the current directory. If you don't pass a directory, it goes to your home folder.

**Under the hood:** this is the classic reason every shell *must* implement `cd` as a built-in rather than an external program. `cd` calls `chdir()`, which changes the working directory of the *current process*. If `cd` were a separate program, it would run in a child process (via `fork`), change *its own* directory, and then exit — leaving the parent shell's directory completely unchanged. So `cd` has to run inside the shell itself to have any lasting effect.

### `exit`
Quits the shell, flushes the history file, and cleans up memory.

### `set VAR=VALUE`
Creates or updates a shell variable.

**Under the hood:** mysh keeps two parallel arrays in memory — one for variable names, one for their values (like a very simple, manual hash map without the hashing). `set` parses out the part before `=` as the name and the part after as the value, checks if that name already exists, and either updates it in place or inserts it into the first empty slot.

### `unset VAR`
Removes a variable.

**Under the hood:** it looks up the variable's slot in the names/values arrays and clears it (zeroes it out), rather than physically removing it from the array — the slot becomes free again for future `set` calls.

### `echo [args]`
Prints text back to you, with support for variable substitution using `$VAR`.

**Under the hood:** it walks through each argument. If an argument starts with `$`, it strips the `$` and looks that name up in the variables array, printing the stored value instead of the literal text. Otherwise, it just prints the argument as-is.

### `help [command]`
Shows a list of all built-in commands, or detailed help for one specific command if you name it.

### `history [N]`
Shows your command history — either everything, or just the last `N` commands.

**Under the hood:** every command you type gets appended to a `history.txt` file (with a line number) the moment you hit enter. When you run `history`, mysh counts how many lines the file has, optionally skips ahead to only show the last `N`, and prints the rest.

### External commands (anything not in the list above)
**Under the hood:** this is where the real magic of "running a program" happens. mysh calls `fork()` to create a child process — an exact copy of mysh itself. In that child, it calls `execvp()`, which replaces the child process's memory entirely with the program you asked for (e.g. `ls`). The parent process (your actual mysh shell) just calls `wait()` and pauses until the child finishes, then takes back control of the prompt. This fork → exec → wait pattern is exactly what real shells like bash do every single time you run a command.

---

## Example session

```
mysh:~$ pwd
/home/username
mysh:~$ set NAME="username"
mysh:~$ echo Hello $NAME
Hello username
mysh:~$ cd projects
mysh:~/projects$ ls
mysh.c  README.md
mysh:~/projects$ history
1 pwd
2 set NAME="username"
3 echo Hello $NAME
4 cd projects
mysh:~/projects$ exit

     This small Shell created by Hovhannes Hovhannisyan
     08.07.2025
```

---

## Why build this?

Because it's easy to *use* a shell every day and never think about what's actually happening — how `cd` works, why some commands feel "instant" while others start a whole new process, or how your shell remembers the commands you typed yesterday. Building even a small, imperfect version of one is one of the best ways to demystify all of that.
