## Catalog list
> Max file name lenth = 255
 
```
# files with attributres
ls -l

# root catalog
ls -l /

# home catalog
ls -l /home/

# current path
pwd

# hidden file
.file

# list with hidden files
cd ~
ls -la

# current catalog
.

# list current catalog
ls -l .
=
ls -l

# parent catalog
..

# list parent catalog
ls -la ..

# help
ls --help
```

## Touch

```
# create new file
touch file
ls -la

# change existing file time (file content will stay untoucheable)
touch file
ls -la

# create a bunch of files
touch file1 file2 file3
```

## Mkdir

```
cd ~
mkdir test
mkdir ./test/dir
=
mkdir test/dir
ls -la ./test/dir
touch ./test/dir/file
ls -la ./test/dir

# create multiple dirs
mkdir -p test/dir1/dir2/dir3

```

## Autocomplete
Multiple Tabs.

## Rm

```
# mask for rest of simbols
rm file*

# mask for single simbol
rm file?

# rem dir
rmdir

# rem all recursive
rm -r dir
```

## Cp Mv

```
# simple copy from - to
cp file1 ./dir2/

# copy with all content
cp -r ./dir2/ ./dir/

# params
cp --help

# move
mv file ./dir2/

# move with all content
mv ./dir2/ ./dir1/
```

## Type
```
# show type of command
type ls
type rm
type cd
```

## Echo
```
# vars
echo $HISTSIZE
echo $PATH
```
> Выполняется только первое вхождение.
> Если одна и та же команда будет расположена по двум путям, то выполняться будет только та, которая расположена по первому пути.

## Which
```
# show util path
which ls
```

## Who
```
# current server connections (list of terminals connected to currnet server)
[admin@igonin-vl bin]$ who
admin    :0           2023-09-22 09:39 (:0)
admin    pts/0        2023-09-22 09:39 (:0)
admin    pts/1        2023-09-22 09:39 (:0)
admin    pts/2        2023-09-22 09:39 (:0)
```
## Misc
```
# comp name
cat /etc/hostname
# hostname
[admin@igonin-vl ~]$ hostnamectl
   Static hostname: localhost.localdomain
Transient hostname: igonin-vl.ad.speechpro.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 149bf10b06d8fa4ab04ebc553cac67cb
           Boot ID: 5236415625934e49976111b7a7437729
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-1160.95.1.el7.x86_64
      Architecture: x86-64
# change hostname
sudo hostname anisimov-lws1
```

## Man

```
# More info compare to --help
man ls
```




