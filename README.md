# bash-progress-bar

A pure bash progress bar library that renders directly to the terminal using ANSI escape sequences. No external dependencies, no ncurses, no Python, just native bash and terminal control codes.

## Philosophy

This library embraces the Unix philosophy: do one thing well. It provides a clean, minimal API for showing progress without cluttering your script's output. The progress bar lives at the bottom of your terminal, your script's output flows normally above it, and everything just works.

## Features

- **Pure bash**: No external dependencies beyond standard Unix utilities (`stty`)
- **Non-invasive**: Progress bar stays at the bottom, your output scrolls above it
- **Terminal-aware**: Automatically adapts to terminal size changes
- **Pipe-friendly**: Detects when output is redirected and gracefully degrades
- **Lightweight**: ~100 lines of bash, no bloat

## Installation

```bash
# Clone the repository
git clone https://github.com/DeathBlade-01/Progress-bar-bash.git

# Source it in your script
. /path/to/bash-progress-bar/progress-bar-mod
```

Or copy `progress-bar-mod` directly into your project's `lib/` directory.

## Quick Start

```bash
#!/usr/bin/bash
. ./progress-bar-mod

# Initialize the terminal (sets up scroll region)
init_term

# Your work loop
for ((i=0; i<=100; i++)); do
    echo "Processing item $i"
    progress-bar "$i" 100
    sleep 0.1
done

# Clean up (restores terminal state)
deinit_term
```

## API Reference

### `init_term`

Initializes the terminal for progress bar display.

**What it does:**
- Creates a reserved line at the bottom of the terminal
- Sets up a scroll region for your normal output
- Preserves cursor position

**Must be called before** any `progress-bar` calls.

```bash
init_term
```

### `progress-bar <current> <total>`

Renders the progress bar.

**Parameters:**
- `current`: Current progress value (integer)
- `total`: Total/maximum value (integer)

**Example:**
```bash
progress-bar 45 100  # Shows 45% complete
```

**Visual output:**
```
[||||||||||||||||||||............................]45%
```

### `deinit_term`

Restores the terminal to its original state.

**What it does:**
- Removes the scroll region restriction
- Clears the progress bar line
- Restores cursor position
- Moves to a new line

**Should be called** when progress is complete or on script exit.

```bash
deinit_term
```

## How It Works: The Native Approach

This library uses ANSI escape sequences to directly manipulate the terminal. Here's what makes it "native":

### Terminal Control Sequences

The progress bar uses these ANSI/VT100 control codes:

- `\e7` / `\e8`: Save and restore cursor position
- `\e[<line>;<col>H`: Move cursor to specific position
- `\e[<top>;<bottom>r`: Set scrollable region
- `\e[0K`: Clear from cursor to end of line
- `\e[1A`: Move cursor up one line

### The Scroll Region Trick

The key insight is the scroll region (`\e[<top>;<bottom>r`):

1. **Init**: Set scroll region to lines 0 to (terminal_height - 1)
   - Your output now scrolls in this restricted area
   - The bottom line is "protected"

2. **Progress**: Jump to bottom line, draw progress, jump back
   - Output continues scrolling above
   - Progress bar stays fixed at bottom

3. **Deinit**: Reset scroll region to full terminal
   - Restore normal terminal behavior

### Pipe Detection

When output is redirected (`> file` or `| command`), the library detects this:

```bash
term_size=$(stty size </dev/tty 2>&1) || return
```

If `/dev/tty` is unavailable, functions return early—no broken pipes, no errors.

### Dynamic Terminal Sizing

Unlike libraries that cache `$LINES` and `$COLUMNS`, this reads terminal size on every call:

```bash
term_size=$(stty size </dev/tty 2>&1)
term_lines=${term_size%% *}
term_cols=${term_size##* }
```

This means the progress bar adapts if you resize your terminal mid-execution.

## Complete Example

```bash
#!/usr/bin/bash
. ./progress-bar-mod

process_files() {
    local files=("$@")
    local total=${#files[@]}
    
    init_term
    
    for i in "${!files[@]}"; do
        echo "Processing: ${files[$i]}"
        
        # Simulate work
        sleep 0.5
        
        progress-bar "$((i + 1))" "$total"
    done
    
    deinit_term
}

# Example usage
process_files file1.txt file2.txt file3.txt file4.txt file5.txt
```

**Output:**
```
Processing: file1.txt
Processing: file2.txt
Processing: file3.txt
Processing: file4.txt
Processing: file5.txt
[||||||||||||||||||||||||||||||||||||||||||||||||]100%
```

## Advanced Usage

### With Error Handling

```bash
#!/usr/bin/bash
. ./progress-bar-mod

main() {
    trap deinit_term EXIT ERR INT TERM
    
    init_term
    
    for ((i=0; i<=100; i++)); do
        # Your code here
        [[ $i -eq 50 ]] && exit 1  # Simulate error
        
        progress-bar "$i" 100
    done
    
    deinit_term
    trap - EXIT
}

main
```

The `trap` ensures terminal cleanup even if your script exits unexpectedly.

### Debug Messages

The library includes a `debug` function that writes to stderr:

```bash
debug "Starting processing..."
```

Debug messages appear above the progress bar and don't interfere with it.

### Custom Progress Characters

Want to customize the bar appearance? Edit these lines in `progress-bar`:

```bash
# Default: [||||||||..........]
s+='|'  # Progress character
s+='.'  # Remaining character

# Custom: [########          ]
s+='#'  # Progress character
s+=' '  # Remaining character
```

## Technical Details

### Why `/dev/tty` Instead of stdout?

The progress bar writes to `/dev/tty` instead of stdout:

```bash
} >/dev/tty 2>&1
```

**Reasons:**
1. Allows progress bar to work even when stdout is redirected
2. Progress bar always appears on the actual terminal
3. Your script's output can be piped while progress shows on screen

### Why stderr for Progress Messages?

Notice that progress output goes to stderr, not stdout:

```bash
printf "%s" "$s">&2
```

**This is crucial for library behavior:**
- Your calling script can write to stdout without collision
- Progress visualization doesn't pollute your data output
- Enables clean separation: data goes to stdout, UI goes to stderr

**Example of why this matters:**
```bash
# Your script generates a BMP file to stdout (Refer to the authors github page for sprite project)
make_bmp() {
    init_term
    for ((i=0; i<100; i++)); do
        bmp-rgb "$r" "$g" "$b"     # Binary data → stdout
        progress-bar "$i" 100       # Progress UI → stderr (via /dev/tty)
    done
    deinit_term
}

make_bmp > output.bmp  # Clean binary output, progress shows on terminal
```

Without stderr separation, progress bar characters would corrupt your binary output!

### Differences from `progress-bar`

This library (`progress-bar-mod`) is an improved version of the original `progress-bar` script:

**Key improvements:**

1. **No dependency on `$LINES` and `$COLUMNS`**
   - Original: Relied on bash's `$LINES`/`$COLUMNS` variables
   - Modified: Calls `stty size </dev/tty` directly every time
   - Why: `$LINES`/`$COLUMNS` only update between commands, can be stale in tight loops

2. **Proper `/dev/tty` handling**
   - Original: Used shell's stdout/stderr directly
   - Modified: Explicitly routes to `/dev/tty` for all terminal operations
   - Why: Works correctly when script output is redirected

3. **Graceful degradation**
   - Original: Could error if terminal unavailable
   - Modified: Returns early if `stty` fails (e.g., in cron jobs, non-TTY environments)
   - Why: Library doesn't break your script in automated environments

4. **Production-ready**
   - Original: Demonstration/prototype script with test `main()` function
   - Modified: Pure library with no executable code, just functions
   - Why: Safe to source in any script without side effects

**When to use which:**
- **`progress-bar`**: Educational reference, understanding the concepts
- **`progress-bar-mod`**: Production use, sourcing in real scripts

### Why No `$LINES` and `$COLUMNS`?

Bash's `$LINES` and `$COLUMNS` are only updated when:
- A command finishes
- The shell receives `SIGWINCH`

In a tight loop, these can be stale. Direct `stty size` calls ensure accuracy.

### Performance Considerations

Calling `stty size` on every update has minimal overhead:
- `stty` is a simple syscall wrapper
- Modern terminals handle escape sequences efficiently
- For 1000 iterations, overhead is ~10-50ms total

If you're updating thousands of times per second, consider calling `progress-bar` less frequently:

```bash
if ((i % 10 == 0)); then
    progress-bar "$i" "$total"
fi
```

## Compatibility

**Tested on:**
- bash 4.0+
- Linux (any modern distribution)
- macOS (bash or zsh in bash mode)
- WSL/WSL2

**Requirements:**
- bash
- `stty` command (standard on all Unix systems)
- Terminal with ANSI escape sequence support

**Not compatible with:**
- sh/dash (uses bash-specific features)
- Windows cmd.exe (use WSL instead)
- Terminals without ANSI support (very rare nowadays)

## Troubleshooting

### Progress bar doesn't appear

**Check:** Is output redirected?
```bash
./script.sh > output.txt  # Progress bar won't show (by design)
```

**Solution:** Progress shows on terminal only. Check terminal directly.

### Progress bar appears but output is garbled

**Check:** Did you call `init_term`?
```bash
# Wrong
progress-bar 1 100  # Missing init_term

# Correct
init_term
progress-bar 1 100
deinit_term
```

### Terminal is messed up after script exits

**Check:** Did you call `deinit_term`?

**Solution:** Always use `trap`:
```bash
trap deinit_term EXIT
```

**Emergency fix:**
```bash
reset  # Restores terminal to sane state
```

## Why This Exists

Most bash progress bar solutions either:
1. Use external tools (Python, Perl)
2. Pollute output with repeated lines
3. Don't handle terminal resizing
4. Break when output is redirected

This library solves all of these by embracing terminal control sequences and proper scroll region management. It's how terminals were meant to be used.

## License

MIT License

## Contributing

Contributions welcome! This library is intentionally minimal—if you're adding features, consider whether they belong in the core or as examples.

**Guidelines:**
- Keep it pure bash
- No external dependencies
- Maintain backward compatibility
- Document ANSI sequences used

## Credits

Inspired by the Unix philosophy and decades of terminal programming tradition.

Created with ❤️ for shell scripters who appreciate doing things the native way.
