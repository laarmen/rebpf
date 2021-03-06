# rebpf
rebpf is a Rust library built on top of libbpf (no bcc dependency) that allows to write and load bpf program, in details this library provides:

- A raw binding of libbpf.
- A safe wrapper of libbpf (Work in progress).
- High level ebpf api built on top of libbpf wrapper (Future work).

## rebpf is not RedBPF
Even if the distance of the name between rebpf and [RedBPF](https://github.com/redsift/redbpf) is very small, this library (rebpf) is a new indipendent project that has nothing to do with RedBPF Rust library.

## Usage
To create your first ebpf program with rebpf library you can copy and rename an [empty project template](https://github.com/uccidibuti/rebpf/tree/master/examples/empty_project) and edit it changing <your_project_name>/src/kern.rs and <your_project_name>/src/user.rs files.

### write your ebpf program
Copy this content in <your_project_name>/src/kern.rs:

```rust
#![no_std]
use rebpf::{xdp::XdpAction, LICENSE, VERSION, rebpf_macro::sec, libbpf::xdp_md};

#[sec("license")]
pub static _license: [u8; 4] = LICENSE;

#[sec("version")]
pub static _version: u32 = VERSION;

#[sec("xdp_drop")]
fn _xdp_drop(ctx: *const _xdp_md) -> XdpAction {
    XdpAction::DROP
}
```
Note: this ebpf program drop every packets received.

### write your ebpf loader program
Copy this content in <your_project_name>/src/user.rs:

```rust
use rebpf::{self, xdp, interface,  error as rebpf_error};
use std::path::Path;

const DEFAULT_FILENAME: &str = "kern.o";
const DEFAULT_DEV: &str = "wlan0"; // replace with your device name

fn load_bpf(interface: &interface::Interface, bpf_program_path: &Path, xdp_flags: &[xdp::XdpFlags]) -> Result<(), rebpf_error::Error> {
    let (_bpf_object, bpf_fd) = rebpf::bpf_prog_load(bpf_program_path, rebpf::BpfProgType::XDP)?;
    xdp::bpf_set_link_xdp_fd(&interface, Some(&bpf_fd), &xdp_flags)?;
    let info = rebpf::bpf_obj_get_info_by_fd(&bpf_fd)?;
    println!("Success Loading\n XDP prog name: {}, id {} on device: {}", info.name()?, info.id(), interface.ifindex());
    
    Ok(())
}

fn unload_bpf(interface: &interface::Interface, xdp_flags: &[xdp::XdpFlags]) -> Result<(), rebpf_error::Error> {
    xdp::bpf_set_link_xdp_fd(&interface, None, &xdp_flags)?;
    println!("Success Unloading.");

    Ok(())
}

fn run(bpf_program_path: &Path, interface_name: &str, unload_program: bool) -> Result<(), rebpf_error::Error> {
    let interface = interface::get_interface(interface_name)?;
    let xdp_flags = vec![xdp::XdpFlags::UPDATE_IF_NOEXIST, xdp::XdpFlags::SKB_MODE];
    if unload_program == false {
        load_bpf(&interface, bpf_program_path, &xdp_flags)
    } else {
        unload_bpf(&interface, &xdp_flags)
    }    
}

fn main() {
    let bpf_program_path = Path::new(DEFAULT_FILENAME);
    let unload_program = false;
    match run(&bpf_program_path, DEFAULT_DEV, unload_program) {
        Err(err) => println!("{:?}", err),
        Ok(_) => {}
    };
}

```

### compile ebpf and loader programs
Move into <your_project_name> folder and run the script build.sh:
```
cd <your_project_name>
./build.sh
```

### load and run ebpf program
```
cd <your_project_name>/ebpf_output
sudo user
```
Expected output:
```
Success Loading
 XDP prog name: _xdp_drop, id 33 on device: 2
```

## Documentations
[link](https://docs.rs/rebpf/0.1.0/rebpf/).

## Requirements
- A recent [linux kernel](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)
- LLVM 9
- libelf
- zlib

## License
Licensed under GNU Lesser General Public License (LGPL), version 3 https://www.gnu.org/licenses/lgpl-3.0.html

## Examples
[link](https://github.com/uccidibuti/rebpf/tree/master/examples).