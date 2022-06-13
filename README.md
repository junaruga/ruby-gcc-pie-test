# ruby-gcc-pie-test

This repository is to test an issue with the `gcc -Wl,-pie` option.

On the [ruby/ruby](https://github.com/ruby/ruby) latest master `c54f4264c281b2392a540042f894bb85c240007b`, I see the issue below.

My environment is as follows.

```
$ cat /etc/fedora-release
Fedora release 36 (Thirty Six)

$ uname -m
x86_64

$ rpm -q gcc
gcc-12.1.1-1.fc36.x86_64
```

Get the ruby/ruby source.

```
$ git clone https://github.com/ruby/ruby.git
$ cd ruby
```

The `configure` below works.

```
$ ./autogen.sh
$ ./configure --enable-shared --with-gcc="gcc -fcf-protection" LDFLAGS="-Wl,-z,now"
```

But the `configure` adding the `-Wl,-pie` below doesn't work.

```
$ ./autogen.sh
$ ./configure --enable-shared --with-gcc="gcc -fcf-protection" LDFLAGS="-Wl,-z,now -Wl,-pie"
...
checking for gcc... gcc -fcf-protection
checking whether the C compiler works... no
configure: error: in `/home/jaruga/git/ruby/ruby':
configure: error: C compiler cannot create executables
See `config.log' for more details
```

See the `config.log`.

```
$ vi config.log
...
configure:5981: gcc -fcf-protection   -Wl,-z,now -Wl,-pie conftest.c  >&5
/bin/ld: /usr/lib/gcc/x86_64-redhat-linux/12/crtbegin.o: relocation R_X86_64_32 against hidden symbol `__TMC_END__' can not be used when making a PIE object
collect2: error: ld returned 1 exit status
...
```

Then I captured the used `conftest.c` with the modified `configure` script below.

```
$ diff -u configure_org configure
--- configure_org	2022-06-10 16:58:40.451994933 +0200
+++ configure	2022-06-10 17:12:57.547974315 +0200
@@ -5978,6 +5978,8 @@
   *\"* | *\`* | *\\*) ac_try_echo=\$ac_try;;
   *) ac_try_echo=$ac_try;;
 esac
+echo "[DEBUG] ac_try_echo: $ac_try_echo"
+cat conftest.c > ~/tmp/conftest.c
 eval ac_try_echo="\"\$as_me:${as_lineno-$LINENO}: $ac_try_echo\""
 printf "%s\n" "$ac_try_echo"; } >&5
   (eval "$ac_link_default") 2>&5
```

So, here is the reproducer.

The `gcc` command below works.

```
$ gcc -fcf-protection -Wl,-z,now conftest.c

$ echo $?
0

$ ls a.out
a.out*
```

The `gcc` command below doesn't work, because the [1] says "For predictable results, you must also specify the same set of options used for compilation (-fpie, -fPIE, or model suboptions) when you specify this linker option.".

```
$ gcc -fcf-protection -Wl,-z,now -Wl,-pie conftest.c
/bin/ld: /usr/lib/gcc/x86_64-redhat-linux/12/crtbegin.o: relocation R_X86_64_32 against hidden symbol `__TMC_END__' can not be used when making a PIE object
collect2: error: ld returned 1 exit status

$ echo $?
1
```

However the `gcc` command with the `-fpie`[2] below doesn't work, because the `crtbegin.o` is not position-independent code.

```
$ gcc -fcf-protection -fpie -Wl,-z,now -Wl,-pie conftest.c
/bin/ld: /usr/lib/gcc/x86_64-redhat-linux/12/crtbegin.o: relocation R_X86_64_32 against hidden symbol `__TMC_END__' can not be used when making a PIE object
collect2: error: ld returned 1 exit status

$ echo $?
1
```

The `gcc` command below works, because with -pie, the gcc links against the `crtbeginS.o`, which is position-independent.


```
$ gcc -fcf-protection -pie -Wl,-z,now conftest.c

$ echo $?
0

$ ls a.out
a.out*
```

## References

* [1] gcc - link options - pie https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html#index-pie
* [2] gcc - options -fpie - https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html#index-fpie
