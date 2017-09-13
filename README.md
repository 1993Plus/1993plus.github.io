This is http://blog.at23.net/ source files.

* 日常修改  
	在本地对博客进行修改（添加新博文、修改样式等等）后，通过以下流程进行管理:  

	* `git add .` or `git add -A`  
	* `git commit -m "..."`  
	* `git push origin hexo` ## 将新改动推送到 GitHub 上 hexo  分支  
	* `hexo g -d` ## 发布到 master 分支上  

* 本地数据丢失
	本地数据被误删或更换电脑修改博客：  
	
	* 使用 `git clone git@github.com:1993Plus/1993plus.github.io.git` 克隆默认分支 hexo 到本地
	* 在目录中执行 `npm install hexo 、npm install 、npm install hexo-deployer-git`
