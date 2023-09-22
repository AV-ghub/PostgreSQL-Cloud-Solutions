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





