# Process Hollowing in Rust

This repository contains a Rust program demonstrating process hollowing, a technique often used in malware to inject a payload into the address space of another process. The code performs the following steps:

1. Starts a new process in a suspended state.
2. Maps a PE (Portable Executable) file into the address space of the target process.
3. Fixes the import address table (IAT) and base relocations.
4. Changes the entry point of the target process to the entry point of the injected PE.
5. Resumes the main thread of the target process, effectively executing the injected code.

## Dependencies

The program uses several external crates and system libraries, including:

- `winapi`: For Windows API functions.
- `ntapi`: For accessing NT API functions.
- `widestring`: For handling wide strings used in Windows API.

Ensure you include these dependencies in your `Cargo.toml` file:

```toml
[dependencies]
winapi = { version = "0.3", features = ["winuser", "libloaderapi", "processthreadsapi", "errhandlingapi", "memoryapi", "synchapi", "ntdef"] }
ntapi = "0.3"
widestring = "0.4"
```

## Code Structure

### Main Function

The `main` function is the entry point of the program. It:

1. Opens the target executable (`cmd.exe` in this case).
2. Reads the payload executable (`calc2.exe`) into a buffer.
3. Creates a new process in a suspended state.
4. Retrieves the image base address of the target process.
5. Calls `ProcessHollow64` to perform the process hollowing.
6. Resumes the target process thread to execute the injected code.

```rust
fn main() {
    let mut processname = "C:\\Windows\\System32\\cmd.exe\0";
    unsafe {
        let mut buffer: Vec<u8> = Vec::new();
        let mut fd = File::open(r#"D:\red teaming tools\calc2.exe"#).unwrap();
        fd.read_to_end(&mut buffer).unwrap();

        let mut si: STARTUPINFOA = std::mem::zeroed();
        si.cb = std::mem::size_of::<STARTUPINFOA>() as u32;
        let mut pi: PROCESS_INFORMATION = std::mem::zeroed();
        let res = CreateProcessA(
            processname.as_ptr() as *mut i8,
            std::ptr::null_mut(),
            std::ptr::null_mut(),
            std::ptr::null_mut(),
            0,
            0x00000004,
            std::ptr::null_mut(),
            std::ptr::null(),
            &mut si as &mut STARTUPINFOA,
            &mut pi as &mut PROCESS_INFORMATION,
        );

        if res == 0 {
            println!("CreateProcessA failed with error: {}", GetLastError());
            return;
        }

        let imagebase = GetProcessImageBase(pi.hProcess);
        ProcessHollow64(pi.hProcess, imagebase as *mut c_void, buffer, pi.hThread);
        ResumeThread(pi.hThread);
    }
}
```

### `ProcessHollow64` Function

This function performs the core process hollowing steps:

1. Unmaps the view of the section in the target process.
2. Allocates memory in the target process.
3. Copies the headers and sections of the payload into the target process.
4. Fixes the IAT.
5. Fixes base relocations.
6. Sets the context of the main thread to the entry point of the injected code.

### Helper Functions

- `GetProcessImageBase`: Retrieves the base address of the image of the target process.
- `FillStructureFromMemory`: Reads memory from the target process and fills a structure.
- `GetHeadersSize` and `GetImageSize`: Extracts the size of the headers and the entire image from the PE file.
- `ReadStringFromMemory`: Reads a null-terminated string from the memory of the target process.

## Usage

To run this example, ensure you have Rust installed. Clone this repository and navigate to its directory. Then, run:

```sh
cargo run --release
```

**Note:** This code is for educational purposes only. Process hollowing can be used maliciously, and running this code on a production system or without proper authorization can have serious legal and ethical implications.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## Disclaimer

This software is provided "as is", without warranty of any kind. The authors are not responsible for any damage caused by the use of this software. Use at your own risk.
