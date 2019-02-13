---
layout: post
title: "使用hexo搭建个人博客，实现不同电脑协作"
date: 2019-02-13 18:12:22
tags:
	- hexo
---

假设现在有两台电脑：电脑A、电脑B
###在电脑A上新建blog并进行如下操作：
1.在github上新建个人博客仓库
	新建hexo分支并设置为默认分支，clone到本地电脑
2.hexo init blog命令执行及生成文件
![命令执行](/assets/blogImg/hexo_blog/hexo1.png)
![生成文件](/assets/blogImg/hexo_blog/hexo2.png)
3.npm install 命令执行及生成文件
![命令执行](/assets/blogImg/hexo_blog/hexo3.png)
![生成文件](/assets/blogImg/hexo_blog/hexo4.png)
4.复制第2、3步执行命令生成的文件到个人博客仓库中
![复制后git仓库文件目录](hexo5.png)
5.hexo generate 命令生成html页面
![命令执行](/assets/blogImg/hexo_blog/hexo6.png)
![生成文件](/assets/blogImg/hexo_blog/hexo7.png)
6.在hexo分支上把更新提交到远端
git add .
git commit -m "hexo源文件提交"
git push origin
![本地文件](/assets/blogImg/hexo_blog/git1.png)
![gitignore](/assets/blogImg/hexo_blog/git2.png)
![github提交文件](/assets/blogImg/hexo_blog/git3.png)
7.修改_config.yml配置，使用hexo deploy命令发布blog
	执行npm install hexo-deployer-git --save
	执行npm i hexo-generator-json-content --save
![命令执行](/assets/blogImg/hexo_blog/hexo8.png)
![生成文件](/assets/blogImg/hexo_blog/hexo9.png)
	修改_config.yml配置
![修改](/assets/blogImg/hexo_blog/hexo10.png)
	执行hexo deploy命令，把blogHTML文件发布到github的master分支
![修改](/assets/blogImg/hexo_blog/hexo11.png)
	输入生成ssh时设置的密码
![修改](/assets/blogImg/hexo_blog/hexo12.png)
	执行命令后仓库目录结构
![生成文件](/assets/blogImg/hexo_blog/hexo13.png)
8.预览效果
![预览效果](/assets/blogImg/hexo_blog/pre1.png)
9.本地预览效果
	执行 hexo server
![命令执行](/assets/blogImg/hexo_blog/pre2.png)
![预览效果](/assets/blogImg/hexo_blog/pre3.png)
10.在hexo分支提交修改到远端
	git commit -a -m "使用默认模板的示例blog源码提交"
	git push origin hexo
![命令执行](/assets/blogImg/hexo_blog/git4.png)
![本地分支目录](/assets/blogImg/hexo_blog/hexo13.png)
![github分支目录](/assets/blogImg/hexo_blog/git5.png)

###在电脑B上给blog换一个主题yilia
1.在本地clone该blog仓库
![clone仓库](/assets/blogImg/hexo_blog/team1.png)
![文件目录](/assets/blogImg/hexo_blog/team2.png)
2.在该目录下执行命令生成相关依赖
	执行npm install
	执行npm install hexo-deployer-git --save
	执行npm i hexo-generator-json-content --save
![文件目录](/assets/blogImg/hexo_blog/team3.png)
3.执行生成预览命令
	hexo g
	hexo s
![命令执行](/assets/blogImg/hexo_blog/team4.png)
![文件目录](/assets/blogImg/hexo_blog/team5.png)
![预览](team6.png)
4.换主题为yilia
	执行hexo clean
	clone yilia主题放在themes目录
	删除yilia目录下的.git目录
![命令执行](/assets/blogImg/hexo_blog/team7.png)
![文件目录](/assets/blogImg/hexo_blog/team8.png)
	修改_config.yml文件中的主题配置
![修改](/assets/blogImg/hexo_blog/team9.png)

5.生成预览
	执行hexo d -g
	执行hexo s -g
	![执行](/assets/blogImg/hexo_blog/team10.png)
	![本地预览](/assets/blogImg/hexo_blog/team11.png)
	![blog预览](/assets/blogImg/hexo_blog/team12.png)
6.在hexo分支提交修改到远端
	git add .
	git commit -a -m "使用yilia主题的示例blog源码提交"
	git push origin hexo
	![github分支目录](/assets/blogImg/hexo_blog/git6.png)

###回到电脑A上自定义主题yilia
1.把github远端hexo分支中的修改pull到本地
	git pull
2.修改yilia相关配置
![修改](/assets/blogImg/hexo_blog/team13.png)
3.生成预览
	执行hexo s -g
	执行hexo d -g
	![本地预览](/assets/blogImg/hexo_blog/team14.png)
	![blog预览](/assets/blogImg/hexo_blog/team15.png)
4.在hexo分支提交修改到远端
	git add .
	git commit -a -m "自定义yilia主题修改提交"
	git push origin hexo
![github分支目录](/assets/blogImg/hexo_blog/git15.png)

#参考文档
[手把手教你使用Hexo + Github Pages搭建个人独立博客](https://segmentfault.com/a/1190000004947261)
