# Everything Is a File: A Hands-On Lab

## Why this idea is worth an afternoon

Open a text file in Python and you get something you can `read()` from and `write()` to. Now think about how many other things a program needs to talk to: the keyboard, the screen, a disk, another program running next door, a server on the other side of the world, the kernel itself when you want to know how much memory is free. Each of those could have needed its own special set of commands. On Linux, most of them don't. They get a name somewhere in the directory tree, or at least a small number handed to your program, and after that you use the same few operations you already know: open, read, write, close.

That's what people mean by "everything is a file." Once it clicks, a lot of Linux stops looking like a pile of unrelated tricks. `cat /proc/uptime` works for the same reason `cat notes.txt` does. A program that prints to the screen can print into a file or into another program without changing a line of its code. A chat server can read from a network connection with the same `for line in ...` loop you'd use on a text file.

The slogan is an exaggeration, and it's worth knowing where. Your network card doesn't appear under `/dev`. A TCP connection to a web server has no name anywhere on disk; your program just gets a number for it. Processes aren't files, though `/proc` shows a view of each one as a folder. Some people prefer "everything is a file descriptor" as the more accurate version, and that isn't literally true either. The honest summary is this: Linux tries hard to give very different things the same shape, and it succeeds often enough that the shape is worth learning well. This lab shows you where it works, and a few places where it doesn't.

---

## Before you start

**Where to do this.** A GitHub Codespace is ideal. A departmental VM, Ubuntu under WSL, or any Linux machine where you can use `sudo` also works. Everything here was tested on Ubuntu 24.04 as an ordinary user. The prompts below show the user `codespace`; you'll see your own name.

**How to read the commands.** A line starting with `$` is something you type (leave out the `$` itself). Lines without it are what the computer prints back.

```
$ whoami
codespace
```

Your output will differ in the details: inode numbers, process IDs, dates, sizes. What should match is the shape. Where a bigger difference is likely, the lab says so.

**Two terminals.** Parts 5, 7, 8 and 9 need two terminals open side by side. In a Codespace, click the split icon at the top of the terminal panel (or open a second one with **+**). Inside a single terminal, `tmux` does the same job: type `tmux`, then press Ctrl+b followed by `%` to split, and Ctrl+b followed by an arrow key to move between the halves.

**A few optional tools.** Some experiments use tools that aren't always installed. If a command says `command not found`, install them all at once:

```
$ sudo apt update
$ sudo apt install -y tmux netcat-openbsd lsof strace
```

**Creating the Python files.** When the lab says "create a file named `something.py`", open it in the editor (`code something.py` in a Codespace, or `nano something.py`), paste the code, and save.

**Every part stands alone.** Each part starts by making its own folder under `~/lab05` and builds everything it needs inside it. You can do them in any order, skip one, or come back to one next week. The suggested order is the numbered one, because later parts are easier when the earlier ideas are fresh.

**The optional boxes.** Anything marked **Try another way** or **Experiment** is optional. The main path works without them. Most of the surprises live in the experiments, though, so do the ones you have time for.

**Predict first.** Many steps ask you to guess what will happen before you press Enter. A wrong guess you notice teaches more than a right answer you copied, so actually make the guess.

| Part | Topic | Needs two terminals |
|---|---|---|
| 1 | The family of file types | no |
| 2 | Names, inodes and directories | no |
| 3 | Hard links: one file, many names | no |
| 4 | Symbolic links: a note with a path on it | no |
| 5 | File descriptors: the numbers a program holds | yes (one step) |
| 6 | `/proc` and `/sys`: the kernel as files | no |
| 7 | Pipes and named pipes | yes |
| 8 | The terminal is a file too | yes (one step) |
| 9 | Sockets | yes |

---

## Part 1: The family of file types

Linux recognises seven kinds of file. You're about to make five of them yourself and find the other two already waiting in `/dev`.

### Step 1.1 Set up

```
$ mkdir -p ~/lab05/part1
$ cd ~/lab05/part1
```

### Step 1.2 Make one of each

```
$ echo "hello" > notes.txt
$ mkdir box
$ ln -s notes.txt shortcut
$ mkfifo mypipe
$ python3 -c "import socket; socket.socket(socket.AF_UNIX).bind('mysock')"
```

One at a time, that's a regular file, a directory, a symbolic link, a named pipe, and a socket. The shell has no built-in command for making a socket file, so a one-line Python program does it: it creates a socket, gives it the address `mysock`, and exits. The address happens to be a file name, so a file appears.

### Step 1.3 Read the first letter

```
$ ls -l
total 8
drwxrwxr-x 2 codespace codespace 4096 Sep 30 16:29 box
prw-rw-r-- 1 codespace codespace    0 Sep 30 16:29 mypipe
srwxrwxr-x 1 codespace codespace    0 Sep 30 16:29 mysock
-rw-rw-r-- 1 codespace codespace    6 Sep 30 16:29 notes.txt
lrwxrwxrwx 1 codespace codespace    9 Sep 30 16:29 shortcut -> notes.txt
```

Everyone looks at the permissions in this column. The very first character, before them, is the type:

| Letter | Type | Made with |
|---|---|---|
| `-` | regular file | `echo > file`, `touch`, any editor |
| `d` | directory | `mkdir` |
| `l` | symbolic link | `ln -s` |
| `p` | named pipe (FIFO) | `mkfifo` |
| `s` | socket | a program calling `bind()` |
| `c` | character device | the kernel, in `/dev` |
| `b` | block device | the kernel, in `/dev` |

Look at the sizes. `mypipe` and `mysock` are 0 bytes and will stay that way. They're meeting points, not storage: data passes through them without ever landing on the disk. `shortcut` is 9 bytes, which is exactly the length of the text `notes.txt`. Hold on to that; Part 4 explains it.

### Step 1.4 Ask for a second opinion

```
$ file notes.txt box shortcut mypipe mysock
notes.txt: ASCII text
box:       directory
shortcut:  symbolic link to notes.txt
mypipe:    fifo (named pipe)
mysock:    socket
```

`file` goes a step further than `ls` for regular files: it peeks at the contents and guesses what kind of data is inside ("ASCII text", "PNG image", "Python script").

> **Try another way.** `stat` prints everything the system knows about a file, and `-c` picks out just the parts you ask for. `%F` is the type in words:
>
> ```
> $ stat -c "%-10n %F" notes.txt box shortcut mypipe mysock
> notes.txt  regular file
> box        directory
> shortcut   symbolic link
> mypipe     fifo
> mysock     socket
> ```

### Step 1.5 The two you didn't make

```
$ ls -l /dev/null /dev/zero /dev/urandom /dev/tty
crw-rw-rw- 1 root root 1, 3 Sep 29 23:17 /dev/null
crw-rw-rw- 1 root root 5, 0 Sep 29 23:17 /dev/tty
crw-rw-rw- 1 root root 1, 9 Sep 29 23:17 /dev/urandom
crw-rw-rw- 1 root root 1, 5 Sep 29 23:17 /dev/zero
```

`c` for character device. Where a size would normally be, there are two numbers, like `1, 3`. The first says which driver in the kernel handles this device, the second says which of that driver's devices it is. Nothing is stored in these files. Reading or writing one is a request to a driver, and the driver decides what happens.

Try each one:

```
$ echo "this goes nowhere" > /dev/null
$ cat /dev/null
$ head -c 16 /dev/zero | od -A d -t x1
0000000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0000016
$ head -c 8 /dev/urandom | od -A n -t x1
 fe fe 60 d2 cf 5a f0 f9
```

`/dev/null` swallows whatever you write and gives back nothing when read. `/dev/zero` hands out zero bytes forever. `/dev/urandom` hands out random bytes forever, which is why you asked `head` for only a few. (`od` shows bytes as numbers, since these bytes aren't printable text.) Your random bytes will be different, of course.

Now the block devices:

```
$ find /dev -maxdepth 1 -type b
```

On a VM or a real machine you'll see names like `/dev/sda`, `/dev/vda` or `/dev/nvme0n1`: the disks, and numbered partitions like `sda1`. In a Codespace the list may well be empty. A Codespace runs inside a container, and containers usually don't get to see the real disks at all.

**About "character" and "block".** You'll often read that character devices send "one character at a time" and block devices work "in blocks". That's a loose description. You can read a thousand bytes from `/dev/urandom` in one go. The real difference is how the kernel treats them. A block device is storage: the kernel keeps a cache of it and lets you jump to any position, which is what a filesystem needs. A character device is a stream: bytes arrive in order, and "go back to position 500" usually means nothing. A keyboard can't rewind.

> **Experiment: fill a file with nothing.** `/dev/zero` is handy for making a file of an exact size:
>
> ```
> $ head -c 1000000 /dev/zero > million.bin
> $ ls -l million.bin
> -rw-rw-r-- 1 codespace codespace 1000000 Sep 30 16:29 million.bin
> $ rm million.bin
> ```
>
> People do this to test how a program copes with big files, or to reserve space ahead of time.

### Step 1.6 Search by type

`find` can filter by the same letters (with `f` for a regular file):

```
$ find . -type p
./mypipe
$ find . -type s
./mysock
$ find . -type l
./shortcut
$ find /dev -maxdepth 1 -type c | wc -l
88
```

The last line counts the character devices in `/dev`. Your number will differ.

### Step 1.7 The Python version

The same zoo, built and labelled from Python. Create `file_zoo.py`:

```python
# file_zoo.py: make one of each kind of file, use a couple of them, then label them all.
import os
import shutil
import socket
import stat
from threading import Thread

ZOO = "zoo"
shutil.rmtree(ZOO, ignore_errors=True)        # start fresh every run
os.mkdir(ZOO)                                 # a directory
os.chdir(ZOO)                                 # work inside it from here on

# 1. A regular file
with open("notes.txt", "w") as f:
    f.write("Hello, this is a regular file.\n")

# 2. A directory with a file in it
os.mkdir("box")
os.rename("notes.txt", "box/notes.txt")

# 3. A symbolic link. The target is read from the folder the LINK sits in,
#    so a link inside box/ that points at notes.txt means box/notes.txt.
os.symlink("notes.txt", "box/shortcut")

# 4. A named pipe (FIFO): one thread writes, the main program reads
os.mkfifo("pipe")

def fifo_writer():
    with open("pipe", "w") as p:
        p.write("Hello through the named pipe!")

Thread(target=fifo_writer).start()
with open("pipe") as p:
    print("FIFO   :", p.read())

# 5. A Unix domain socket: its address is a file name, so it shows up on disk
server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
server.bind("socket")
server.listen()

def socket_server():
    conn, _ = server.accept()
    with conn:
        conn.sendall(b"Hello through the socket!")

Thread(target=socket_server).start()
with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as client:
    client.connect("socket")
    print("SOCKET :", client.recv(1024).decode())
server.close()          # the socket FILE stays behind until someone removes it

# Now label everything, the way `ls -l` decides its first letter
KINDS = [(stat.S_ISREG, "-", "regular file"), (stat.S_ISDIR, "d", "directory"),
         (stat.S_ISLNK, "l", "symbolic link"), (stat.S_ISFIFO, "p", "named pipe"),
         (stat.S_ISSOCK, "s", "socket"), (stat.S_ISCHR, "c", "character device"),
         (stat.S_ISBLK, "b", "block device")]

print()
for path in ["box", "box/notes.txt", "box/shortcut", "pipe", "socket", "/dev/null"]:
    mode = os.lstat(path).st_mode      # lstat: look at the link itself, don't follow it
    for test, letter, name in KINDS:
        if test(mode):
            print(f"  {letter}  {path:<15} {name}")
```

Run it:

```
$ python3 file_zoo.py
FIFO   : Hello through the named pipe!
SOCKET : Hello through the socket!

  d  box             directory
  -  box/notes.txt   regular file
  l  box/shortcut    symbolic link
  p  pipe            named pipe
  s  socket          socket
  c  /dev/null       character device
$ cat zoo/box/shortcut
Hello, this is a regular file.
```

A few details worth a second look:

- **The symlink target is `"notes.txt"`, not `"box/notes.txt"`.** A relative target is looked up starting from the folder the link lives in. A link inside `box/` pointing at `box/notes.txt` would send the system looking for `box/box/notes.txt`, which doesn't exist. Part 4 lets you make that mistake on purpose.
- **The socket is `AF_UNIX`.** A network socket (`AF_INET`, the kind Part 9 uses) never appears as a file. A Unix domain socket uses a file name as its address, so this one shows up in `ls`.
- **`os.lstat`, not `os.stat`.** `stat` follows a symlink and reports on whatever it points to, so the shortcut would have been labelled "regular file". The `l` in `lstat` means "look at the link itself".
- **It runs again cleanly.** The script deletes the old `zoo` folder at the start. Without that, the second run would crash on `os.mkdir` because the folder already exists.

> **Experiment: poke the leftover socket.** The program has ended, so nobody is listening on `zoo/socket` any more. Try reading it:
>
> ```
> $ cat zoo/socket
> cat: zoo/socket: No such device or address
> ```
>
> The file is there, but a socket file can't be opened like an ordinary file. Programs have to `connect()` to it instead, and with nobody listening, a connect gets `Connection refused`. A leftover socket file like this is harmless but confusing. Part 9 shows why servers delete theirs.

---

## Part 2: Names, inodes and directories

A file seems like one thing: a name with some contents. Linux actually keeps it in two separate pieces, and most of the surprises in Parts 3, 4 and 5 come from that split.

- The **inode** holds everything about the file except its name: type, owner, permissions, size, timestamps, how many names point at it, and where its bytes sit on the disk. Every inode has a number.
- The **name** lives somewhere else entirely: in the directory. A directory is a small table that says "the name `essay.txt` means inode 794685".

A library works much the same way. The book on the shelf has a call number. The catalogue cards say "Moby Dick, see 813.3" and "Whale stories, see 813.3". The book doesn't carry its own title card around. It just sits at 813.3, and any number of cards can send you there.

### Step 2.1 Set up and look at an inode

```
$ mkdir -p ~/lab05/part2
$ cd ~/lab05/part2
$ echo "first draft" > essay.txt
$ ls -i essay.txt
794685 essay.txt
$ stat essay.txt
  File: essay.txt
  Size: 12        	Blocks: 8          IO Block: 4096   regular file
Device: 254,0	Inode: 794685      Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1001/codespace)   Gid: ( 1002/codespace)
Access: 2026-09-30 16:44:08.901670707 +0000
Modify: 2026-09-30 16:44:08.904170952 +0000
Change: 2026-09-30 16:44:08.904170952 +0000
 Birth: 2026-09-30 16:44:08.901670707 +0000
```

Your inode number will be different. Everything `stat` printed, apart from the `File:` line at the top, comes out of the inode. `Links: 1` means exactly one directory entry points at this inode right now.

### Step 2.2 Rename it, then copy it

Predict first: after a rename, will the inode number change? After a copy?

```
$ mv essay.txt final.txt
$ ls -i final.txt
794685 final.txt
$ cp final.txt copy.txt
$ ls -i final.txt copy.txt
794686 copy.txt
794685 final.txt
```

`mv` within the same filesystem touched only the directory table: it crossed out one name and wrote another next to the same number. No data moved, which is why renaming a 10 GB file is instant. `cp` made a genuinely new file, with a new inode and its own copy of the bytes.

### Step 2.3 Change the contents, keep the identity

```
$ echo "second draft" > final.txt
$ ls -i final.txt
794685 final.txt
```

Same number. `>` empties the existing file and writes into it. The inode (the "which file") stayed put while the contents (the "what's in it") changed. Part 3 shows a tool that does *not* behave this way, and why that matters.

### Step 2.4 A directory is a table of names

```
$ mkdir -p shelf/a shelf/b shelf/c
$ ls -ia shelf
794687 .
794684 ..
794688 a
794689 b
794690 c
```

`-a` shows the hidden entries. That listing *is* the directory's content: names and inode numbers, nothing else. Two of the entries are always there. `.` is the directory's own inode, and `..` is its parent's.

Now look at how many names `shelf` has:

```
$ ls -ld shelf
drwxrwxr-x 5 codespace codespace 4096 Sep 30 16:44 shelf
```

The `5` is the link count. Where do five names come from? Count them:

```
$ ls -id shelf shelf/. shelf/a/.. shelf/b/.. shelf/c/..
794687 shelf
794687 shelf/.
794687 shelf/a/..
794687 shelf/b/..
794687 shelf/c/..
```

One in `part2`, one as its own `.`, and one `..` inside each of its three subfolders. So an empty directory always starts at 2, and each subfolder adds one. That's also a quick way to count subfolders without listing them.

(Inode numbers are recycled: once a file is gone for good, its number can be handed to a new file. So a number identifies a file only while that file exists.)

### Step 2.5 Three timestamps, and none of them is "created"

```
$ stat -c "modify: %y" final.txt
modify: 2026-09-30 16:44:08.914450895 +0000
$ chmod 600 final.txt
$ stat -c "modify: %y" final.txt
modify: 2026-09-30 16:44:08.914450895 +0000
$ stat -c "change: %z" final.txt
change: 2026-09-30 16:44:09.927812363 +0000
```

The classic inode has three times:

- **Access** (`atime`): last time the contents were read. Many systems update this lazily to save disk writes, so don't trust it much.
- **Modify** (`mtime`): last time the contents changed. This is the one `ls -l` shows.
- **Change** (`ctime`): last time the *inode* changed: contents, permissions, owner, link count, anything.

`chmod` left Modify alone and bumped Change. A lot of older notes, including some you may have seen, describe `ctime` as "creation time". It isn't. Newer filesystems such as ext4 do record a creation time as well; `stat` shows it as `Birth`. Some filesystems don't, and then `Birth` shows a `-`.

### Step 2.6 Inodes can run out

```
$ df -i .
Filesystem       Inodes  IUsed    IFree IUse% Mounted on
/dev/vda       16777216 274216 16503000    2% /
```

Many filesystems create a fixed number of inodes when the disk is formatted. Every file uses one, however small it is. A program that writes millions of tiny files can fill the inode table while plenty of bytes remain, and then every attempt to create a file fails with "No space left on device", even though `df -h` shows free space. It's rare, but when it happens `df -i` is how you find out.

### Step 2.7 A model you can run

Here is the whole idea in about sixty lines of Python. Directories are dictionaries from names to inode numbers; a table maps numbers to inodes; and nothing about a file's name is stored in its inode. Create `fs_model.py`:

```python
# fs_model.py: a tiny model of how Linux finds a file by its name.
# Directories hold names. Inodes hold everything else. Names point at inode numbers.

inode_table = {}        # inode number -> Inode
next_free = 2           # inode numbers handed out in order


class Inode:
    def __init__(self, kind, contents):
        global next_free
        self.number = next_free
        next_free += 1
        self.kind = kind            # "file" or "directory"
        self.contents = contents    # file: some text; directory: {name: inode number}
        self.links = 0              # how many names point here
        inode_table[self.number] = self


def add_name(directory, name, inode):
    directory.contents[name] = inode.number
    inode.links += 1


def remove_name(directory, name):
    number = directory.contents.pop(name)
    inode = inode_table[number]
    inode.links -= 1
    if inode.links == 0:
        del inode_table[number]
        print(f"  (inode {number} has no names left, its space is freed)")


def lookup(path):
    """Walk a path like /home/notes.txt one name at a time, starting at the root."""
    current = root
    for name in path.strip("/").split("/"):
        if current.kind != "directory" or name not in current.contents:
            return None
        current = inode_table[current.contents[name]]
    return current


def show(label, path):
    inode = lookup(path)
    if inode is None:
        print(f"{label:<28} {path:<22} -> not found")
    else:
        print(f"{label:<28} {path:<22} -> inode {inode.number}, links={inode.links}, {inode.contents!r}")


# Build a small tree: / and /home
root = Inode("directory", {})
root.links = 1                      # the root has no parent directory to name it; count it once
home = Inode("directory", {})
add_name(root, "home", home)

# A file with one name
notes = Inode("file", "buy milk")
add_name(home, "notes.txt", notes)
show("one name", "/home/notes.txt")

# A second name for the same inode: this is what `ln` does
add_name(home, "also.txt", notes)
show("second name (hard link)", "/home/also.txt")

# Renaming inside one filesystem only moves the name: this is what `mv` does
add_name(home, "renamed.txt", notes)
remove_name(home, "notes.txt")
show("after rename, old name", "/home/notes.txt")
show("after rename, new name", "/home/renamed.txt")

# Deleting a name: this is what `rm` does
remove_name(home, "renamed.txt")
show("after rm renamed.txt", "/home/also.txt")
remove_name(home, "also.txt")
show("after rm also.txt", "/home/also.txt")

print("inodes still in use:", sorted(inode_table))
```

```
$ python3 fs_model.py
one name                     /home/notes.txt        -> inode 4, links=1, 'buy milk'
second name (hard link)      /home/also.txt         -> inode 4, links=2, 'buy milk'
after rename, old name       /home/notes.txt        -> not found
after rename, new name       /home/renamed.txt      -> inode 4, links=2, 'buy milk'
after rm renamed.txt         /home/also.txt         -> inode 4, links=1, 'buy milk'
  (inode 4 has no names left, its space is freed)
after rm also.txt            /home/also.txt         -> not found
inodes still in use: [2, 3]
```

Follow inode 4 down the output. It never moves and never changes. Names come and go around it, the link count goes up and down, and only when the count reaches zero does the file itself disappear. Notice there's no "delete file" function in the model at all. There's only "remove a name". That matches the real system: the system call behind `rm` is called `unlink`.

(One more reason the real system frees an inode later than this model does is in Part 5.)

**A word about "dentries".** You'll meet this word in Linux material, sometimes used as if it meant the name-to-inode entries above. Strictly, the entries stored on disk inside a directory are *directory entries*. A *dentry* is the kernel's in-memory copy of one of them, kept in the *dentry cache* so that looking up `/home/codespace/lab05/part2/final.txt` a second time doesn't mean reading five directories from disk again. So the dentry cache is a speed-up for the lookup your model does in `lookup()`, not a separate part of how files are stored.

> **Experiment: add a dentry cache to the model, and break it.** Paste this at the bottom of `fs_model.py` and run again:
>
> ```python
> print("\n--- with a lookup cache ---")
> cache = {}                               # path -> inode number
>
> def cached_lookup(path):
>     if path in cache:
>         print(f"  cache hit for {path}")
>         return inode_table.get(cache[path])
>     inode = lookup(path)
>     if inode is not None:
>         cache[path] = inode.number
>     return inode
>
> story = Inode("file", "once upon a time")
> add_name(home, "a.txt", story)
> add_name(home, "b.txt", story)
> cached_lookup("/home/a.txt")             # slow path, fills the cache
> cached_lookup("/home/a.txt")             # fast path
> remove_name(home, "a.txt")               # rm a.txt ... but nobody told the cache
> found = cached_lookup("/home/a.txt")
> print("a.txt after rm:", "still found!" if found else "not found")
> ```
>
> The cache happily finds a file under a name that was deleted, because the inode is still alive under `b.txt` and nothing told the cache the name was gone. The fix is one line in `remove_name`: forget any cached path for that name. The real kernel does exactly that on every `unlink` and `rename`. Keeping a cache fast is the easy half; knowing when to throw entries away is the hard half, all over computing.

> **Experiment: a move that isn't a rename.** `mv` is only a name change when source and destination are on the same filesystem. `/dev/shm` is a separate, memory-backed filesystem on most Linux systems (check with `df /dev/shm`). Ask Python to rename across the boundary:
>
> ```
> $ cp final.txt moving.txt
> $ python3 -c "import os; os.rename('moving.txt', '/dev/shm/moving.txt')"
> OSError: [Errno 18] Invalid cross-device link: 'moving.txt' -> '/dev/shm/moving.txt'
> $ python3 -c "import shutil; shutil.move('moving.txt', '/dev/shm/moving.txt'); print('moved')"
> moved
> $ ls -i /dev/shm/moving.txt
> 10 /dev/shm/moving.txt
> $ rm /dev/shm/moving.txt
> ```
>
> A directory on one filesystem can only point at inodes on that same filesystem, so a real rename is impossible. `os.rename` refuses. `shutil.move` (and the `mv` command) quietly copy the bytes into a new inode over there and delete the original. Same command, very different amount of work.

---

## Part 3: Hard links: one file, many names

Part 2 showed that a name is only an entry in a directory table. Nothing stops two entries from holding the same inode number. That's a hard link: a second name for the very same file, with neither name more "original" than the other.

The scenario: two projects need the same database settings, and you want to change them in one place.

### Step 3.1 Set up

```
$ mkdir -p ~/lab05/part3
$ cd ~/lab05/part3
$ echo '{"host": "initial_db_host", "user": "initial_db_user"}' > db_config.json
$ mkdir project1 project2
$ ln db_config.json project1/db_config.json
$ ln db_config.json project2/db_config.json
```

`ln` without `-s` makes a hard link. The order is always *existing file first, new name second*, the same order as `cp`.

### Step 3.2 Three names, one inode

```
$ ls -li db_config.json project1/db_config.json project2/db_config.json
794694 -rw-rw-r-- 3 codespace codespace 55 Sep 30 16:44 db_config.json
794694 -rw-rw-r-- 3 codespace codespace 55 Sep 30 16:44 project1/db_config.json
794694 -rw-rw-r-- 3 codespace codespace 55 Sep 30 16:44 project2/db_config.json
```

Same inode number on all three lines, and a link count of 3. It's one file listed three times. No copies were made.

### Step 3.3 A script that reads its own copy

Create `project1/script.py`:

```python
import json
from pathlib import Path

here = Path(__file__).parent              # the folder this script lives in
config_path = here / "db_config.json"     # so it always reads the file next to it

with open(config_path) as f:
    credentials = json.load(f)
print(f"{here.name} connects to: {credentials}")
```

Copy it into the other project and run both from where you are:

```
$ cp project1/script.py project2/script.py
$ python3 project1/script.py
project1 connects to: {'host': 'initial_db_host', 'user': 'initial_db_user'}
$ python3 project2/script.py
project2 connects to: {'host': 'initial_db_host', 'user': 'initial_db_user'}
```

**Why `Path(__file__).parent`?** A plain `open("db_config.json")` looks in the *current* folder, which is wherever you happened to be when you typed the command, not where the script lives. You're in `part3`, and `part3` has its own `db_config.json`, so a plain `open` would quietly read that one and the demo would look like it worked for the wrong reason. `__file__` is the path of the running script, so building the path from it always finds the file sitting next to the script.

### Step 3.4 Change it in one place

```
$ echo '{"host": "new_db_host", "user": "new_db_user"}' > db_config.json
$ python3 project1/script.py
project1 connects to: {'host': 'new_db_host', 'user': 'new_db_user'}
$ python3 project2/script.py
project2 connects to: {'host': 'new_db_host', 'user': 'new_db_user'}
```

Both projects see the change, because there was only ever one file to change. You'd get the same result writing through `project1/db_config.json` instead. Every name is an equal door into the same room.

### Step 3.5 Delete the "original"

Predict first: after `rm db_config.json`, what happens to the projects?

```
$ rm db_config.json
$ ls -li project1/db_config.json project2/db_config.json
794694 -rw-rw-r-- 2 codespace codespace 47 Sep 30 16:44 project1/db_config.json
794694 -rw-rw-r-- 2 codespace codespace 47 Sep 30 16:44 project2/db_config.json
$ python3 project1/script.py
project1 connects to: {'host': 'new_db_host', 'user': 'new_db_user'}
```

Nothing broke. One name is gone and the link count dropped to 2. Put the central name back so the rest of the part works:

```
$ ln project1/db_config.json db_config.json
```

### Step 3.6 Find every name of a file

A hard link doesn't record where the other names are, so the only way to find them is to search:

```
$ find . -samefile db_config.json
./project1/db_config.json
./db_config.json
./project2/db_config.json
$ find . -type f -links +1
```

The second command lists every regular file with more than one name, which is a quick check for hard links you might have forgotten about.

### Step 3.7 How to break a hard link without meaning to

Some tools don't edit a file. They write a brand-new file and rename it over the old name. `sed -i` (edit in place) is one of them:

```
$ sed -i 's/new_db_host/sed_host/' db_config.json
$ ls -li db_config.json project1/db_config.json project2/db_config.json
794699 -rw-rw-r-- 1 codespace codespace 44 Sep 30 16:44 db_config.json
794694 -rw-rw-r-- 2 codespace codespace 47 Sep 30 16:44 project1/db_config.json
794694 -rw-rw-r-- 2 codespace codespace 47 Sep 30 16:44 project2/db_config.json
$ python3 project1/script.py
project1 connects to: {'host': 'new_db_host', 'user': 'new_db_user'}
```

`db_config.json` now has a new inode and a link count of 1. The projects still share the old one and never saw the edit. No error, no warning: the setup is simply split in two. Text editors vary here too. Some save by writing into the existing file, some by writing a new file and renaming it, and some change behaviour depending on settings. Don't trust memory on this; after saving, run `ls -i` and see.

Rejoin them before moving on (`-f` replaces the existing name):

```
$ ln -f project1/db_config.json db_config.json
```

### Step 3.8 What can't be hard linked

```
$ ln project1 p1link
ln: project1: hard link not allowed for directory
$ ln db_config.json /dev/shm/config_copy
ln: failed to create hard link '/dev/shm/config_copy' => 'db_config.json': Invalid cross-device link
```

Directories are refused so the tree can never loop back into itself. (Imagine a folder that contains a hard link to its own grandparent. Every program that walks the tree, like `find` or a backup tool, would go round forever.) The second failure is the Part 2 rule again: a directory entry can only point at an inode on its own filesystem, and `/dev/shm` is a different one. Symbolic links, in Part 4, have neither limit.

### Step 3.9 Permissions belong to the inode

Predict first: if you make one name read-only, what happens to the others?

```
$ chmod 444 project1/db_config.json
$ ls -l db_config.json project1/db_config.json project2/db_config.json
-r--r--r-- 3 codespace codespace 47 Sep 30 16:44 db_config.json
-r--r--r-- 3 codespace codespace 47 Sep 30 16:44 project1/db_config.json
-r--r--r-- 3 codespace codespace 47 Sep 30 16:44 project2/db_config.json
$ chmod u+w db_config.json
```

All three changed, and the last command (through yet another name) gives the owner write permission back on all three. Permissions are stored in the inode, and there's only one. The same goes for the owner and the timestamps. The *only* thing that differs between hard links is the name and the folder it sits in.

### Step 3.10 A careful updater

The idea behind the next script: keep the shared file read-only so nobody edits it by accident, and let one function lift the protection, update, and put it back. Create `safe_update.py`:

```python
# safe_update.py: one config file, several names, careful updates.
import json
import os
import stat

CENTRAL = "central_config.json"
PROJECTS = ["project1", "project2"]


def initialize_config():
    """Create the central file, but only the first time."""
    if os.path.exists(CENTRAL):
        print(f"{CENTRAL} already exists, leaving it alone")
        return
    config = {"database": {"host": "default_host", "user": "default_user"}}
    with open(CENTRAL, "w") as f:
        json.dump(config, f, indent=4)


def create_hard_links():
    """Give each project its own name for the same inode."""
    for folder in PROJECTS:
        os.makedirs(folder, exist_ok=True)
        link = os.path.join(folder, "config.json")
        if not os.path.exists(link):
            os.link(CENTRAL, link)


def set_value(config, dotted_key, value):
    """set_value(cfg, "database.host", "x") sets cfg["database"]["host"] = "x"."""
    *parents, last = dotted_key.split(".")
    for key in parents:
        config = config[key]
    config[last] = value


def safe_update(dotted_key, value):
    with open(CENTRAL) as f:
        config = json.load(f)
    set_value(config, dotted_key, value)

    # Turn the data into text FIRST. If this fails, the file has not been touched yet.
    try:
        text = json.dumps(config, indent=4)
    except TypeError as e:
        print(f"Update refused, file unchanged: {e}")
        return

    os.chmod(CENTRAL, 0o644)                 # lift read-only
    try:
        with open(CENTRAL, "w") as f:        # "w" rewrites the SAME inode, so every link sees it
            f.write(text)
        print(f"Updated {dotted_key} = {value!r}")
    finally:
        os.chmod(CENTRAL, 0o444)             # read-only again, even if the write failed


initialize_config()
create_hard_links()
os.chmod(CENTRAL, 0o444)

safe_update("database.host", "new_secure_host")
safe_update("database.port", {5432})         # a set is not valid JSON: watch it get refused

for name in [CENTRAL] + [os.path.join(p, "config.json") for p in PROJECTS]:
    info = os.stat(name)
    print(f"{name:<26} inode={info.st_ino} links={info.st_nlink} mode={stat.filemode(info.st_mode)}")
```

Run it twice:

```
$ python3 safe_update.py
Updated database.host = 'new_secure_host'
Update refused, file unchanged: Object of type set is not JSON serializable
central_config.json        inode=794700 links=3 mode=-r--r--r--
project1/config.json       inode=794700 links=3 mode=-r--r--r--
project2/config.json       inode=794700 links=3 mode=-r--r--r--
$ python3 safe_update.py
central_config.json already exists, leaving it alone
Updated database.host = 'new_secure_host'
Update refused, file unchanged: Object of type set is not JSON serializable
...
$ cat project2/config.json
{
    "database": {
        "host": "new_secure_host",
        "user": "default_user"
    }
}
```

If you've seen an older version of this script, three things changed, and each fixes a real failure:

1. **It can run twice.** The old version crashed on the second run. It tried to rewrite a file it had just made read-only (`PermissionError`), and then to create links that already existed (`FileExistsError`).
2. **A bad value can't wreck the file.** The old version opened the file with `"w"`, which empties it immediately, and only *then* converted the data to JSON. When the conversion failed halfway, the file was left cut off in the middle, something like `"port": ` and nothing after it, and every project read a broken config. Converting to text first means a failure happens before anything is touched.
3. **`os.chmod` instead of `os.system("chmod ...")`.** Same effect, but no separate shell program to start, and an error raises a proper Python exception instead of being printed and ignored. The `finally` puts read-only back even if the write itself fails.

**What `chmod 444` does and doesn't protect.** It stops accidents: a stray `echo ... >` or an editor save gets "Permission denied". It doesn't stop anyone determined. The owner can always `chmod` it back, as the script itself does, and root ignores these permission bits entirely:

```
$ echo "oops" > central_config.json
bash: central_config.json: Permission denied
$ sudo sh -c 'echo "root was here" >> central_config.json'
$ tail -n 1 project1/config.json
root was here
```

Think of it as a seatbelt, not a lock. (Run `python3 safe_update.py` again afterwards and watch what happens to that extra line. Why does `json.load` complain?)

> **Experiment: the atomic-write dilemma.** Careful programs usually save important files in two steps: write the new version to a temporary file, then rename it over the old name. A rename is all-or-nothing, so any reader sees either the whole old file or the whole new one, never half of each. That's the safest way to update a config, and it's precisely what broke the link in Step 3.7: the rename installs a new inode under one name and leaves every other hard link on the old one. `safe_update.py` writes in place to keep the links together, which means a reader that opens the file mid-write could see half of it.
>
> So with hard links you get to pick one: shared updates, or all-or-nothing updates. A symbolic link (Part 4) gets both, because the link can be switched to a freshly written file in one rename. This is one reason real deployment tools prefer symlinks for this job. Try it: write the steps for a symlink-based version of this setup in plain words before reading Part 4.

---

## Part 4: Symbolic links: a note with a path on it

A hard link is a second name for an inode. A symbolic link is something else: a tiny file of its own whose entire content is a path, like a sticky note that says "the thing you want is over at `config_dev.json`". Whenever a program opens the link, the system reads the note and goes there instead.

The scenario: one program that always reads `config.json`, and a quick way to switch it between development and production settings without touching the code.

### Step 4.1 Set up

```
$ mkdir -p ~/lab05/part4
$ cd ~/lab05/part4
```

Create `config_dev.json`:

```json
{
  "environment": "development",
  "database": {"host": "localhost", "port": 5432}
}
```

Create `config_prod.json`:

```json
{
  "environment": "production",
  "database": {"host": "prod.mydatabase.com", "port": 5432}
}
```

And `read_config.py`:

```python
# read_config.py: always opens config.json, and reports which real file that turned out to be.
import json
import os
from pathlib import Path

config_path = Path(__file__).parent / "config.json"

with open(config_path) as f:          # open() follows the symlink for us
    config = json.load(f)

real = os.path.realpath(config_path)  # where the link chain actually ends
print(f"config.json leads to: {os.path.basename(real)}")
print(f"Environment:   {config['environment']}")
print(f"Database host: {config['database']['host']}")
print(f"Database port: {config['database']['port']}")
```

Before linking anything, see exactly how the two configs differ:

```
$ diff config_dev.json config_prod.json
2,3c2,3
<   "environment": "development",
<   "database": {"host": "localhost", "port": 5432}
---
>   "environment": "production",
>   "database": {"host": "prod.mydatabase.com", "port": 5432}
```

`2,3c2,3` reads "lines 2 to 3 of the first file were changed into lines 2 to 3 of the second". Lines starting with `<` come from the first file, `>` from the second. Line 1 and line 4 match, so they aren't shown.

> **Try another way.** `diff -y config_dev.json config_prod.json` puts the files side by side and marks changed lines with `|`. `diff -q` only says *whether* they differ, which is what a script usually wants.

### Step 4.2 Link, run, switch, run

```
$ ln -s config_dev.json config.json
$ python3 read_config.py
config.json leads to: config_dev.json
Environment:   development
Database host: localhost
Database port: 5432
$ ln -sf config_prod.json config.json
$ python3 read_config.py
config.json leads to: config_prod.json
Environment:   production
Database host: prod.mydatabase.com
Database port: 5432
```

Same argument order as `ln` and `cp`: *target first, link name second*. `-f` (force) replaces a link that already exists.

### Step 4.3 What a symlink is made of

```
$ ls -li
total 12
794706 lrwxrwxrwx 1 codespace codespace  16 Sep 30 16:44 config.json -> config_prod.json
794702 -rw-r--r-- 1 codespace codespace  86 Sep 30 16:44 config_dev.json
794703 -rw-r--r-- 1 codespace codespace  95 Sep 30 16:44 config_prod.json
794704 -rw-r--r-- 1 codespace codespace 586 Sep 30 16:44 read_config.py
$ readlink config.json
config_prod.json
```

Three things set it apart from a hard link:

- **It has its own inode**, different from its target's.
- **Its size is 16**, the number of characters in `config_prod.json`. That text is the whole content.
- **Its permissions read `rwxrwxrwx` and mean nothing.** Linux ignores a symlink's own permissions; what counts is the permissions of the file at the end.

### Step 4.4 The relative-path trap

This one catches nearly everyone once. Predict: will `bad_link` work?

```
$ mkdir settings
$ echo "hi" > settings/file1.txt
$ ln -s settings/file1.txt settings/bad_link
$ ln -s file1.txt settings/good_link
$ ls -l settings
total 4
lrwxrwxrwx 1 codespace codespace 18 Sep 30 16:44 bad_link -> settings/file1.txt
-rw-rw-r-- 1 codespace codespace  3 Sep 30 16:44 file1.txt
lrwxrwxrwx 1 codespace codespace  9 Sep 30 16:44 good_link -> file1.txt
$ cat settings/bad_link
cat: settings/bad_link: No such file or directory
$ cat settings/good_link
hi
```

`ln -s` stores the target text exactly as you typed it, without checking anything. When the link is used later, a relative target is looked up *starting from the folder the link sits in*. `bad_link` lives in `settings/`, so its note sends the system to `settings/settings/file1.txt`. A rule of thumb: write the target as it would look to someone standing next to the link, or use a full path starting with `/`.

### Step 4.5 Dangling links

A symlink can point at something that doesn't exist yet, or no longer exists:

```
$ ln -s config_staging.json config_next.json
$ ls -l config_next.json
lrwxrwxrwx 1 codespace codespace 19 Sep 30 16:44 config_next.json -> config_staging.json
$ cat config_next.json
cat: config_next.json: No such file or directory
$ test -e config_next.json || echo "-e: nothing there"
-e: nothing there
$ test -L config_next.json && echo "-L: but the link itself exists"
-L: but the link itself exists
```

The error message says `config_next.json` doesn't exist, while `ls` shows it right there. What's missing is the thing at the end of the note. To find all broken links under a folder:

```
$ find . -xtype l
./settings/bad_link
./config_next.json
```

Now make one go dangling on purpose, and see what Python reports:

```
$ mv config_prod.json old_prod.json
$ python3 read_config.py
...
FileNotFoundError: [Errno 2] No such file or directory: '/home/codespace/lab05/part4/config.json'
$ mv old_prod.json config_prod.json
```

Python names `config.json` as missing, too. When you see "No such file" for a file you can plainly see, check whether it's a link with `ls -l`.

**Your task:** create `config_staging.json` with `"environment": "staging"` and a host of your choice. The moment you save it, `config_next.json` stops being broken without you touching the link. Then point `config.json` at staging and run `read_config.py`.

### Step 4.6 Links to folders, and the `-n` trap

Symlinks can point at directories, which hard links can't. A common pattern is a `current` link pointing at the release in use:

```
$ mkdir -p releases/v1 releases/v2
$ echo "version 1" > releases/v1/app.txt
$ echo "version 2" > releases/v2/app.txt
$ ln -s releases/v1 current
$ cat current/app.txt
version 1
```

Now switch to v2 the way you switched configs. Predict first.

```
$ ln -sf releases/v2 current
$ cat current/app.txt
version 1
$ ls -l releases/v1
total 4
-rw-rw-r-- 1 codespace codespace 10 Sep 30 16:44 app.txt
lrwxrwxrwx 1 codespace codespace 11 Sep 30 16:44 v2 -> releases/v2
```

It didn't switch. Because `current` points at a directory, `ln` treated it as "put the new link *inside* this folder", and quietly created `releases/v1/v2`, which is itself broken thanks to the Step 4.4 trap. `-n` tells `ln` to treat the existing link as a plain name and not follow it:

```
$ rm releases/v1/v2
$ ln -sfn releases/v2 current
$ cat current/app.txt
version 2
```

For links that point at folders, use `-sfn` every time.

> **Experiment: watch `ln` switch the link safely.** Programs may be reading `current` at the very moment you switch it. If `ln` first deleted the old link and then created the new one, there'd be a tiny gap where `current` didn't exist at all. See what recent versions of `ln` actually do (`strace` lists every request a program makes to the kernel):
>
> ```
> $ strace -e trace=symlinkat,renameat ln -sfn releases/v1 current
> symlinkat("releases/v1", AT_FDCWD, "current") = -1 EEXIST (File exists)
> symlinkat("releases/v1", AT_FDCWD, "CuNdRspU") = 0
> renameat(AT_FDCWD, "CuNdRspU", AT_FDCWD, "current") = 0
> +++ exited with 0 +++
> ```
>
> It tries the plain way first and gets "File exists". Then it makes the new link under a random temporary name and *renames* it over `current`. A rename replaces the old entry in one step, so there's no gap. Older versions of `ln` didn't do this, which is why you'll see deployment scripts spell it out by hand: `ln -s releases/v2 current_tmp && mv -T current_tmp current`. Now reread the atomic-write dilemma at the end of Part 3: this rename trick is the missing piece.

> **Experiment: follow a chain.** Commands on your system are often reached through symlinks, sometimes several in a row:
>
> ```
> $ which python3
> /usr/bin/python3
> $ readlink /usr/bin/python3
> python3.12
> $ readlink -f /usr/bin/python3
> /usr/bin/python3.12
> ```
>
> That's stock Ubuntu 24.04. `readlink` alone shows one step; `-f` follows the chain to the end. Your version number, and the number of steps, may differ: some systems go through `/etc/alternatives/` on the way. Then try `ls -ld /bin`. On current Ubuntu, `/bin` itself is a symlink to `usr/bin`.

### Step 4.7 The Python side

```python
import os

os.symlink("config_dev.json", "config.json")      # like ln -s (fails if config.json exists)
os.readlink("config.json")                         # like readlink: returns the stored text
os.path.islink("config.json")                      # True for the link itself
os.path.realpath("config.json")                    # like readlink -f: the end of the chain
```

`os.symlink` has no `-f`. If the name exists, you get `FileExistsError`. The safe way to switch a link from Python is the same trick `ln` uses: make the new link under a temporary name, then rename it into place. Try it on `config.json`:

```
$ python3 -c "
import os
os.symlink('config_dev.json', 'config.json.new')
os.replace('config.json.new', 'config.json')
print(os.readlink('config.json'))
"
config_dev.json
```

`os.replace` is a rename that's allowed to overwrite whatever has the destination name.

**Hard link or symlink?** Both give one file several ways in. Pick with these differences in mind:

| | Hard link | Symbolic link |
|---|---|---|
| What it is | another name for the same inode | a small file holding a path |
| Survives deleting the target's name | yes | no, it dangles |
| Can point at a directory | no | yes |
| Can cross filesystems | no | yes |
| Can be switched to another target in one step | no | yes (rename a new link over it) |
| Easy to spot | no, `find -samefile` | yes, `ls -l` shows `->` |

---

## Part 5: File descriptors: the numbers a program holds

Names live in directories and files live in inodes. A running program uses neither directly. When it opens something, the kernel hands back a small number, a **file descriptor**, and from then on the program says "read from 3" or "write to 1". Think of a coat-check ticket: the program holds the ticket, the kernel holds the coat.

Descriptors are the reason so many different things feel alike. A regular file, a pipe, a terminal, a network connection: once opened, each is just a number you can `read` and `write`.

### Step 5.1 Set up and look at your shell's descriptors

```
$ mkdir -p ~/lab05/part5
$ cd ~/lab05/part5
$ ls -l /proc/$$/fd
total 0
lrwx------ 1 codespace codespace 64 Sep 30 16:44 0 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:44 1 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:44 2 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:44 255 -> /dev/pts/0
```

`$$` is your shell's process ID, and `/proc/<id>/fd` is a folder the kernel invents on demand, with one entry per open descriptor. (Part 6 has more on `/proc`.) Every program starts with three:

| Number | Name | Normally connected to |
|---|---|---|
| 0 | standard input (stdin) | the keyboard, via the terminal |
| 1 | standard output (stdout) | the screen, via the terminal |
| 2 | standard error (stderr) | the screen, via the terminal |

All three point at `/dev/pts/0`, your terminal. (Bash keeps an extra one, 255, for its own use.) If you have two terminals open, run `tty` in each: they'll say `/dev/pts/0` and `/dev/pts/1`, or similar.

### Step 5.2 Watch a program's descriptors

Create `fd_peek.py`:

```python
# fd_peek.py: open a few things and look at this process's descriptor table.
import os

pid = os.getpid()
print("my process id:", pid)

f = open("notes.txt", "w")              # a regular file
r, w = os.pipe()                        # an unnamed pipe: two descriptors, one per end

print("notes.txt got descriptor", f.fileno())
print("the pipe got descriptors", r, "(read end) and", w, "(write end)")
print()

for fd in sorted(os.listdir(f"/proc/{pid}/fd"), key=int):
    try:
        target = os.readlink(f"/proc/{pid}/fd/{fd}")
    except FileNotFoundError:
        continue    # listdir used a descriptor of its own while reading; it's already closed
    print(f"  fd {fd:>2} -> {target}")

f.close()
```

```
$ python3 fd_peek.py
my process id: 3265
notes.txt got descriptor 3
the pipe got descriptors 4 (read end) and 5 (write end)

  fd  0 -> /dev/pts/0
  fd  1 -> /dev/pts/0
  fd  2 -> /dev/pts/0
  fd  3 -> /home/codespace/lab05/part5/notes.txt
  fd  4 -> pipe:[8881]
  fd  5 -> pipe:[8881]
```

New descriptors take the lowest free number, so the first file you open is almost always 3. The pipe has no name on disk, so the kernel describes it as `pipe:[8881]`; both ends show the same number because they're two ends of one pipe.

About that `try`/`except`: `os.listdir` needs a descriptor of its own to read the folder, so it briefly appears in the list and is gone by the time the loop asks about it. Looking changes what you're looking at, just a little.

### Step 5.3 Redirection rewires the numbers

Predict first: what will descriptor 1 point at?

```
$ python3 fd_peek.py > out.txt
$ cat out.txt
my process id: 3267
notes.txt got descriptor 3
the pipe got descriptors 4 (read end) and 5 (write end)

  fd  0 -> /dev/pts/0
  fd  1 -> /home/codespace/lab05/part5/out.txt
  fd  2 -> /dev/pts/0
  fd  3 -> /home/codespace/lab05/part5/notes.txt
  fd  4 -> pipe:[8891]
  fd  5 -> pipe:[8891]
$ python3 fd_peek.py | cat
...
  fd  1 -> pipe:[8901]
...
```

The program never changed. It prints to descriptor 1 every time, and the shell decided beforehand what 1 would be: the terminal, a file, or the write end of a pipe into `cat`. That's the whole trick behind `>`, `<`, `2>` and `|`: they set up the numbers before the program starts. The program doesn't need to know or care.

> **Experiment: `ls` watching itself.** Run `ls -l /proc/self/fd`. `self` always means "whichever process is asking", so here it's `ls` looking at its own table. You'll find an extra descriptor, 3, pointing at `/proc/<some number>/fd`: the folder `ls` opened in order to list it.

### Step 5.4 `/dev/stdin` isn't the keyboard

```
$ ls -l /dev/stdin /dev/stdout /dev/stderr
lrwxrwxrwx 1 root root 15 Sep 30 16:28 /dev/stderr -> /proc/self/fd/2
lrwxrwxrwx 1 root root 15 Sep 30 16:28 /dev/stdin -> /proc/self/fd/0
lrwxrwxrwx 1 root root 15 Sep 30 16:28 /dev/stdout -> /proc/self/fd/1
```

Symlinks (Part 4) into `/proc/self/fd`. So `/dev/stdin` means "whatever descriptor 0 of the program opening it happens to be". Often that's the keyboard, but not always:

```
$ echo "not from the keyboard" | cat /dev/stdin
not from the keyboard
```

### Step 5.5 A file with no name

Part 2's model freed an inode the moment its last name was removed. The real rule has one more condition: an inode is freed when it has no names left **and** no program has it open. Create `ghost.py`:

```python
# ghost.py: delete a file while it is still open, then read it anyway.
import os

with open("ghost.txt", "w") as f:
    f.write("still here\n")

f = open("ghost.txt")                  # hold it open
print("my process id:", os.getpid())
info = os.fstat(f.fileno())
print("before rm: inode", info.st_ino, "links", info.st_nlink)

os.remove("ghost.txt")                 # remove the only name
print("exists by name?", os.path.exists("ghost.txt"))
print("after rm:  inode", os.fstat(f.fileno()).st_ino, "links", os.fstat(f.fileno()).st_nlink)
print("/proc says:", os.readlink(f"/proc/self/fd/{f.fileno()}"))
print("reading anyway:", f.read().strip())

input("Press Enter to close it for good... ")
f.close()
```

```
$ python3 ghost.py
my process id: 3303
before rm: inode 794723 links 1
exists by name? False
after rm:  inode 794723 links 0
/proc says: /home/codespace/lab05/part5/ghost.txt (deleted)
reading anyway: still here
Press Enter to close it for good...
```

Leave it waiting there. The file has zero names and is still perfectly readable through the open descriptor.

**In your second terminal**, rescue it by copying the data out through `/proc`. Use the process id your `ghost.py` printed in place of `3303`:

```
$ cd ~/lab05/part5
$ ls -l /proc/3303/fd
total 0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 0 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 1 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 2 -> /dev/pts/0
lr-x------ 1 codespace codespace 64 Sep 30 16:45 3 -> /home/codespace/lab05/part5/ghost.txt (deleted)
$ cp /proc/3303/fd/3 rescued.txt
$ cat rescued.txt
still here
```

Now press Enter in the first terminal. Only at that moment is the space really given back.

This isn't a party trick. It's how a server's disk can be "full" after someone deleted a giant log file: the program writing the log still has it open, so the space isn't freed until that program closes it or restarts. `lsof +L1` lists every open file that has lost all its names.

> **Try another way.** `lsof -p <pid>` shows a process's open files with extra detail, such as the type and the current read position. It isn't always installed; the `/proc/<pid>/fd` folder always works.

### Step 5.6 Three ideas side by side

| | Name | Inode | File descriptor |
|---|---|---|---|
| Lives in | a directory, on disk | the filesystem, on disk | the kernel's memory, per process |
| Looks like | `final.txt` | `786880` | `3` |
| Unique within | one directory | one filesystem | one process |
| Lasts until | `rm` or `mv` | no names and no open descriptors | the program closes it or exits |
| How you see it | `ls` | `ls -i`, `stat` | `ls -l /proc/<pid>/fd` |

Two processes can both have a descriptor 3 pointing at completely different files, just as two coat-check counters can both hand out ticket 3.

---

## Part 6: `/proc` and `/sys`: the kernel as files

Nothing in `/proc` or `/sys` is stored on any disk. When you read one of these files, the kernel writes the answer on the spot, fresh, just for you. It's the "everything is a file" idea used in the other direction: instead of a new command for every question you might ask the kernel, there's a file to read.

### Step 6.1 Set up and ask a few questions

```
$ mkdir -p ~/lab05/part6
$ cd ~/lab05/part6
$ cat /proc/uptime
33.57 58.13
$ cat /proc/loadavg
0.06 0.05 0.01 1/94 1321
$ head -3 /proc/meminfo
MemTotal:        8223868 kB
MemFree:         7581244 kB
MemAvailable:    7704136 kB
$ grep -m1 "model name" /proc/cpuinfo
model name	: Intel(R) Xeon(R) Processor @ 2.80GHz
$ cat /proc/version
Linux version 6.18.44-fc-v50 (builder@sandboxing) (gcc (GCC) 15.3.0, GNU ld (GNU Binutils) 2.46) #1 SMP PREEMPT_DYNAMIC @0
```

Seconds since boot (the second number is total idle time added up across CPUs), system load averaged over 1, 5 and 15 minutes, memory, CPU model, kernel version. All of this will differ on your machine; a Codespace typically reports an `azure` kernel. Commands like `uptime`, `free` and `top` are mostly polite wrappers around these files: they read them and format the numbers nicely.

### Step 6.2 The sizes lie

Predict first: how big is `/proc/cpuinfo`?

```
$ ls -l /proc/cpuinfo
-r--r--r-- 1 root root 0 Sep 30 16:28 /proc/cpuinfo
$ wc -c /proc/cpuinfo
2434 /proc/cpuinfo
```

`ls` says 0 bytes. `wc`, which actually reads the file, counts thousands. The content doesn't exist until someone reads it, so there's nothing to measure beforehand. Programs that check a file's size and then read exactly that many bytes get nothing from `/proc`. It's a good reminder that "everything is a file" means "can be read like a file", not "behaves like a file on disk in every way".

### Step 6.3 Every process gets a folder

```
$ ls /proc | head -4
1
10
11
12
$ ls /proc/$$
```

Each number is a running process's ID, and inside each folder is that process described as files: `cmdline` (how it was started), `status` (state and memory), `cwd` (a symlink to its current folder), `fd` (Part 5), and dozens more.

```
$ grep -E "^(Name|State|VmRSS)" /proc/$$/status
Name:	bash
State:	S (sleeping)
VmRSS:	    7604 kB
$ ls -l /proc/$$/cwd
lrwxrwxrwx 1 codespace codespace 0 Sep 30 16:40 /proc/2407/cwd -> /home/codespace/lab05/part6
```

`S (sleeping)` is right: your shell spends nearly all its time waiting for you to type.

> **Experiment: the moving target.** Run `ls -l /proc/self` three times. It's a symlink whose target changes every time, because each `ls` is a new process asking about itself. Compare with `echo $$`, which stays the same: that's the shell's number, and the shell is still the same process.

### Step 6.4 `/sys`: devices as folders

`/sys` is organised around hardware and drivers. Network interfaces are a good place to look, since they're one of the things with no entry in `/dev`:

```
$ ls /sys/class/net
eth0  ifb0  ifb1  lo
$ cat /sys/class/net/eth0/address
02:fc:00:00:00:01
$ cat /sys/class/net/eth0/statistics/rx_bytes
2511751
```

`rx_bytes` is the number of bytes received on that interface so far. `lo` is the loopback interface, the machine talking to itself; the others depend on the machine (the `ifb` ones are a quirk of the test system). Yours may be called `eth0`, `ens3`, `enp0s3` or similar; use one from your own list. Read `rx_bytes` again after running something that uses the network, like `pip download requests`, and watch it climb.

> **Try another way.** `watch -n 1 cat /sys/class/net/eth0/statistics/rx_bytes` rereads it every second. Ctrl+C stops it.

**Writing, briefly.** Some files in `/proc/sys` and `/sys` accept writes, and writing to them changes kernel settings on the spot. That's powerful enough that only root may do it, and in a Codespace most of them are locked read-only even for root, because a container shouldn't reconfigure the machine it shares. Reading is always fine.

### Step 6.5 A system report from nothing but files

Create `sysreport.py`:

```python
# sysreport.py: a tiny system report built only by reading files.

def read(path):
    with open(path) as f:
        return f.read()

uptime_seconds = float(read("/proc/uptime").split()[0])

meminfo = {}
for line in read("/proc/meminfo").splitlines():
    key, value = line.split(":")
    meminfo[key] = int(value.split()[0])          # the numbers are in kB

cpu_count = read("/proc/cpuinfo").count("processor\t:")
load_1min = read("/proc/loadavg").split()[0]

print(f"up for      {uptime_seconds / 60:.1f} minutes")
print(f"CPUs        {cpu_count}")
print(f"load (1m)   {load_1min}")
print(f"memory      {meminfo['MemAvailable'] / 1024:.0f} MB available of {meminfo['MemTotal'] / 1024:.0f} MB")
print(f"hostname    {read('/proc/sys/kernel/hostname').strip()}")
```

```
$ python3 sysreport.py
up for      3.8 minutes
CPUs        2
load (1m)   0.06
memory      7529 MB available of 8031 MB
hostname    vm
```

No special library, no system calls you haven't met: `open` and `read`, the same as for a text file.

> **Try another way.** Python can answer some of these directly: `os.cpu_count()`, `os.getloadavg()`, `socket.gethostname()`. Those don't necessarily read `/proc`. `os.getloadavg()` on Ubuntu 24.04, for instance, asks the kernel through a dedicated system call called `sysinfo`. So the file interface is one door to this information, not the only one. (If you have `strace`, you can check for yourself: `strace python3 -c "import os; os.getloadavg()" 2>&1 | grep sysinfo`.)

---

## Part 7: Pipes and named pipes

A pipe is a one-way channel between two programs: whatever one writes into the write end, the other reads out of the read end, in the same order. Nothing lands on disk. The kernel holds the bytes in a small buffer in memory until they're read.

### Step 7.1 Set up and catch a pipe in the act

```
$ mkdir -p ~/lab05/part7
$ cd ~/lab05/part7
$ ls -l /proc/self/fd | cat
total 0
lrwx------ 1 codespace codespace 64 Sep 30 16:41 0 -> /dev/pts/0
l-wx------ 1 codespace codespace 64 Sep 30 16:41 1 -> pipe:[5082]
lrwx------ 1 codespace codespace 64 Sep 30 16:41 2 -> /dev/pts/0
lr-x------ 1 codespace codespace 64 Sep 30 16:41 3 -> /proc/2568/fd
```

`ls` is describing its own descriptors (Part 5), and its standard output, descriptor 1, is `pipe:[5082]`: the write end of the pipe the shell built when you typed `|`. `cat` got the read end as its descriptor 0. Neither program knows the other exists; each just uses 0 and 1 as always.

A pipe made with `|` has no name. It exists only between the two programs the shell connected, and it disappears when they finish.

### Step 7.2 A pipe with a name

A **named pipe**, or FIFO (first in, first out), is the same thing with an entry in a directory, so any two programs can find it.

```
$ mkfifo my_pipe
$ ls -l my_pipe
prw-rw-r-- 1 codespace codespace 0 Sep 30 16:32 my_pipe
```

**In terminal 1**, start a reader:

```
$ cd ~/lab05/part7
$ cat my_pipe
```

It just sits there. Opening a FIFO for reading waits until someone opens it for writing, and the other way round. Neither side can start alone.

**In terminal 2**, start a writer:

```
$ cd ~/lab05/part7
$ cat > my_pipe
```

Type a few lines, pressing Enter after each. They appear in terminal 1 as you go. Press Ctrl+D in terminal 2 to finish writing. Terminal 1's `cat` ends too: when every writer has closed its end, the reader gets end-of-file.

Check the size afterwards: `ls -l my_pipe` still says 0. The bytes went through the kernel, not through the disk.

### Step 7.3 The Python version: a writer and a reader

Create `writer.py`:

```python
# writer.py: send lines into the named pipe until you type exit.
PIPE = "my_pipe"

print("Opening the pipe... (this waits until a reader opens the other end)")
with open(PIPE, "w") as pipe:
    print("Reader connected. Type messages; type exit to finish.")
    while True:
        message = input("> ")
        if message == "exit":
            break
        pipe.write(message + "\n")
        pipe.flush()               # push it through now, not when a buffer fills up
print("Writer closed its end of the pipe.")
```

Create `reader.py`:

```python
# reader.py: print every line that arrives through the named pipe.
PIPE = "my_pipe"

print("Opening the pipe... (this waits until a writer opens the other end)")
with open(PIPE) as pipe:
    print("Writer connected. Waiting for messages.")
    for line in pipe:              # blocks until a line arrives; ends when the writer closes
        print("Received:", line.rstrip("\n"))
print("The writer closed the pipe, so there is nothing more to read. Bye.")
```

Run `python3 reader.py` in terminal 1 and `python3 writer.py` in terminal 2 (either order). Type a few messages, then `exit`:

Here are the two terminals side by side:

```
terminal 2                                terminal 1
$ python3 writer.py                       $ python3 reader.py
Opening the pipe... (this waits ...)      Opening the pipe... (this waits ...)
Reader connected. Type messages; ...      Writer connected. Waiting for messages.
> hello                                   Received: hello
> second line                             Received: second line
> exit                                    The writer closed the pipe, so there is
Writer closed its end of the pipe.        nothing more to read. Bye.
```

The reader needs no special "stop" message. Closing the write end *is* the stop signal: the `for` loop ends by itself on end-of-file, exactly as it does at the end of a text file.

**If you've seen an older version of these two scripts:** it had the writer send the word `stop` when you typed `exit`, while the reader waited for the word `exit`, so the reader never recognised the signal. Worse, the reader treated an empty `readline()` as "nothing yet, sleep a second and try again". On a pipe, `readline()` doesn't return early; it waits for data. An empty result means the writer is gone for good, so that reader kept sleeping and retrying forever and had to be killed with Ctrl+C. Letting end-of-file end the loop avoids both problems.

> **Experiment: `cat` gives up, `tail -f` doesn't.** In terminal 1 run `tail -f my_pipe`. In terminal 2 run `echo one > my_pipe`, then `echo two > my_pipe`. Both lines show up in terminal 1, and `tail` keeps waiting even though each `echo` closed its end. Now try the same with `cat my_pipe` in terminal 1: it ends after the first `echo`. `tail -f` is built to keep watching after end-of-file (that's what `-f`, "follow", means), which makes it handy for listening to a series of writers. Ctrl+C stops it.

> **Experiment: how big is a pipe?** Paste this into a Python session (`python3`), or save it as a file:
>
> ```python
> import os
> r, w = os.pipe()
> os.set_blocking(w, False)          # fail instead of waiting when the pipe is full
> total = 0
> try:
>     while True:
>         total += os.write(w, b"x" * 1024)
> except BlockingIOError:
>     pass
> print("the pipe held", total, "bytes before the writer would have to wait")
> ```
>
> ```
> the pipe held 65536 bytes before the writer would have to wait
> ```
>
> 64 KiB. Nobody ever reads from `r` here, so the pipe fills up. Normally the writer would then simply pause until the reader catches up, which is how `|` keeps a fast program from burying a slow one. `set_blocking(False)` turns that pause into an error so you can see where the limit is.

> **Try another way.** The file-types script from Part 1 used a thread as the writer so a single program could play both roles. That's fine for a demo, but the normal use of a named pipe is two separate programs, possibly started by different people, which is the whole point of giving it a name.

---

## Part 8: The terminal is a file too

Every program in Part 5 had descriptors 0, 1 and 2 pointing at `/dev/pts/0`. That's your terminal, and it's a character device like `/dev/null`. Between your keyboard and the program sits a layer of the kernel called the *line discipline*, which quietly does a lot of work: showing the letters you type, letting Backspace fix mistakes, and holding a line back until you press Enter. This part lets you switch those jobs off one at a time.

**Before you start:** a couple of these steps make your terminal behave strangely on purpose. If it ever stops showing what you type or stops responding to Enter normally, type `stty sane` and press Enter (even if you can't see the letters), or type `reset`. Closing the terminal tab and opening a new one also always works.

### Step 8.1 Set up and find your terminal

```
$ mkdir -p ~/lab05/part8
$ cd ~/lab05/part8
$ tty
/dev/pts/0
$ ls -l $(tty)
crw------- 1 codespace codespace 136, 0 Sep 30 16:41 /dev/pts/0
```

`c`, a character device, owned by you. `pts` stands for pseudo-terminal slave: a terminal made of software, which is what every terminal window is these days. The permissions may look slightly different on your system.

**With two terminals open,** run `tty` in the second one too. Say it prints `/dev/pts/1`. Then, from the first:

```
$ echo "hello from next door" > /dev/pts/1
```

The text appears in the other terminal. You wrote to a terminal the same way you'd write to a file, because as far as `echo` is concerned, it is one.

### Step 8.2 Look at the switches

```
$ stty -a
speed 38400 baud; rows 24; columns 80; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>;
eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R;
werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
-parenb -parodd -cmspar cs8 -hupcl -cstopb cread -clocal -crtscts
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl ixon -ixoff
-iuclc -ixany -imaxbel -iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofill -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -xcase -tostop -echoprt
echoctl echoke -flusho -extproc
```

The first lines map keys to jobs: Ctrl+C interrupts (`intr`), Ctrl+D means end-of-file (`eof`). The long lists are switches, where a leading `-` means "off". Find two of them near the end: `icanon` and `echo`. (Your `rows` and `columns` match your window size.)

- **`echo`**: the terminal shows the characters you type.
- **`icanon`** (canonical mode): input is collected into lines. The program receives nothing until you press Enter, and Backspace edits the line before it's sent.

These are two separate switches. Some notes describe canonical mode as "line-buffered with echo on", as if they came as a pair, but you can turn either one off without the other, as the next steps show.

### Step 8.3 Echo off

```
$ stty -echo
$ cat
```

Type `secret` and press Enter. You don't see your typing, but `cat` still receives the line and prints it back once:

```
secret
```

Press Ctrl+D to end `cat`, then turn echo back on (you won't see yourself typing this either):

```
$ stty echo
```

This is how password prompts work. Python wraps it up for you:

```
$ python3 -c "import getpass; p = getpass.getpass('Password: '); print('length', len(p))"
Password:
length 7
```

`getpass` turns echo off, reads one line, and turns echo back on.

### Step 8.4 Ctrl+D isn't a signal

Run `cat` and type `abc` without pressing Enter. Then press Ctrl+D once.

```
$ cat
abcabc
```

`cat` printed `abc` right away, even though you never pressed Enter, and it's still running. Press Ctrl+D again on the empty line and `cat` ends.

What Ctrl+D actually means is "send whatever has been typed so far, right now". The first press sent `abc`. The second press sent nothing at all, and a read that gets zero bytes is exactly how a program recognises end-of-file. So Ctrl+D isn't a special character that travels to the program, and it isn't a signal like Ctrl+C. It's the terminal handing over an empty line, which programs agree to treat as "the end".

### Step 8.5 Canonical mode off

```
$ stty -icanon
$ cat
```

Type `ab` slowly. Each letter appears twice: `aabb`. The first copy is the terminal echoing your key; the second is `cat`, which now receives every key the instant you press it instead of waiting for Enter, and prints it straight back.

Now try Ctrl+D. It shows up as `^D` and `cat` keeps going: with canonical mode off, there are no lines to send, so the "send the line now" key has no meaning. Use Ctrl+C to stop `cat`, then switch back:

```
$ stty icanon
```

Editors like `nano` and `vim`, and programs like `less` and `top`, run in this mode, which is how they react to single keys.

### Step 8.6 Single keys from Python

Create `keys.py`:

```python
# keys.py: read single keypresses, no Enter needed.
import sys
import termios
import tty

fd = sys.stdin.fileno()
saved = termios.tcgetattr(fd)          # remember the terminal's current settings
try:
    tty.setcbreak(fd)                  # no line buffering, no echo; Ctrl+C still works
    print("Press some keys (try the arrow keys too). Press q to quit.")
    while True:
        ch = sys.stdin.read(1)
        if ch == "q":
            break
        print(f"got {ch!r}  (code {ord(ch)})")
finally:
    termios.tcsetattr(fd, termios.TCSADRAIN, saved)   # always put the settings back
    print("Terminal settings restored.")
```

```
$ python3 keys.py
Press some keys (try the arrow keys too). Press q to quit.
got 'a'  (code 97)
got '\x1b'  (code 27)
got '['  (code 91)
got 'A'  (code 65)
Terminal settings restored.
```

That's `a` followed by the up arrow. The arrow key isn't one character: it arrives as three, the Escape character followed by `[A`. Try the other arrows, Enter, Tab and Backspace. Every key on the keyboard, however special it seems, reaches the program as ordinary bytes on descriptor 0.

The `try`/`finally` matters. If the program crashed without restoring the settings, you'd be left with a terminal that doesn't echo, which is exactly the mess `stty sane` exists to clean up.

> **Experiment: bypass the redirection.** `/dev/tty` always means "the terminal this program belongs to", whatever descriptors 0, 1 and 2 have been redirected to. Try:
>
> ```
> $ echo "to the file" > out.txt; echo "to the screen" > /dev/tty
> $ ( echo "normal output"; echo "straight to you" > /dev/tty ) > out2.txt
> ```
>
> Only `normal output` ends up in `out2.txt`. Programs like `ssh` and `sudo` use `/dev/tty` to ask for a password even when their output is going into a pipe.

---

## Part 9: Sockets

A pipe goes one way and connects two programs that can find the same name. A **socket** goes both ways, and the program on the other end can be on the same machine or anywhere on the network. It's the most "file-like" of all the non-files: once connected, a program reads and writes through a descriptor as usual. What's different is how you get connected.

The usual arrangement has two roles. A **server** picks an address, waits there, and accepts whoever connects. A **client** connects to that address. After that, both sides are equal and either can send.

```
server:  socket() -> bind(address) -> listen() -> accept() -> recv/send ... -> close()
client:  socket() -------------------------------> connect(address) -> send/recv ... -> close()
```

### Step 9.1 Set up

```
$ mkdir -p ~/lab05/part9
$ cd ~/lab05/part9
```

Create `server.py`:

```python
# server.py: a small TCP chat server. Each client gets its own thread.
import os
import socket
from threading import Thread

HOST = "localhost"      # only this machine can connect
PORT = 65432


def handle_client(conn, addr):
    with conn:
        conn.sendall(b"Hello from the socket server!")
        while True:
            try:
                data = conn.recv(1024)
            except ConnectionResetError:
                break                           # the client vanished without saying goodbye
            if not data:                        # b"" means the client closed its end
                break
            text = data.decode("utf-8")
            print(f"From {addr}: {text!r}")
            conn.sendall(f"Got it: {text}".encode("utf-8"))
    print(f"Connection with {addr} closed.")


def main():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
        # Let a restarted server reuse the port right away instead of waiting about a minute
        server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server.bind((HOST, PORT))
        server.listen()
        print(f"Listening on {HOST}:{PORT} as process {os.getpid()}. Press Ctrl+C to stop.")
        while True:
            conn, addr = server.accept()
            print(f"Connected by {addr}")
            # daemon=True: these threads won't keep the program alive after Ctrl+C
            Thread(target=handle_client, args=(conn, addr), daemon=True).start()


try:
    main()
except KeyboardInterrupt:
    print("\nServer stopped.")
```

Create `client.py`:

```python
# client.py: talk to server.py. Type exit to leave.
import socket

HOST = "localhost"
PORT = 65432

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    sock.connect((HOST, PORT))
    print(sock.recv(1024).decode("utf-8"))      # the welcome message

    while True:
        msg = input("You: ")
        if msg.lower() == "exit":
            break
        if not msg:
            continue                            # sending zero bytes sends nothing, and we'd wait forever
        sock.sendall(msg.encode("utf-8"))
        reply = sock.recv(1024)
        if not reply:
            print("The server closed the connection.")
            break
        print("Server:", reply.decode("utf-8"))

print("Connection closed.")
```

`AF_INET` means an internet (IPv4) address, and `SOCK_STREAM` means TCP: a reliable, ordered stream of bytes. The address is a pair: a host name and a port number.

### Step 9.2 Talk

**Terminal 1:**

```
$ python3 server.py
Listening on localhost:65432 as process 3331. Press Ctrl+C to stop.
```

**Terminal 2:**

```
$ python3 client.py
Hello from the socket server!
You: hello
Server: Got it: hello
You: second
Server: Got it: second
You: exit
Connection closed.
```

Meanwhile, terminal 1 shows each message arriving:

```
Connected by ('127.0.0.1', 50318)
From ('127.0.0.1', 50318): 'hello'
From ('127.0.0.1', 50318): 'second'
Connection with ('127.0.0.1', 50318) closed.
```

The `50318` is the client's own port, picked at random by the system for this one connection. Open a third terminal and start another client while the first is still connected: the threads let the server talk to both at once.

**If you've seen an older version of these scripts,** three things changed:

1. **Pressing Enter on an empty line froze the old client.** Sending zero bytes sends nothing at all, so the server never answered, and the client waited for a reply forever. The new client skips empty lines.
2. **Ctrl+C couldn't stop the old server while a client was connected.** The main program stopped, but the thread serving the client kept the process alive until that client left. The new threads are `daemon=True`, which means "don't wait for me on the way out".
3. **The welcome message and cleanup** now live inside the client's thread and a `with` block, so every connection gets closed however it ends.

### Step 9.3 A network connection, seen as a file

Keep the server running (restart it if you stopped it). It printed its process id when it started; use yours in place of `3331`. In terminal 2:

```
$ ls -l /proc/3331/fd
total 0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 0 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 1 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 2 -> /dev/pts/0
lrwx------ 1 codespace codespace 64 Sep 30 16:45 3 -> socket:[9170]
```

The listening socket is descriptor 3, right alongside the terminal. Like the pipe in Part 7, it has no name on disk, so the kernel shows a number instead.

That number isn't random. The kernel keeps a table of TCP sockets as, of course, a file:

```
$ grep -i ff98 /proc/net/tcp
   0: 0100007F:FF98 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1001        0 9170 1 ...
```

Decode it: `0100007F` is 127.0.0.1 written backwards in hexadecimal, `FF98` is hex for 65432, `0A` is the state "listening", `1001` is your user ID, and `9170` is the same number as `socket:[9170]`. That's how tools find out which program owns which network port.

> **Try another way.** `ss` reads those tables for you and prints them readably:
>
> ```
> $ ss -tlnp | grep 65432
> LISTEN 0      128        127.0.0.1:65432      0.0.0.0:*    users:(("python3",pid=3331,fd=3))
> ```
>
> `-t` TCP, `-l` listening, `-n` numbers instead of names, `-p` show the program. Connect a client and run `ss -tnp | grep 65432` (no `-l`) to see the connection from both ends.

### Step 9.4 It's only bytes

The client doesn't have to be `client.py`. `nc` (netcat) connects to any address and shows you whatever bytes arrive:

```
$ printf "hi from nc" | nc -q 1 localhost 65432
Hello from the socket server!Got it: hi from nc
```

(`-q 1` means "quit one second after the input runs out". If `nc` complains about `-q`, try without it.) Two messages came out stuck together on one line. There's no newline between them because the server never sent one. TCP delivers a stream of bytes, in order, with no notion of where one message ends and the next begins. Any boundaries have to be part of what you send.

> **Experiment: messages merge.** Save as `burst.py` and run it a few times while the server is running:
>
> ```python
> import socket, time
> s = socket.create_connection(("localhost", 65432))
> print(s.recv(1024))
> s.sendall(b"one")
> s.sendall(b"two")          # sent right away, without waiting for a reply
> time.sleep(0.3)
> print(s.recv(1024))
> s.close()
> ```
>
> Sometimes the server's log shows `'onetwo'` as a single message; sometimes it shows `'one'` and `'two'` separately, and then the client receives `Got it: oneGot it: two` in one piece. It depends on timing you don't control. `client.py` gets away with `recv(1024)` because it always waits for a reply before sending again, and its messages are short. Real protocols settle the question with a rule, such as "every message ends with a newline" or "every message starts with its length". The next step uses the newline rule.

### Step 9.5 A socket that *is* a file

Swap `AF_INET` for `AF_UNIX` and the address becomes a file name, just like the socket in Part 1. This kind only works within one machine, and many local services use it: databases, Docker, the desktop's sound system. Create `unix_server.py`:

```python
# unix_server.py: the same idea as server.py, but the address is a FILE NAME.
import os
import socket

ADDRESS = "chat.sock"

if os.path.exists(ADDRESS):
    os.remove(ADDRESS)          # a leftover from last time would block bind()

with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as server:
    server.bind(ADDRESS)        # this line creates chat.sock on disk
    server.listen()
    print(f"Listening on {ADDRESS}. Press Ctrl+C to stop.")
    try:
        while True:
            conn, _ = server.accept()
            with conn:
                # makefile() wraps the socket in an ordinary file object
                stream = conn.makefile("rw", encoding="utf-8")
                for line in stream:                 # read it like a text file, line by line
                    stream.write(line.upper())      # and write back like a text file
                    stream.flush()
    except KeyboardInterrupt:
        print("\nStopped.")
    finally:
        os.remove(ADDRESS)      # tidy up the socket file
```

Stop the TCP server (Ctrl+C), and run this one in terminal 1. In terminal 2:

```
$ ls -l chat.sock
srwxrwxr-x 1 codespace codespace 0 Sep 30 16:34 chat.sock
$ printf "hello there\nsecond line\n" | nc -q 1 -U chat.sock
HELLO THERE
SECOND LINE
```

`-U` tells `nc` the address is a Unix socket file. Look at the server's loop: `for line in stream`, `stream.write`, `stream.flush`. That's word for word how you'd process a text file. `makefile()` wraps the socket in a file object, and because each message ends in a newline, "read one line" is exactly "read one message", so the merging problem from Step 9.4 disappears.

Stop the server with Ctrl+C and check that `chat.sock` is gone. It removes the file on the way out, and deletes any leftover before `bind()` on the way in, because binding to a name that's already taken fails:

```
$ python3 -c "import socket; socket.socket(socket.AF_UNIX).bind('stale.sock')"
$ python3 -c "import socket; socket.socket(socket.AF_UNIX).bind('stale.sock')"
OSError: [Errno 98] Address already in use
$ rm stale.sock
```

The first program exited long ago, yet its socket file blocks the second. A TCP port is released by the kernel itself once the program ends, after a short cool-down that `SO_REUSEADDR` in `server.py` lets a restarted server skip. A socket file is a name in a directory, and names stay until someone removes them (Part 2), so cleaning up is the program's job.

> **Experiment: across machines.** Sockets become really interesting when the other end is another computer. On two machines that can reach each other, such as two lab VMs on the same network, change `HOST` in `server.py` to `"0.0.0.0"` ("accept connections arriving on any network interface"), run `hostname -I` to find the server's address, and put that address in `client.py`. A few honest caveats:
>
> - A Codespace runs in a data centre, not on your local network, so a classmate's laptop can't reach it by address. This experiment needs machines on the same network.
> - A firewall on the server may block the port. On Ubuntu with `ufw` enabled, `sudo ufw allow 65432/tcp` opens it.
> - `0.0.0.0` lets anyone who can reach the machine talk to your server, and this server believes everything it receives. Stop it when you're done.

---

## Reflection

Short answers are fine. Write them in your own words, and where a question asks "why", point to something you actually saw in the lab.

1. In Part 2, renaming a file kept its inode number and copying gave a new one. In your own words, what does `mv` change, and what does it leave alone?
2. Why can deleting `db_config.json` leave the projects working with hard links (Part 3) but break them with a symlink (Part 4)?
3. `sed -i` split the hard-linked config into two separate files without any error. Describe a real situation where that silent split could cause trouble, and how you'd catch it.
4. Part 5 showed a file with zero names that could still be read. Explain how a disk can stay full after someone deletes a huge log file, and what would free the space.
5. The same `python3 fd_peek.py` printed to the screen, into a file, and into a pipe without changing a line. Who decided where descriptor 1 pointed, and when?
6. Ctrl+D ended `cat` in Part 8, but only on an empty line. Explain what Ctrl+D really does.
7. Part 9's `unix_server.py` reads a network-style connection with `for line in stream`. What made reading line by line safe there, when `recv(1024)` in Step 9.4 could merge messages?
8. Name one thing on Linux that is *not* a file in any useful sense, and one thing that surprised you by being one.

---

## Cleanup

Everything this lab made lives in `~/lab05`, plus possibly one file in `/dev/shm` if an experiment stopped halfway:

```
$ cd ~
$ rm -rf ~/lab05
$ rm -f /dev/shm/moving.txt /dev/shm/config_copy
```

If any terminal is still acting strangely, run `stty sane` in it. If a server or `ghost.py` is still running in some terminal, press Ctrl+C (or Enter, for `ghost.py`) there.

---

## Troubleshooting

| What you see | Likely cause | What to do |
|---|---|---|
| `File exists` or `FileExistsError` | You ran a step twice | Delete the file it names, or start the part again in a fresh folder |
| A command just sits there and nothing happens | A named pipe is waiting for its other end (Part 7) | Open the other end in a second terminal, or press Ctrl+C |
| `No such file or directory` for a file `ls` shows | It's a symlink whose target is missing (Part 4) | `ls -l` it and check the path after `->` |
| Typing doesn't show up, or Enter behaves oddly | `stty -echo` or `stty -icanon` still active (Part 8) | Type `stty sane` and Enter, or `reset`, or open a new terminal |
| `OSError: [Errno 98] Address already in use` | A server is still running, or a stale socket file exists | Stop the other server; for a Unix socket, `rm` the `.sock` file |
| `ConnectionRefusedError` from `client.py` | The server isn't running, or is on a different port | Start `server.py` first, in another terminal |
| `PermissionError` in `safe_update.py` | The central file is read-only (by design) | Use the script's `safe_update`, or `chmod u+w` it yourself |
| `command not found` for `nc`, `lsof`, `strace`, `tmux` | Not installed | See the install line in [Before you start](#before-you-start) |
| `nc` rejects the `-q` option | A different version of netcat | Leave out `-q 1` and press Ctrl+C when it's done |
| No block devices in `/dev` | You're in a container (Codespace) | Normal; try on a VM to see them |
| After `su` to another user, `Permission denied` on `/dev/pts/...` | The terminal device still belongs to the user who opened it | Run `script -q /dev/null` for a fresh terminal owned by the new user; `exit` leaves it |

---

## Quick reference: shell and Python side by side

| Job | Shell | Python | Watch out for |
|---|---|---|---|
| Write a file | `echo "hi" > f` | `open("f", "w").write("hi")` | `echo` adds a newline and `write` doesn't, so the files differ by one byte; `diff` reports "No newline at end of file" |
| Append | `echo "hi" >> f` | `open("f", "a").write("hi\n")` | |
| Read | `cat f` | `open("f").read()` | |
| Create empty / update time | `touch f` | `Path("f").touch()` | `open("f", "a").close()` creates a missing file but does *not* update the time of an existing one |
| Rename | `mv a b` | `os.rename("a", "b")` | Across filesystems, `os.rename` fails; `shutil.move` copies instead (Part 2) |
| Copy | `cp a b` | `shutil.copy2("a", "b")` | `shutil.copy` doesn't keep the modification time; `copy2` does |
| Delete a name | `rm f` | `os.remove("f")` | Removes a *name*; the inode goes when no names and no open descriptors remain (Part 5) |
| Delete a folder tree | `rm -r d` | `shutil.rmtree("d")` | No undo in either |
| Make a folder path | `mkdir -p a/b/c` | `os.makedirs("a/b/c", exist_ok=True)` | |
| Hard link | `ln a b` | `os.link("a", "b")` | Not for directories, not across filesystems |
| Symlink | `ln -s target name` | `os.symlink("target", "name")` | Relative targets are read from the link's folder; no `-f` in Python, use temp name + `os.replace` |
| Where a link leads | `readlink -f name` | `os.path.realpath("name")` | |
| Inode number, link count | `stat -c "%i %h" f` | `os.stat("f").st_ino`, `.st_nlink` | `os.lstat` to examine a symlink itself |
| Permissions | `chmod 644 f` | `os.chmod("f", 0o644)` | Python needs the `0o` for octal |
| Named pipe | `mkfifo p` | `os.mkfifo("p")` | Opening one end waits for the other |
| List a folder | `ls` | `os.listdir(".")` | `listdir` includes hidden files; `ls` needs `-a` |
