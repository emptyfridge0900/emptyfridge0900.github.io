+++
title="Package manager"
date=2025-07-21

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

`rust-analyzer` had been working fine until a few days ago, but suddenly stopped working, so I checked where it was installed. I remember struggling with a `rust-analyzer` issue before too, and I expect something similar may happen again, so I wrote down the commands I used today.

First, check where `rust-analyzer` is installed.

```bash
which package_name
which rust-analyzer
```

The install location shows up as `~/.cargo/bin/rust-analyzer`.

## cargo

#### List installed programs

```bash
cargo install --list
```

I checked installed programs with `cargo install --list`, but `rust-analyzer` was not there.

Then check the directory where cargo-installed programs are placed.

#### Program install location

```bash
ls ~/.cargo/bin/
```

In `.cargo/bin`, `rust-analyzer` really exists, but the filename has `@` at the end. That means it is a symbolic link. In simple terms, it is like a shortcut on Windows.

#### Find the real location behind a symbolic link

```bash
readlink -f path
readlink ~/.cargo/bin/rust-analyzer
```

This shows that the real location of `rust-analyzer` is managed by `rustup`.

## rustup

If you check the default location of `rustup` with **which rustup**, it also shows `~/.cargo/bin`. I am not sure why, but that is not the real location; it is just where symbolic links are gathered.

For reference, rustup metadata and toolchains live under `~/.rustup`.

Maybe I previously installed `rust-analyzer` with rustup, removed it, then installed it again with Homebrew, leaving a link behind in `.cargo/bin`.

So I decided to install `rust-analyzer` with rustup again and remove the Homebrew-installed one.

#### Install a program

```bash
rustup component add package_name
rustup component add rust-analyzer
```

Where are packages installed through `component add`?

They are in **`~/.rustup/toolchains/<toolchain>/bin`**. In my case, it was under:

```text
~/.rustup/toolchains/stable-aarch64-apple-darwin/lib
```

## Homebrew

First, check whether Homebrew has `rust-analyzer`.

```bash
brew list
```

`rust-analyzer` is in the list. But I still do not know why `which rust-analyzer` showed `.cargo`, or why `rust-analyzer` had been working fine until now. Maybe it was related to a recent `brew upgrade`.

Anyway, the actual binary for programs installed with **brew install** should be in one of the following directories.

```bash
ls /opt/homebrew/bin
ls /opt/homebrew/opt
ls /opt/homebrew/Cellar
```

Now remove the `rust-analyzer` installed through Homebrew.

```bash
brew uninstall package_name
brew uninstall rust-analyzer
```

After opening a Rust file in Helix, it finally worked correctly again.
