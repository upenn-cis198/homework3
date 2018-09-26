# homework3
A Rust Shell

## Setup
No code will be given for this assignment. However you will need the following crates:
- termion: A crate for handling terminal information.
- nix: Unix high-level wrappers.
- log: Base crate for logging.
- env_logger: Simple logging crate for Rust.

You will need to familiarize yourself
with [termion](https://github.com/redox-os/termion) and [env_logger](https://docs.rs/env_logger/*/env_logger).

## Assignment
We will be creating a simple shell in Rust. This shell should print a prompt, accept commands by the user, and when the user
presses enter, execute those commands. The output will then be printed to the terminal. (The original assignment allowed the user
to use pipes `|` for parallel output/input redirection among multiple processes, but that was too hard).

A typical session in the shell may look like:
```bash
> ls
Cargo.lock
Cargo.toml
src
target

> more cis198.txt
more: stat of cis198.txt failed: No such file or directory

> pwd
/home/gatowololo/Programming/Rust/RustShellSimplified

> echo "hello class" > cis198.txt

> more cis198.txt
"hello class"

> 
> 
> 
> 
> 
> garbage
ENOENT: No such file or directory

> exit
```

We will be using termion's raw terminal mode to handle characters ourselves. Your terminal must support the backspace key to delete
characters, as well as empty newlines which do nothing but reprint the prompt. Similarly, typing "exit" or "quit" should exit
the terminal, as well as `ctrl-c`.

### Input
This program takes no command line arguments as input. Once it is running, it should clear the terminal, and print a prompt "> "
before every line.

If a command fails to run, the appropriate error should be printed to the terminal the the shell continued (as seen in the above
example with the command "garbage").

Your program should support stdout redirection if `> some_file.txt` is present. You can assume this command will always be well
formed, so you can simply .split('>') to get both parts of the command. (Otherwise it's a pain to properly parse commands like
this).

Any other input e.g. tab, alt+K, should simply be logged and ignored.

### Output
The program can simply output the results of running a command directly to stdout/stderr.
If the user specified *some_file* through stdout redirection, the results of stdout should go to that file instead.
stderr can always be printed to the terminal.

## Design and implementation

### Termion
Programs usually deffer the job of dealing with tabs, newlines, user input, etc, to the shell/terminal. In this case we will have
to handle this explicitly. Please use [termion raw mode](https://docs.rs/termion/1.5.1/termion/raw/index.html) for this task.

To facilitate it's use we will create a wrapper type around this raw terminal. Our type `struct RawTerminal` will live in it's
own file and module `raw_terminal.rs`:

```rust
/// Object representing the cursor. Allows transparent creation of new lines.
pub struct RawTerminal {
    y_position: u16,
    stdout: termion::raw::RawTerminal<Stdout>,
}
```

It is your job to implement several methods around this type:
```rust
/// Creates a new RawTerminal and changes the terminal to raw mode.
pub fn new() -> Self;
/// Write a string to the terminal.
pub fn write(&mut self, msg: &str);
/// Write a string ending with a newline to the terminal.
pub fn writeln(&mut self, msg: &str);
/// Write a new line to the terminal.
pub fn newline(&mut self);
/// Move the cursor to the beginning of line.
pub fn move_to_beginning_of_line(&mut self);
/// Write a message to our output log. (See next section).
pub fn debug(&mut self, msg: &str);
/// Flush results. Results are not written to the terminal immediately, so we flush after a command to write our
/// output.
fn flush(&mut self);
```
This way, in main I can write code like:
```rust
// Enter raw mode.
let mut terminal = RawTerminal::new();
terminal.write(&"> ");
terminal.write(&line);
```

You may find this [termion blog](https://ticki.github.io/blog/making-terminal-applications-in-rust-with-termion/) useful for
understanding how it all works.

### env_log and Testing
As you might have noticed from last assignment, testing code that interacts heavily with the OS can be trickle. Therefore, it is
nice to have a logger which writes important events when debugging is enabled. `env_log` those this for us.

However, this doesn't work too well without manually flushing the output. So please wrap env_logger's debug!() macro in the
RawTerminal `debug` method (see last section).

Please print important debugging events to the command line when debugging is enabled. It is up to you which events you chose
to log. Some examples might be: Which key has been pressed, what command the user ran, the return code of the command, etc.

It is okay, if the debug output is jumbled in and unaligned, as it is too tedious to get right, the simpler approach would be
to simply write to a file, but we want to use env_logger for this assignment.

Feel free to keep unit testing very light if at all.

### Nix
Nix is a Rust crate wrapping libc with higher level, ergonomic code. Please use libc to interact spawn a separate process for
execution of the command, specifically, use `fork`, `waitpid`, and `execvp`, `pipe`, `close`, `dup2`, for running subcommands. **ERROR: DO NOT** use
Struct std::process::Command.

### Dealing with errors 
Propagating all errors up to main() quickly becomes intrusive to our function signatures and code. We should only trickle errors up
when there is actually something we can do about it, or want to continue the program. Otherwise it is much easier to simply print
out a helpful error message and exit the program.

Please do so at your discretion.

### Useful functions
You may find `as_raw_fd()`, `read_to_string()`, `File::from_raw_fd()`, `File write_all()`, and str `as_bytes()` useful.

### Unsafe
One of the functions you have is marked as unsafe. Therefore you have to wrap it in a unsafe {... } block to use. Notice the
unsafe block is like any other block in Rust, and values can be returned as expressions "up" a block.

## Recommended Steps for Implementation.
1) First implement the raw terminal functionality by using the raw terminal functions directly. That is, create a shell that
is able to handle backspace, key presses, printing prompt, and when the user presses enter, it merely prints to the command
like the command as a string.
2) If #1 is working, then factor out your code into a RawTerminal type.
3) Now add the functionality to actually execute the process using execvpe. At first, do not use pipes at
all, this way, output will just be written to the terminal directly.
4) Once you're able to call commands and have their output printed to the shell, work on getting '>' stdout redirection using
pipes working. Everything else should work before you get here.

## Future Thoughts
For the sake of simplicity, this is all. However, you should see how it is possible to extend the terminal functionality,
in all sorts of ways: multiple command redirection using '|' pipes, to auto-completion of commands, color highlighting,
a command history, etc.
