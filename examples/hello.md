In this example we'll be creating a virtual filesystem with a single directory at the base containing a single file, "hello.txt". The contents of this file will be set as a string in our program.

Add the latest version of `fuser` to your  `Cargo.toml`. `libc` is also required to return error codes.

```rs
[dependencies]
fuser = "0.11.0"
libc = "0.2"
```

Include the following crates in your `main.rs`.

```rs
use fuser::{
    FileAttr, FileType, Filesystem, MountOption, ReplyAttr, ReplyData, ReplyDirectory, ReplyEntry, Request,
};
use libc::ENOENT;
use std::ffi::OsStr;
use std::time::{Duration, UNIX_EPOCH};

const TTL: Duration = Duration::from_secs(1); // 1 second
```

We need to create a directory at the base of our filesystem. Specify the attributes of the directory using the `FileAttr` struct. We'll use boilerplate values, but note that your program will need to manually manage all file attributes.

```rs
const HELLO_DIR_INO: u64 = 1;
const HELLO_DIR_ATTR: FileAttr = FileAttr {
    ino: HELLO_DIR_INO,
    size: 0,
    blocks: 0,
    atime: UNIX_EPOCH, // 1970-01-01 00:00:00
    mtime: UNIX_EPOCH,
    ctime: UNIX_EPOCH,
    crtime: UNIX_EPOCH,
    kind: FileType::Directory,
    perm: 0o755,
    nlink: 2,
    uid: 501,
    gid: 20,
    rdev: 0,
    flags: 0,
    blksize: 512,
};
```

Now let's create a filesystem struct called `HelloFS`. The `Filesystem` trait allows us to implement various standard filesystem functions. We'll first implement `getattr`, the minimum necessary function for a file to "exist".

```rs
struct HelloFS;

impl Filesystem for HelloFS {
    fn getattr(&mut self, _req: &Request, ino: u64, reply: ReplyAttr) {
        match ino {
            HELLO_DIR_INO => reply.attr(&TTL, &HELLO_DIR_ATTR),
            _ => reply.error(ENOENT),
        }
    }
}
```

We can mount our create filesystem as read-only at the directory `/mnt/hellofuser`. 

```rs
fn main() {
    let mount_point = "/mnt/hellofuser";

    let options = vec![
        MountOption::RO, 
        MountOption::FSName("hello".to_string()),
        MountOption::AutoUnmount,
        MountOption::AllowRoot
    ];

    fuser::mount2(HelloFS, mount_point, &options).unwrap();
}
```

`cargo run` your program. The filesystem will be mounted at `/mnt/hellofuser` on run. You may need to `mkdir` the directory first. Let's check to see that the mount was successful.

```
$ stat /mnt/hellofuser
... 1 drwxr-xr-x 2 youruser yourgroup 0 0 "Dec 31 16:00:00 1969" ...
```

Now let's create a single file in this directory called "hello.txt". We again set its attributes using the `FileAttr` struct.

```rs
const HELLO_TXT_NAME: &str = "hello.txt";
// String we want "hello.txt" to contain when read
const HELLO_TXT_CONTENT: &str = "Hello World!\n";

const HELLO_TXT_INO: u64 = 2;
const HELLO_TXT_ATTR: FileAttr = FileAttr {
    ino: HELLO_TXT_INO,
    size: 13, // Content length
    blocks: 1,
    atime: UNIX_EPOCH,
    mtime: UNIX_EPOCH,
    ctime: UNIX_EPOCH,
    crtime: UNIX_EPOCH,
    kind: FileType::RegularFile,
    perm: 0o644,
    nlink: 1,
    uid: 501,
    gid: 20,
    rdev: 0,
    flags: 0,
    blksize: 512,
};
```

Update `HelloFS` to return the file attributes. The new function, `lookup`, returns attributes for file lookups in directories only.

```rs
impl Filesystem for HelloFS {
    fn getattr(&mut self, _req: &Request, ino: u64, reply: ReplyAttr) {
        match ino {
            HELLO_DIR_INO => reply.attr(&TTL, &HELLO_DIR_ATTR),
            HELLO_TXT_INO => reply.attr(&TTL, &HELLO_TXT_ATTR), // Now return file attrs
            _ => reply.error(ENOENT),
        }
    }
    
    fn lookup(&mut self, _req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry) {
        if parent == HELLO_DIR_INO && name.to_str() == Some(HELLO_TXT_NAME) {
            reply.entry(&TTL, &HELLO_TXT_ATTR, 0);
        } else {
            reply.error(ENOENT);
        }
    }
}
```

If we `$ stat /mnt/hellofuser/hello.txt` the file will be shown to exist, but we won't see it in the result of `$ cd /mnt/hellofuser`. We still need to implement `readdir` to allow iteration over the directory.

```rs

impl Filesystem for HelloFS {

    ...

    fn readdir(
        &mut self,
        _req: &Request,
        ino: u64,
        _fh: u64,
        offset: i64,
        mut reply: ReplyDirectory,
    ) {
        if ino != HELLO_DIR_INO {
            reply.error(ENOENT);
            return;
        }

        let entries = vec![
            (HELLO_DIR_INO, FileType::Directory, "."),
            (HELLO_DIR_INO, FileType::Directory, ".."),
            (HELLO_TXT_INO, FileType::RegularFile, HELLO_TXT_NAME),
        ];

        for (i, entry) in entries.into_iter().enumerate().skip(offset as usize) {
            // i + 1 means the index of the next entry
            // ino, offset, file type, file name
            if reply.add(entry.0, (i + 1) as i64, entry.1, entry.2) {
                break;
            }
        }
        reply.ok();
    }
}
```

The directory contents should now be be visible.

```
$ ls -lah /mnt/hellofuser
drwxr-xr-x    2 youruser  yourgroup     0 Dec 31  1969 .
drwxr-xr-x+ 128 youruser  yourgroup  4096 May  1 21:18 ..
-rw-r--r--    1 youruser  yourgroup    13 Dec 31  1969 hello.txt
```

The last step is to allow programs to `read` the contents of `hello.txt`.

```rs
impl Filesystem for HelloFS {

    ...

    fn read(
        &mut self,
        _req: &Request,
        ino: u64,
        _fh: u64,
        offset: i64,
        _size: u32,
        _flags: i32,
        _lock: Option<u64>,
        reply: ReplyData,
    ) {
        if ino == HELLO_TXT_INO {
            reply.data(&HELLO_TXT_CONTENT.as_bytes()[offset as usize..]);
        } else {
            reply.error(ENOENT);
        }
    }
}
```

Now we have a complete, read-only `fuser` filesystem.

```
$ cat /mnt/hellofuser/hello.txt
Hello World!
```
