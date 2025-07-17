# From std to no_std

We have successfully flashed and run our first program, which creates a blinking effect. However, we have not yet explored the code or the project structure in detail. In this section, we will recreate the same project from scratch instead of using the template. I will explain each part of the code and configuration along the way. Are you ready for the challenge?


## Create a Fresh Project

We will start by creating a standard Rust binary project. Use the following command:

```rust
cargo new std_to_no_std
```

At this stage, the project will contain the usual files as expected.

```sh
├── Cargo.toml
└── src
    └── main.rs
```

Our goal is to reach the following final project structure:

```sh
├── build.rs
├── .cargo
│   └── config.toml
├── Cargo.toml
├── rust-toolchain.toml
├── src
│   ├── bin
│   │   └── main.rs
```

