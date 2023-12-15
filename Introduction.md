# Kelinci fuzzing usage introdution


```
java -cp bin-instrumented:/home/asiv/Project/fuzzing/casetest/src/pro/target/lib/* org.apache.sling.Main in_dir/seed.txt

zsh: no matches found: bin-instrumented:/home/asiv/Project/fuzzing/casetest/src/pro/target/lib/*
```

由于我用的终端程序是`zsh`,`zsh`会尝试扩展通配符，但命令中使用了通配符`*`因此,`zsh`无法找到匹配的文件。


所以这里`classpath`或者`cp`后面的库需要用`""`

连接
```
java -cp "bin-instrumented:/home/asiv/Project/fuzzing/casetest/src/pro/target/lib/*" edu.cmu.sv.kelinci.Kelinci org.apache.sling.Main @@
```


```
afl-fuzz -i in_dir -o out_dir /home/asiv/Project/kelinci/fuzzerside/interface @@
```
