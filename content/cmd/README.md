
####MultiThread Downloader

```bash
 axel -n 10 -av -o filename link 
 ```

####Sed Usage

```bash

# -i: edit files in place
# /start/,/end/: set the range we want to edit. 
# Another usage is `2,5` that sets the line 2 to 5 range
# s/^/$/: s/regexp/replacement/
sed -i '/start/,/end/s/^/#/' files

# d: delete
sed -i '/start/,/end/d' files

# i: insert words evey line above in those range matches the pattern
sed -i '/start/,/end/i words' files

# a: append words evey line below in those range matches the pattern
sed -i '/start/,/end/a words' files

# c: replace the range with the words
sed -i '/start/,/end/c words' files

# -n: quiet,suppress automatic printing of pattern space.
# p: print the range
sed -n '/start/,/end/p' files

# '{ COMMANDS }': A group of commands 
# '/start1/        s/regexp1/replacement1/': deal with the line that contains 'start1'
sed -i '/start/,/end/{
	/start1/        s/regexp1/replacement1/
	/start2/        s/regexp2/replacement2/
}' files

# insert multi-lines
content="\
line1\n\
line2\n"
sed -i '1s/^/${content}/' files

```

#### diff&patch

```
diff -rupN origin new > original.patch
patch -s -p0 <original.patch
```

#### du

Too lookup the size of each directory

```
du -sh dir
```

#### Count the number of files

```
find . -type f | wc -l
```

#### Check processes which used the dir if mounting

```
fuser -muv dir
```

#### Write the buffer back to disk

```
sync
```

#### Rsync usage (increasing backup)

```
rsync -artlHSqP --delete -e "ssh -p #port" dir user@remote_ip:/path/to/dir
```
