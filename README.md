Code

```
git clone https://github.com/iissnan/hexo-theme-next themes/next
git config --global --edit
git config --global user.name "zmquan666"
git config --global user.email "zmquan666@gmail.com"
git init
git branch -m code
git remote add origin  git@github.com:zmquan/zmquan.github.io.git
git remote remove origin
git pull origin code

npm install -g hexo-cli 
npm install hexo
npm install hexo-deployer-git -save
npm install hexo-renderer-swig --save
hexo n
hexo g
hexo d

git add .
git commit -m "code"
git push origin code

```
