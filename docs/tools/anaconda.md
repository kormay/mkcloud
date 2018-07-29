## conda 环境管理
```bash
# 创建一个名为python34的环境，指定Python版本是3.4（不用管是3.4.x，conda会为我们自动寻找3.4.x中的最新版本）
conda create --name python34 python=3.4
# 激活环境 for Windows
activate python34
# 激活环境 for Linux & Mac
source activate python34
# 退出环境 for Windows
deactivate python34
# 退出环境 for Linux & Mac
source deactivate python34
# 删除一个已有的环境
conda remove --name python34 --all
# 查看已安装的环境
conda info -e
```
## conda 包管理
```bash
conda list
conda search django
conda install django
conda update django
conda remove django

conda update conda
conda update anaconda
conda update python  # 假设当前环境是python 3.4, conda会将python升级为3.4.x系列的当前最新版本
# 添加Anaconda的TUNA镜像
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
```
## anaconda 安装不存在的包
```bash
anaconda search -t conda mkdocs 
anaconda show conda-forge/mkdocs
conda install --channel https://conda.anaconda.org/conda-forge mkdocs
```
