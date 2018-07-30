## conda 环境管理
创建一个名为python34的环境，指定Python版本是3.4（不用管是3.4.x，conda会为我们自动寻找3.4.x中的最新版本）
```bash
conda create --name python34 python=3.4
```
激活环境 for Windows
```bash
activate python34
```
激活环境 for Linux & Mac
```bash
source activate python34
````
退出环境 for Windows
```bash
deactivate python34
```
退出环境 for Linux & Mac
```bash
source deactivate python34
```
删除一个已有的环境
```bash
conda remove --name python34 --all
```
查看已安装的环境
```bash
conda info -e
```
## conda 包管理
conda安装列表
```bash
conda list
```
conda查询安装包
```bash
conda search django
```
安装包
```bash
conda install django
```
更新包
```bash
conda update django
```
删除包
```bash
conda remove django
```
更新conda
```bash
conda update conda
```
更新anaconda
```bash
conda update anaconda
```
更新python，假设当前环境是python 3.4, conda会将python升级为3.4.x系列的当前最新版本
```bash
conda update python  
```
添加Anaconda的国内镜像
```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
```
## anaconda 安装不存在的包
```bash
anaconda search -t conda mkdocs 
anaconda show conda-forge/mkdocs
conda install --channel https://conda.anaconda.org/conda-forge mkdocs
```
