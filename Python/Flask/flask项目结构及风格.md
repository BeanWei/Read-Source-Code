# Flask项目结构及风格

>[参考 miguelgrinberg 的微博客项目](https://github.com/miguelgrinberg/microblog) 

## 项目结构
```
-- microblog                (项目文件)
   |
   -- app                   (app项目文件)
      |
      -- api
         |
         -- __init__.py
         -- auth.py
         -- errors.py
         -- tokens.py
         -- users.py      
      -- auth
         |
         -- __init__.py
         -- email.py
         -- forms.py
         -- routes.py
      -- errors
         |
         -- __init__.py
         -- handlers.py
      -- main
         |
         -- __init__.py
         -- forms.py
         -- routes.py
      -- static
      -- templates
      -- translations/es/LC_MESSAGES
      -- __init__.py
      -- cli.py
      -- email.py
      -- models.py
      -- search.py
      -- tasks.py
      -- translate.py

   -- deployment
   -- migrations
   -- .gitignore
   -- Dockerfile
   -- LICENSE
   -- Procfile
   -- README.md
   -- Vagrantfile
   -- babel.cfg
   -- boot.sh               (app启动脚本)
   -- config.py             (配置文件)
   -- microblog.py          (app启动入口)
   -- requirements.txt      (项目依赖)
   -- tests.py              (测试)
```