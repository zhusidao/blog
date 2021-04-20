输出依赖树
dependency:tree -Dverbose



**package install deploy三者区别**

- **mvn clean package依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)等７个阶段。**
- **mvn clean install依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install等8个阶段。**
- **mvn clean deploy依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install、deploy等９个阶段。**
- mvn compile下载依赖

```
<localRepository>/Users/zhusidao/Documents/.m2/repository</localRepository>
```