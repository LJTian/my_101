### Create test lab (创建 测试 标签 )


```shell
$ mkdir testlab && cd $_

$ mkdir upper lower merged work

$ echo "from lower" > lower/in_lower.txt
$ echo "from upper" > upper/in_upper.txt
$ echo "from lower" > lower/in_both.txt
$ echo "from upper" > upper/in_both.txt
```

### Check it(检查目录情况)

```log
root@k8s-master:~/testlab# tree .
.
├── lower
│   ├── in_both.txt
│   └── in_lower.txt
├── merged
├── upper
│   ├── in_both.txt
│   └── in_upper.txt
└── work

4 directories, 4 files
```

### Combine (合并目录)

使用mount 挂载 新文件目录
```shell
$ sudo mount -t overlay overlay -o lowerdir=`pwd`/lower,upperdir=`pwd`/upper,workdir=`pwd`/work `pwd`/merged
```

### Check the merged file(检查合并的文件)

看一下谁被覆盖了

```shell
$ cat merged/in_both.txt

$ cat merged/in_lower.txt

$ cat merged/in_upper.txt
```

查一下新文件目录大小
```shell
$ df -h | grep testlab

overlay                             29G  9.7G   18G  36% /root/testlab/merged
```

### Delete merged file(删除文件)

```shell
$ delete merged/in_both.txt
$ delete merged/in_lower.txt
$ delete merged/in_upper.txt
```
