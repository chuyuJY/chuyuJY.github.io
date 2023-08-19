# Git-代码冲突


[toc]

## 一、基本流程

拉取最新代码——创建分支——提交分支至云端——分支代码完成后查看状态——将代码添加至暂存区——本地提交代码——推送至云端——切换为主分支——合并子分支代码——推送至云端——删除分支

## 二、执行命令

1. 在主分支拉取最新代码：`git pull`
2. 基于主分支创建并且切换到新的子分支（没有这个分支就会新建）：`git checkout -b 分支名`
3. 先把新的子分支推送到云端仓库，因为此时云端没有这个分支。如果有则忽略此步骤：`git push -u origin 分支名`
4. 然后创建对应文件，开始写代码
5. 将修改过的文件全部添加到暂存区：`git add .`
6. 将代码保存到本地仓库：`git commit -m "完成了分类功能开发"`
7. 将本地代码推送到云端：`git push`
8. 切换到主分支：`git checkout master`
9. 拉取最新代码：`git pull`
10. 如果上一步骤存在更新，则进行变基操作即处理冲突（第四步），如果没有，则进行下一步
11. 切换到主分支，合并代码：`git merge 分支名`
12. 把主分支推送到云端: `git push`
13. 删除本地的分支：`git branch -d 分支名`
14. 删除云端的分支：`git push origin --delete 分支名`

## 三、常用 `git `命令

初始化一个 `git` 仓库：`git init`

查看状态：`git status`

添加所有修改：`git add -A`

提交修改：`git commit -m "提交说明"`

查看提交历史，按字母 q(uit) 退出：`git log`

查看没 add 的不同：`git diff(erence)`

查看本地分支：`git branch`

查看所有分支：`git branch -a`

删除未合并代码分支：`git branch -D 分支名`

拉取本地不存在的远程分支：`git checkout -b 本地分支名 origin/远程分支名`

与远程仓库关联：`git remote add origin 地址`

## 四、解决冲突

pre-1: 在基准分支（主分支）上接取最新的代码 `git pull`

pre-2: 切换到自己的分支上 `git checkout` 分支名

1. `git rebase dev`
2. 解决冲突（需要和团队之间商量）
3. `git add -A`
4. `git rebase --continue`
5. 重复 2，3，4
6. 直到 `reabase` 完成,会出现 `applying` 字样
7. 有可能本地自己的分支和远程分支还有冲突，这时候需要 `git pull origin qh/home` 之后，解决冲突再 `push` 即可

## 五、`git` 的语义化 `commit`

`chore`：add Oyster build script（其他修改, 比如构建流程, 依赖管理）

`docs`：explain hat wobble（更改文档）

`feat`：add beta sequence（新增了一个功能）

`fix`：remove broken confirmation message（修复了一个 bug）

`refactor`：share logic between 4d3d3d3 and flarhgunnstow（代码重构，既不修复错误也不添加功能）

`style`：convert tabs to spaces（不影响代码含义的变化（空白、格式化、缺少分号等，注意不是 css 修改））

`test`：ensure Tayne retains clothing（测试用例修改）

## 六、`git merge` 和 `git rebase` 的区别

### `merge`

- `merge` 特点：⾃动创建⼀个新的 `commit` 如果合并的时候遇到冲突，仅需要修改后重新
- `commit` 优点：记录了真实的 `commit` 情况，包括每个分⽀的详情
- 缺点：因为每次 `merge` 会⾃动产⽣⼀个 `merge commit`，所以在使⽤⼀些 `git` 的 `GUI tools`，特别是 `commit` ⽐较频繁时，看到分支很杂乱。

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230516155809230.png" alt="image-20230516155809230" style="zoom: 33%;" />

### `rebase`

- `rebase` 特点：会合并之前的 `commit` 历史
- 优点：得到更简洁的项目历史，去掉了 `merge commit`
- 缺点：如果合并出现代码问题不容易定位，因为 `re-write` 了 `history`

<img src="https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230516155842429.png" alt="image-20230516155842429" style="zoom:33%;" />

总结：因此，当需要保留详细的合并信息的时候建议使⽤ `git merge`，特别是需要将分支合并进入 `master` 分支时；当发现自己修改某个功能时，频繁进⾏了 `git commit` 提交时，发现其实过多的提交信息没有必要时，可以尝试 `git rebase`.
