### npm命令

1. 创建``package.json`` 文件

   ```shell
   npm init     //有引导,一步步填写相关配置
   npm init --yes //跳过引导,全部设为默认值
   npm init -y //上一项的简写
   ```

   ``package.json`` 是整个项目的配置文件,保存项目名称,版本号,以及依赖项.如下

   ```json
   {
     "name": "express-demo",
     "version": "1.0.0",
     "description": "",
     "main": "index.js",
     "scripts": {
       "test": "echo \"Error: no test specified\" && exit 1"
     },
     "keywords": [],
     "author": "",
     "license": "ISC"
   }
   
   ```

2. 安装第三方库

   ```shell
   npm install express   //临时安装,不保存配置项
   npm i express //上面的简写
   npm install express --save //保存依赖项到package.json配置文件 
   npm i -S express //上面简写
   
   npm install //安装package.json中的所有依赖项
   
   ```

   

### 脚本更改自动重启服务器

1. 安装``nodemon`` 命令行工具

   ```shell
   //在任何目录下都可以执行
   npm install --global nodemon  //全局安装
   npm i -g nodemon  //简写
   ```

2. 使用``nodemon`` 命令启动服务器

   ```shell
   nodemon app.js
   ```

   只要进行``control+s`` 对文件进行保存,则服务器自动重启

### Express结合art-template模板引擎

1. 安装

   ```shell
   npm install --save art-template
   npm install --save express-art-template
   ```

2. 配置

   ```javascript
   app.engine('art', require('express-art-template'));
   ```

3. 使用

   ```javascript
   var express = require('express');
   var app = express();
   app.engine('html', require('express-art-template'));
   
   app.get('/',function(req,res){
   //express默认会到views目录下去找相应的视图文件
       res.render('index.html',{
           comments:comments
       })
   
   })
   ```

 4. 注意

 5. 

### http重定向

- 301永久重定向

  如[新浪首页](http://www.sina.com) 

- 302临时重定向

  ```javascript
   res.setHeader("Location",'/')
      res.statusCode = 302
      res.send()
  ```

  

