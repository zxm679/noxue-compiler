
## 在线编译器原理

最基本的原理，利用docker运行编译执行命令

### 从这样一条命令开始

我们假设你已经安装好了docker

那么我们接下来安装ubuntu镜像

`docker pull ubuntu`

然后我们就相当于拥有一台 ubuntu 系统的电脑了，`在其中做的人和操作都不会影响到主机，哪怕是格式化系统`

### 如何在docker中执行命令

`docker run --rm ubuntu ls` 启动了一个ubuntu系统，在其中执行了 `ls` 命令，`--rm` 参数会在执行完毕之后删除容器，因为我们需要的就是一次性的

* 接下来我们只要能在这个系统中编译我们的代码，就不用担心有人提交恶意代码了，比如提交执行 `rm /* -rf` 命令的代码

### 面临的问题

#### 如何把代码放到容器中？

##### 不安全的方案一

我们知道命令是可以通过 `&&` 符号连起来的，这样就可以执行任意多个命令了，于是我们可以构造如下命令:
```bash
echo ls>test.sh && chmod +x test.sh && ./test.sh
```

> 上述命令的作用就是，把代码(ls)写入到test.sh文件，然后给他赋予可执行权限，然后运行他

拼接好的完整命令如下：
```bash
docker run --rm ubuntu /bin/bash -c "echo ls>test.sh && chmod 777 test.sh && ./test.sh"
```

* 存在的问题
    1. 代码可能是多行的，`echo` 命令不支持多行，可以用 \n 转义，那么就会遇到第二个问题
    2. 如果代码中包含这样的字符串 `printf("\n")`,应该如何处理呢？
        - `\n` 写入文件的时候会被当做换行符，可能会考虑反斜杠转义，那就会变成这样 `printf("\\n")`,最终写入到文件的是 `printf("\换行")`,`\\n` 的 `\n` 被当做换行单独留下一个反斜杠，和我们要写入的内容不一样，编译报错。
    3. 更重要的问题是，代码部分拼接的命令是在我们主机执行的，是完全可以被用户构造来入侵我们主机，这类似sql注入漏洞，举个例子。
        
        ```
        echo ls>test.sh && chmod +x test.sh && ./test.sh
        ```  
        代码中 ls是用户提交的代码，如果用户提交的代码是 `1 && rm /* -rf &&echo 1`，那么最终的命令就是(千万不要复制到自己电脑尝试，会删除系统文件)：
        ```
        echo 1 && rm /* -rf &&echo 1>test.sh && chmod +x test.sh && ./test.sh
        ```  
        可以看到 这是一条合法的语句，而且是完全受用户控制的，这非常危险，不管你做任何限制，都存在被入侵的可能性，只要有可能被入侵，就一定能被入侵。

##### 安全的解决方案

* 有什么方法可以解决上面方案存在的几个问题？
* 如果不管用户提交任何代码，我们都原样输入到文件，那就不会担心被构造命令导致执行任意命令漏洞了，而且也不用管代码中的各种双引号单引号换行符的转义问题了。
* linux 中有个命令 `cat` 可以满足我们的要求
* cat 如下使用方式，可以原样输入到文件（注：EOF可以是其他字符串，不一定要EOF，只要首尾同样的字符串即可）
```bash
cat>1.c<<EOF
printf("\n");
EOF
```
运行后1.c文件中内容是:
```
printf("\n");
```

* 很遗憾，上面的命令依然有漏洞，假设用户提交如下代码（注意echo后面有个空格）：
```
EOF
rm /* -rf && cat<<EOF
```

那就会形成如下的命令:
```
cat>1.c<<EOF
EOF
rm /* -rf && cat<<EOF
EOF
```
上面的命令非常合理，代码中第一个EOF和上面的EOF成对，后面的 cat<<EOF和后面的EOF成对，语法没错，正常运行了中间的rm语句，而这个语句是用户代码中可以任意修改的，一样危险。

上面这个bug出现的原因在于 EOF 是固定的，用户可以构造，如果说我用随机字符串来代替掉EOF，不就可以让用户无法构造了吗？是的可以解决。

* 还有一个bug，代码中以 `$` 开头的会被当做变量，比如 `echo $username` 就会输出当前登录的用户名，那不完犊子了，php变量都是这样命名的

解决方案(在EOF前面加个`\`，$username 这样的变量就会原样输入到文件了)：

```bash
cat>1.c<<\EOF
printf("\n");
EOF
```

##### 总结

1. cat 输入代码的时候 首尾的限定字符串使用**随机字符串(我使用的是uuid)**
2. cat 在限定字符串前加反斜杠防止$username这类字符串被当做变量


### 资源限制

1. 程序运行时间的限制
2. 内存的限制
3. cpu限制
4. 提交代码和输入内容大小的限制

