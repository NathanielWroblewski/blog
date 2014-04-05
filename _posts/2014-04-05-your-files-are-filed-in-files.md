---
layout: post
title:  Your Files are Filed in Files
---

Folders in Unix are actually just files, binary files containing a list of the other files it contains (actually, filename-inode number pairs).  If you type `ls -l` in a directory
containing a folder and a file, you'll end up with something like this:

```sh
...
drwxr-xr-x   4 myusername  mygroup   136 Mar 28 01:15 folder_name
-rw-r--r--   1 myusername  mygroup   290 Mar 27 21:56 file_name.html
...
```

The first group of letters is common to all files. They're the access modes for the file, i.e. whether or not the file is readable, writeable, or executeable for a
given user, group, or 'other'.  If you have read access to a file, you can see what's in it.  If you have write access, you can change it.  If you have execute access, you can run it.  Typical files, like the one shown above, usually
start with a `-`.  Directories, on the other hand, start with a `d`.

So what does read, write, and execute mean for a directory file?  You probably use
it all the time!  Read access allows you to list the file names in the directory file, think `ls`.  Write access let's you delete file (or remove a file's entry from a directory file).  With execute access, you can `cd` into a directory; otherwise, you can't.

So, now, while your coworkers may insist that they're filing their files in folders, you'll know what's really up: they're filing their files in files.
