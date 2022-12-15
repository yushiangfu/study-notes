Many of the example snippets come from 'setup' in 'openbmc' project. 

- set

```
-x: enable debugging
+x: disable debugging
```

- -z: return true if the string length is zero, and that includes undefined and empty string.
```
if [ -z "$COLUMN" ]; then
    # If it is not, use 'cat'
    COLUMN=$(which cat)
fi
```

- true/false: note that the definitions are opposite to most programming languages.
```
$ true; echo $?
0
$ false; echo $?
1
```

- #: delete the shortest match from the start
- ##: delete the longest match from the start
```
$ cfg=meta-alibaba/meta-thor/conf/machine/thor.conf
$ echo ${cfg#*/}
meta-thor/conf/machine/thor.conf
$ echo ${cfg##*/}
thor.conf
```

- %: delete the shortest match from the _end_
- %%: delete the longest match from the _end_
```
$ cfg=meta-alibaba/meta-thor/conf/machine/thor.conf
$ echo ${cfg%/*}
meta-alibaba/meta-thor/conf/machine
$ echo ${cfg%%/*}
meta-alibaba
```

- sed

Remove target path in $PATH
```
echo $PATH | sed -re "s#(^|:)$newpath(:|$)#\2#g;s#^:##"
1. Remove 'newpath' from $PATH.
2. If it's the first path in $PATH, remove the remaining colon as well.
```
