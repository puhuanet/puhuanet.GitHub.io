# Bash Shell输出json格式

```
#!/usr/bin/env bash

## input file format
# enable=1,   rep_server=server,   con_name=name

# output
# {"data":[{"enable":"1","rep_server":"server","con_name":"name"}]}

if [ $# != 1 ] ; then
    echo "{\"data\":[{\"USAGE\":\"$0 FILENAME\"},{\"e.g.\":\"$0 custom.net.tcp.service.cfg\"}]}"
    exit 1;
fi

file=$1
awk -v LL=$(wc -l < $file) '
BEGIN {FS="\\s*,\\s*"; ORS="";  print"{\"data\":["} 
{
    print "{";
    for (i=1;i<=NF;i++){
        split($i,arr,"="); 
        printf("\"%s\":\"%s\"",arr[1],arr[2]); 
        print i==NF?"":","
    };
    print LL==NR?"}":"},"
} 
END {print"]}"}' $file
```