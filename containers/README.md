Fedora image for mdBook-based generation
========================================


This Fedora container image contains:

  - mdBook: https://github.com/rust-lang/mdBook  
    create book from markdown files, like Gitbook
  - mdBook preprocessor: https://crates.io/crates/mdbook-callouts  
    to add Obsidian Flavored Markdown's Callouts


## Usage instructions
Start the container in the folder that contains your documentation source

```bash
$ podman run --rm -v $PWD:/workspace \
    quay.io/crc-org/mdbook:0.4.42 \
    mdbook build
```

This will generate a `book` output.

Or using

```bash
$ podman run --rm -v $PWD:/workspace -p 3000:3000 \
    quay.io/crc-org/mdbook:0.4.42 \
    mdbook serve
```

the generated content will be published using the embedded server on [`http://localhost:3000`](http://localhost:3000)
