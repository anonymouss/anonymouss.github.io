#### basic configuration

- install hexo: `npm install hexo -g`
- init project: `hexo init`
- intall dependencies: `npm install`
- config deployment: modify `_config.yml`
```yml
deplpy:
    type: git
    repository: git_url
    branch: master
```
- install deplpy ext: `npm install hexo-deployer-git --save`
- deploy: `hexo d -g`
