---
date: 2016-12-29
layout: default

title: nginx rewrite

---

##nginx rewrite

	一．正则表达式匹配，其中：
	* ~ 为区分大小写匹配
	* ~* 为不区分大小写匹配
	* !~和!~*分别为区分大小写不匹配及不区分大小写不匹配
	
	二．文件及目录匹配，其中：
	* -f和!-f用来判断是否存在文件
	* -d和!-d用来判断是否存在目录
	* -e和!-e用来判断是否存在文件或目录
	* -x和!-x用来判断文件是否可执行
	
	三．rewrite指令的最后一项参数为flag标记，flag标记有：
	1.last    相当于apache里面的[L]标记，表示rewrite。
	2.break本条规则匹配完成后，终止匹配，不再匹配后面的规则。
	3.redirect  返回302临时重定向，浏览器地址会显示跳转后的URL地址。
	4.permanent  返回301永久重定向，浏览器地址会显示跳转后的URL地址。