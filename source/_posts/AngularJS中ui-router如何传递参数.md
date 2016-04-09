title: AngularJS中ui-router如何传递参数
date: 2016-04-09 23:36:18
categories: 前端
tags: AngularJS
---
ui-router是AngualrJS中常用的路由框架。其中ui-sref 一般使用在 a标签中，\$state.go('someState')一般使用在controller里面。这两个本质上是一样的东西，查看ui-sref的源码，ui-sref最后调用的还是$state.go()方法。

### 如何传递参数
首先定义路由状态：
```js
.state('monitorInstance', {
   url: "/monitorInstance/:instanceId/:templateId",
   views: {
    "appContent" :{
       templateUrl: baseUrl +'/tpl/monitorInstance.html',
       controller: MonitorInstanceCtrl
    }
 }
})
```

ui-sref传参：
```js
<a ui-sref="monitorInstance({instanceId: {{instance.id}},  templateId: {{instance.template.id}} })">
    <span>{{instance.name}}</span>
</a>

```
$state.go传参：
```js
$state.go('monitorInstance', {instanceId: instance.id, templateId:instance.template.id});
```
接收参数：
```js
function listInstanceCtrl($scope,$stateParams,listInstanceServices) {
    console.log($stateParams.instanceId);
    console.log($stateParams.templateId);
    }
```