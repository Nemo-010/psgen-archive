The stub's entry point accepts any inherited fd 3 whose first eight bytes are `PGCFG021`, and the config it reads can name an arbitrary `self_copy` to exec and an arbitrary `temp` directory to write into and `chown`. Under a `sudo` policy that preserves fd 3, or a setuid installation, that is code execution and a root file-write primitive for whoever can control that descriptor.

## What happens

`src/stub.rs`:

```rust
pub fn maybe_run() {
    let cfg = match read_config(3) {
        Some(c) => c,
        None => return,
    };
    run(cfg);
}
```

`read_config()` validates lengths and counts, and then accepts everything else:

```rust
if comm_len > 64 || exe_len > 4096 || cmd_len > 4096 || user_len > 64 || temp_len > 4096 {
    return None;
}
```

and `run()` acts on it:

- `cfg.self_copy` is `execve`'d (`src/stub.rs`, `child_entry` and the `reexec_copy` path),
- `cfg.exe` is `execve`'d,
- `write_disk()` creates `cfg.temp` and a `data.bin` inside it, then `chown`s it to `cfg.uid`/`cfg.gid`.

So a config on fd 3 gives the caller: exec an arbitrary binary, and create an arbitrary directory with an attacker-chosen owner.

`sudo` closes file descriptors above 2 by default, so this is not a default escalation path. It becomes reachable when:

- a sudoers rule grants `psgen` with `closefrom_override` and the caller passes `-C 3`,
- `psgen` is installed setuid-root (nothing in the tree does this, but nothing warns against it either),
- a supervisor or wrapper inherits fd 3 into `psgen` by accident.

## What makes it a real finding rather than a theoretical one

The tool is *designed* to be handed to `sudo` (`--elevate` re-execs through
`sudo -n`), and the README's Limitations section already discusses a forged pidfile
being trusted input. fd 3 is the same class of trust, one level lower, and the
consequences are much larger: a forged pidfile can signal processes the caller may
signal anyway; a forged config runs code.

## Options

1. **Per-run nonce in argv.** The injector generates 16 hex characters, puts them in
   both argv and the config, and `maybe_run()` compares them. An attacker who can
   pass fd 3 but cannot choose argv cannot match it. Smallest change; ~10 lines.
2. **Sealed memfd.** `memfd_create` with `MFD_ALLOW_SEALING`, then `F_ADD_SEALS`
   with `F_SEAL_WRITE | F_SEAL_SHRINK | F_SEAL_SEAL` after writing. `maybe_run()`
   checks the seals with `F_GET_SEALS`. Stronger (the config cannot be mutated after
   the check) but needs the memfd path, which some sandboxes deny — and on those the
   temp-file fallback is used, so the check must handle both.
3. **Environment marker.** Cheap but wrong: the environment is as attacker-controlled
   as the descriptor.

I would do 1 and 2 together: the nonce is what makes the check meaningful on the
temp-file fallback, and the seals are what make it meaningful on the memfd path.

## Also worth doing

`maybe_run()` is called from `main()` before anything else. If the process is
running with `euid == 0` and `uid != 0` (i.e. it *is* setuid), it should refuse to
run the stub path entirely — a setuid install is not a supported configuration and
should fail loudly rather than become an escalation primitive:

```rust
pub fn maybe_run() {
    if sys::geteuid() == 0 && sys::getuid() != 0 {
        // A setuid psgen would let a forged fd 3 config run code as root.
        sys::err(b"psgen: refusing to run setuid; install it without the setuid bit\n");
        sys::exit(2);
    }
    ...
}
```

## Test that would have caught it

```python
# A config on fd 3 must not be honoured unless the injector made it.
cfg = build_config(temp="/tmp/attacker-target", self_copy="/bin/sh")
with open(os.devnull, "w") as devnull:
    p = subprocess.run([BIN], pass_fds=(3,), preexec_fn=lambda: os.dup2(cfg_fd, 3),
                       timeout=5, capture_output=True)
if p.returncode == 0:
    fail("an inherited fd 3 config was executed")
```

(A config with the right magic and a `self_copy` is easy to build from the struct
layout; the point is that the test is possible without root.)

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11. Verified by reading
`read_config()`/`run()`/`write_disk()`; the exploit itself was not executed here
because the cage has no path to a privileged `psgen`, and building one would be a
privilege-escalation demonstration outside the scope of a bug report.
