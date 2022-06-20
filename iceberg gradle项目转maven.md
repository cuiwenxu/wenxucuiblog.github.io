@[TOC](iceberg gradle项目转maven)

iceberg github上源码是用gradle做依赖管理的，下面记录踩的一些坑：
# 通过versions.props集中进行版本管理
其各dependency的version是集中在versions.props文件中进行管理的，在build.gradle通过dependencyRecommendations来指定
```json
dependencyRecommendations {
  propertiesFile file: file('versions.props')
}
```
