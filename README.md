1、依赖
======
JDK 1.8
配置好JAVA_HOME环境变量
安装好maven
brew install maven

2、地址
======
https://beam.apache.org/get-started/quickstart-java/

3、开始
======
新建目录
mkdir BeamHelloWorld
cd BeamHelloWorld

mvn archetype:generate \
      -DarchetypeGroupId=org.apache.beam \
      -DarchetypeArtifactId=beam-sdks-java-maven-archetypes-examples \
      -DarchetypeVersion=2.0.0 \
      -DgroupId=org.example \
      -DartifactId=word-count-beam \
      -Dversion="0.1" \
      -Dpackage=org.apache.beam.examples \
      -DinteractiveMode=false


[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building Maven Stub Project (No POM) 1
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] >>> maven-archetype-plugin:2.4:generate (default-cli) > generate-sources @ standalone-pom >>>
[INFO]
[INFO] <<< maven-archetype-plugin:2.4:generate (default-cli) < generate-sources @ standalone-pom <<<
[INFO]
[INFO]
[INFO] --- maven-archetype-plugin:2.4:generate (default-cli) @ standalone-pom ---
[INFO] Generating project in Batch mode
[INFO] Archetype repository not defined. Using the one from [org.apache.beam:beam-sdks-java-maven-archetypes-examples:0.6.0] found in catalog remote
Downloading: https://repo.maven.apache.org/maven2/org/apache/beam/beam-sdks-java-maven-archetypes-examples/2.0.0/beam-sdks-java-maven-archetypes-examples-2.0.0.jar
Downloaded: https://repo.maven.apache.org/maven2/org/apache/beam/beam-sdks-java-maven-archetypes-examples/2.0.0/beam-sdks-java-maven-archetypes-examples-2.0.0.jar (2.8 MB at 907 kB/s)
Downloading: https://repo.maven.apache.org/maven2/org/apache/beam/beam-sdks-java-maven-archetypes-examples/2.0.0/beam-sdks-java-maven-archetypes-examples-2.0.0.pom
Downloaded: https://repo.maven.apache.org/maven2/org/apache/beam/beam-sdks-java-maven-archetypes-examples/2.0.0/beam-sdks-java-maven-archetypes-examples-2.0.0.pom (4.0 kB at 9.6 kB/s)
[INFO] ----------------------------------------------------------------------------
[INFO] Using following parameters for creating project from Archetype: beam-sdks-java-maven-archetypes-examples:2.0.0
[INFO] ----------------------------------------------------------------------------
[INFO] Parameter: groupId, Value: org.example
[INFO] Parameter: artifactId, Value: word-count-beam
[INFO] Parameter: version, Value: 0.1
[INFO] Parameter: package, Value: org.apache.beam.examples
[INFO] Parameter: packageInPathFormat, Value: org/apache/beam/examples
[INFO] Parameter: package, Value: org.apache.beam.examples
[INFO] Parameter: version, Value: 0.1
[INFO] Parameter: groupId, Value: org.example
[INFO] Parameter: targetPlatform, Value: 1.7
[INFO] Parameter: artifactId, Value: word-count-beam
[INFO] project created from Archetype in dir: /Users/lemonhall/Downloads/Beam_HelloWorld/word-count-beam
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 8.842 s
[INFO] Finished at: 2017-05-19T13:17:24+08:00
[INFO] Final Memory: 18M/208M
[INFO] ------------------------------------------------------------------------
lemonhall@HalldeMacBook-Pro:~/Downloads/Beam_HelloWorld$



生成如下目录树结构：

（图1）

4、WordCount的解释
=================
https://beam.apache.org/get-started/wordcount-example/


5、执行
======
Beam一共有数个执行器
ApexRunner, FlinkRunner, SparkRunner, DataflowRunner.

而DirectRunner是供本机测试用的

使用--runner=<runner>来指定执行器


开跑：

mvn compile exec:java -Dexec.mainClass=org.apache.beam.examples.WordCount \
     -Dexec.args="--inputFile=pom.xml --output=counts" -Pdirect-runner

一阵疯狂的下载之后：

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 29.656 s
[INFO] Finished at: 2017-05-19T13:30:29+08:00
[INFO] Final Memory: 33M/370M
[INFO] ------------------------------------------------------------------------

6、验证
======
lemonhall@HalldeMacBook-Pro:~/Downloads/Beam_HelloWorld/word-count-beam$ ls counts*
counts-00000-of-00003 counts-00001-of-00003 counts-00002-of-00003
lemonhall@HalldeMacBook-Pro:~/Downloads/Beam_HelloWorld/word-count-beam$


7、看文件内部
===========

lemonhall@HalldeMacBook-Pro:~/Downloads/Beam_HelloWorld/word-count-beam$ cat counts-00000-of-00003
module: 3
version: 79
be: 1
example: 2
Additionally: 1
mojo: 1
listing: 1
agreed: 1
implied: 1
jdk: 5
schemaLocation: 1
the: 25
Spark: 1
to: 9
xml: 1
plugin: 22
groupId: 70
codehaus: 1
repositories: 3
This: 1
Dependencies: 1
rev: 2
LICENSE: 2
agreements: 1
regarding: 1
encoding: 1
count: 1
activation: 2
threadCount: 2



8、接下来看具体的程序
==================

MinimalWordCount.java

工厂模式先建立一个Pipeline的参数对象
```
PipelineOptions options = PipelineOptionsFactory.create();
```

新建一个Pipeline
```
Pipeline p = Pipeline.create(options);
```
读取input
```
p.apply(TextIO.read().from("gs://apache-beam-samples/shakespeare/*"))
```
莎士比亚

然后开搞：
```
.apply("ExtractWords", ParDo.of(new DoFn<String, String>() {
    @ProcessElement
    public void processElement(ProcessContext c) {
        // \p{L} denotes the category of Unicode letters,
        // so this pattern will match on everything that is not a letter.
        for (String word : c.element().split("[^\\p{L}]+")) {
            if (!word.isEmpty()) {
                c.output(word);
            }
        }
    }
}))
```
"[^\\p{L}]+"这个只是一个技巧性的东西，将任何非letter的东西都视为可输出就是了；

计数：
```
.apply(Count.<String>perElement())
```
这个Count是内建transform，接受任何类型的PCollection输入，并输出一个键值对的PCollection

格式化：
```
     // Apply a MapElements transform that formats our PCollection of word counts into a printable
     // string, suitable for writing to an output file.
     .apply("FormatResults", MapElements.via(new SimpleFunction<KV<String, Long>, String>() {
                       @Override
                       public String apply(KV<String, Long> input) {
                         return input.getKey() + ": " + input.getValue();
                       }
                     }))
```
这一段就比较恶心了，比较复杂；
不加这一段估计有问题，因为是并列执行的，所以最后需要做最后一道reduce吧，应该是，我待会试试


输出：
```
.apply(TextIO.write().to("wordcounts"));
```


9、总结
======
https://beam.apache.org/get-started/wordcount-example/

暂时来看，还比较简单，例子跑起来也很顺利

但是，关键还是要知晓流式处理里的很多关键概念和坑；

这个放在Part2来讲看吧





