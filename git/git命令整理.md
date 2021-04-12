如何使用Git建立本地仓库并上传代码到GitHub
1. execute: git init  
2. create remote repository
3. execute: git remote add origin ${remote git url repository}





git本地版本回退

1.Mixed（默认）：它回退到某个版本，本地会保留源码，回退commit和index信息，若要提交重新commit。

2.soft: 回退到某个版本，只回退了commit的信息，不会恢复到index file一级，若要提交重新commit。

3.Hard:彻底回退到某个版本，本地的源码也会变为上一个版本的内容。

要选择想退回到的版本

