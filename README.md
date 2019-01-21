#### basic configuration

- install hexo: `npm install hexo -g`
- init project: `hexo init`
- intall dependencies: `npm install`
- config deployment: modify `_config.yml`
```yml
deploy:
    type: git
    repository: git_url
    branch: master
```
- install deploy ext: `npm install hexo-deployer-git --save`
- deploy: `hexo d -g`

#### theme

Next: `git clone https://github.com/theme-next/hexo-theme-next themes/next`

#### references

- Hexo: https://hexo.io/zh-cn/docs/configuration
- Next: http://theme-next.iissnan.com/getting-started.html

#### notes

> themes/next 's .git folder is removed

- config files in branch `hexo-config`
- blog static files in branch `master`
 