# 5
Project 5 introduces a primitive module system and a linker to support it.

## Work Completed
Checkpoint commit for this phase is [`538eb031d77d44ed7fe6cc28f56c35aee624c410`](https://github.com/bferris413/chasm-toolchain/tree/538eb031d77d44ed7fe6cc28f56c35aee624c410).

- Module imports:
    ```
    ; main.cas
    ; ...

    ; make `other-module` public definitions and labels accessible
    import!(other-module)
    ```

- Public definitions and labels:
    - The syntax for definition references (those with `$`) is admittedly noisy. I'll probably revisit it.
    ```
    ; module.cas
    define32-pub!(some-special-addr, xAABB.CCDD)
    define16-pub!(some-thing, xAABB)
    define8-pub!(some-byte, xAA)

    @pub my-public-label
    @pub my-other-label
    @my-private-label
    ```
    ```
    ; main.cas
    import!(module)

    ; sizes for pseudo definitions are explicitly declared
    $32:module::some-special-addr
    $16:module::some-thing
    $8:module::some-byte

    ; sizes for labels are pointer sized
    &module::my-public-label
    &module::my-other-label
    ```
- A hybrid binary/human-readable format for compiled modules:
    - I wanted something with a human-readable header that would enable me to debug without writing a custom `objdump` like
    program right off the bat. 
    - I also didn't want to waste a _ton_ of space making all compiled code human readable.
    - Given the constaints, I ended up with a hybrid format containing three parts (for now):
        1. Binary prelude: 
            ```
            CHASM MODULE\n
            version 1\n
            headerlen <8 byte length>\n\n
            ```
        2. Text header, containing the serialized fields of an [`AssemblyModule`](https://github.com/bferris413/chasm-toolchain/blob/538eb031d77d44ed7fe6cc28f56c35aee624c410/chasm/src/assemble/mod.rs#L451-L459) (currently JSON until I have a good reason to switch.
        I tried TOML, not a fan of how it presents nested structures):

            ```json
            {
                "modname": "test-module",
                "imports": [
                    "other-mod",
                    "my-mod"
                ],
                "definitions": {
                    "CONST": {
                        "U16": 4660
                    }
                },
                "pub_definitions": {
                    "START": {
                        "U32": 536870912
                    },
                    "OTHER": {
                        "U32": 74565
                    }
                },
                "labels": {
                    "PRIVATE": [
                        false,
                        70000
                    ],
                    "PUBLIC": [
                        true,
                        0
                    ]
                },
                "linker_patches": [
                    {
                        "LabelNewOffset": {
                            "patch_at": 100,
                            "patch_size": "U16",
                            "unpatched_value": 50
                        }
                    },
                    {
                        "ImportLabelRef": {
                            "patch_at": 200,
                            "patch_size": "U32",
                            "import_module": {
                                "module": "my-mod",
                                "member": "LABEL"
                            }
                        }
                    }
                ]
            }
            \n\n
            ```
        3. The module's code, followed by \n\n.
- A linker, with compiler driver support for linking both at build time and as a standalone step:
    - Having imports and cross-module references implies the existence of a tool to resolve them, so here we have it.
    - `chasm assemble` will link the provided assembly modules into a single binary by default.
    - if `--no-link` is specified, the modules will be serialized and written to disk.
    - `chasm link` takes a list of paths pointing to serialized assembly modules and links them into a single binary.


## Next Steps
Adding the module system was a significant step for Chasm and I'm overall pretty happy with the result. I'm sure there are things
that will need cleanup or refactoring, but for now it's working as intended. Now that we have it in place there's a couple directions
to go:
- Venture back to the embedded side of things and integrate with more of the hardware. The TM4C123GXL kit I have has some pluggable
modules, including a screen, that I may try to draw simple graphics on.
- An editor. I _really_ want the Chasm development experience to be 100% focused. After developing some plugins for VS Code during
another project, I'd like to explore the idea of not using Microsoft's editor and instead build my own.
- The high-level Chasm language. Probably won't tackle this yet, but some ideas are cooking.