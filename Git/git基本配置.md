
## 基础配置

Git 自带一个`git config`的文件来进行配置。在三个不同的地方可以访问该文件：

- `/etc/gitconfig`：系统级别的配置，影响系统里每一个用户和他们git仓库的配置。如果使用--system选项的`git config`时，就会从此读写配置；

- `~/.gitconfig`或`~/.config/git/config`: 只针对当前用户。可以传递`--global`选项让git读写此配置。

- 当前项目仓库的Git目录中的config文件（`.git/config`）：针对该仓库。

每一个级别覆盖上一级别的配置，所以 `.git/config` 的配置变量会覆盖 `/etc/gitconfig` 中的配置变量。

比如，设置项目的git账户信息：

项目级别的配置：

```
cd ./project 
git config user.name "John Doe"
git config user.email johndoe@example.com

```

用户级别的配置：
```
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com

```
系统级别的配置：
```
git config --system user.name "John Doe"
git config --system user.email johndoe@example.com

```

## 检查配置

- 检查所有配置：

`git config --list`

- 检查具体某个配置：

`git config <key>` 比如：
`$ git  config user.name`


