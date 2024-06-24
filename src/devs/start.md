<div class="warning">

[Chinese PumpBin Documentation is here](https://pumpbin.github.io/book-zh/devs/start.html)

</div>

# Quick Start

In this chapter, we will create a `Local` type Plugin and understand how PumpBin works.

Plugins of type `Local` have the shellcode stored inside the final implant. This method has certain stability advantages, but the shellcode is easily extracted and analyzed, which makes memory scanning impossible.

PumpBin will read the shellcode file and dynamically patch the encrypted shellcode into the binary implant template according to the encryption settings. The random password used for encryption will also be patched.

So PumpBin is essentially a binary data search and replace tool. This implementation requires that placeholder data be placed in the binary implant in advance, and once the compilation is complete, the length of the placeholder data cannot be changed.

In general, we make the shellcode placeholder data slightly larger than the required length (if you know the length of the shellcode to be used). There are two reasons for this:

1. For greater compatibility (if the Plugin shellcode placeholder is shorter than the encrypted shellcode, the Plugin will not be able to use this shellcode)
1. The extra space is not useless, PumpBin will fill it with random data to ensure that each implant is unique

## Create binary implant template

I will use [create_thread](https://github.com/b1nhack/rust-shellcode/blob/main/create_thread/src/main.rs) from [rust-shellcode](https://github.com/b1nhack/rust-shellcode) as the code to load.

Clone `rust-shellcode`

```sh
git clone --depth 1 https://github.com/b1nhack/rust-shellcode.git
```

Go to the `create_thread` directory

```sh
cd rust-shellcode
cd create_thread
```

Compile and run, test if it loads the demo shellcode successfully\
By default, [w64-exec-calc-shellcode-func.bin](https://github.com/peterferrie/win-exec-calc-shellcode/blob/master/build/bin/w64-exec-calc-shellcode-func.bin) will be run, and you should see the `calc` program start.

```sh
cargo r
```

We need to replace `w64-exec-calc-shellcode-func` with shellcode placeholder data.\
Create build.rs in the `create_thread` directory and paste the following code

```rust
use std::{fs, iter};

fn main() {
    let mut shellcode = "$$SHELLCODE$$".as_bytes().to_vec();
    shellcode.extend(iter::repeat(b'0').take(1024 * 1024));
    fs::write("shellcode", shellcode.as_slice()).unwrap();
}
```

This will generate a placeholder data of about 1 MiB, and it starts with `$$SHELLCODE$$` (we call it `Prefix`, PumpBin uses it to locate the placeholder data, so `Prefix` needs to be unique, PumpBin will use the first match).

Replace line 11 of main.rs with the following code to include the placeholder data

```rust
let shellcode = include_bytes!("../shellcode");
```

Since the shellcode will be filled with random data, we need to know the length of the shellcode to correctly extract the shellcode.

Add the following code after line 11 of main.rs

```rust
const SIZE_HOLDER: &str = "$$99999$$";
let shellcode_len = usize::from_str_radix(SIZE_HOLDER, 10).unwrap();
let shellcode = &shellcode[0..shellcode_len];
```

We added a constant string reference, `$$99999$$` is also a `Prefix`, but we prefer to call it a `Place Holder`, because it will be completely replaced.
(`Prefix` will be completely replaced with valid data + random data, while `Place Holder` will be completely replaced with valid data)

With the length information of the shellcode, we can get the correct shellcode. Compile the modified `create_thread` project, and we will get a simple binary implant template.

```sh
cargo b -r
```

## Creating a Plugin

Now that we have a simple binary implant template, we can use PumpBin Maker to create a Plugin that only contains a Windows Exe

Plugin Name: Enter first_plugin. (This field is the unique identifier for the plugin, which means that users cannot install two plugins with the same name at the same time.)

Prefix: Enter `$$SHELLCODE$$`. (This is the `Prefix` of the shellcode placeholder data we used above. You can use any `Prefix` you like, as long as it is unique or the first one to be matched.)

MaxLen is the total size of the shellcode placeholder data, which in this case should be 1024\*1024 + the size of the prefix = 1048589. (Units are Bytes)

Type: Keep Local selected, Size Holder: Enter `$$99999$$`. (This is the string reference we used above to identify the length of the shellcode. You can use any Size Holder you like. The rules are the same.)

Encrypt Type: Leave as None

Windows Exe Select the binary implant template we compiled above. (You can also directly fill in the file path)

Click Generate, save the generated plugin

## Test the plugin

Install the plugin created by PumpBin and use `w64-exec-calc-shellcode-func` to generate a final implant. You should see the `calc` program start.

At this point, we have created a plugin of type "Local" and understood how PumpBin works.

The next chapter will introduce the encryption system in PumpBin. You don't need to worry about the encryption process anymore. You only need to know that whenever you select a plugin in the plugin list, PumpBin will regenerate a random encryption password for the corresponding plugin for you.
(If you want to regenerate the encryption password for the currently selected plugin, just click on the currently selected plugin in the plugin list again.)

The complete project file for this example is available in the PumpBin code repository under [examples/create_thread](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread/src/main.rs).
