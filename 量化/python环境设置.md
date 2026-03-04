```
启动Qbot项目：
# 1) 安装 3.10（如果你还没有）
brew install pyenv
pyenv install 3.8.19

# 在当前 shell 中创建别名，临时别名
alias python310='~/.pyenv/versions/3.10.14/bin/python3'
alias python308='~/.pyenv/versions/3.8.19/bin/python3'

# 2) 重新建虚拟环境
rm -rf venv
python308 -m venv venv
source venv/bin/activate

# 3) 安装依赖
pip install -i https://mirrors.aliyun.com/pypi/simple/ -r requirements.txt
```