+++
title = "基于 Vue 和 OpenLayers 地图的轨迹绘制功能"
categories = ["project"]
tags = ["Vue", "Openlayers"]
date = 2019-03-22
+++

## 前言

近期工作中主要都在做 `gais` 地图的相关业务，本文会对项目中遇到的一些问题和挑战做一次梳理和总结，方便同事们在遇到类似业务场景时有所参考，快速上手

## 准备工作

![巡检轨迹页面](https://upload-images.jianshu.io/upload_images/4260383-e314ab512a3d2795.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 需求分析

1. 根据查询结果返回的巡检数据，对经过的监控点进行**轨迹绘制**
1. 监控点默认状态为**未经过**，当查询结束后，根据查询结果更新矢量图层中的点标记，将状态更新为**经过**或**未经过**
1. 用户可以手动调整监控点是否经过，修改后监控点状态和轨迹**同步更新**

### 根据业务封装的地图方法

- `generateOrbit` 轨迹绘制
- `clearOrbit` 清除轨迹
- `updateStatus` 更新点标记状态

## 实现

### 轨迹绘制

#### 1. 获取数据

输入查询条件，点击查询请求接口，获取到所有监控点的数据记录 `trailData`
![监控点数据](https://upload-images.jianshu.io/upload_images/4260383-e7d22ce194673da6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 处理数据

当返回的`trailData`形如`[经过，经过，未经过，经过，经过]`时，轨迹需要分段显示，此处参考 split 的实现原理，定义`chunkTrails` 方法并结合`Vue`的计算属性`computed`，计算出轨迹数据 `orbitData`

```javascript
  computed: {
    orbitData() {
      if (this.trailData && this.trailData.length) {
        return this.chunkTrails(this.trailData)
          .map(coordinates => {
            return {
              peopleId: '' //业务需要，与其他地方保持一致
              coordinates
            };
          })
          .filter(val => val.coordinates.length);
      } else {
        return [];
      }
    }
    ......
  }
  /**
   * 根据巡检点状态进行轨迹切分，1 经过 2 未经过
   *
   * @params {object[]} arr 后端返回的巡检点对象数组
   * return {number[][]} 分段轨迹
   */
  chunkTrails(arr = []) {
    let prev = 0;
    let len = arr.length;
    // 提取坐标
    let getCoordinate = (coordinateArr, end) =>
      coordinateArr.push(
        arr.slice(prev, end).map(({ lon: x, lat: y } = {}) => {
          return { x, y };
        })
      );
    let res = arr.reduce((acc, val, i) => {
      if (val.type !== 1) {
        if (prev !== i) {
          getCoordinate(acc, i);
        }
        prev = i + 1;
      }
      return acc;
    }, []);

    if (prev <= len) {
      getCoordinate(res, len);
    }
    return res;
  },
```

![分段轨迹轨迹](https://upload-images.jianshu.io/upload_images/4260383-9d6063f960722055.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3. 绘制轨迹

通过`Vue`侦听器`watch`响应数据变化，调用`generateOrbit`方法进行轨迹绘制

```javascript
  orbitData: {
    handler(val, oldVal) {
      if (val) {
        this.clearTrail();
        for (const orbit of val) {
          // 轨迹段有两个点以上才绘制轨迹
          if (orbit && orbit.coordinates && orbit.coordinates.length > 1) {
            this.appMap.generateOrbit(orbit);
          }
        }
      }
    },
    deep: true
  },
}
```

`generateOrbit`方法内部会`new`一个`ol.geom.LineString`的实例`lineString`，如果坐标数组长度大于 1，就通过`appendCoordinate`方法遍历新增坐标，否则就重新`new`为`ol.geom.Point`的实例，画一个点。由于此处不需要画点，在调用方法前就对数据进行预过滤，减少了函数调用次数，提高了执行效率，也避免了直接对`gais`源码进行修改

```
  // generateOrbit核心实现
  function (param) {
    ……
    var lineString = new ol.geom.LineString();
    for (var i = 0; i < coordinates.length; i++) {
      var item = coordinates[i];
      if (item && item.x && item.y) {
        var point = [item.x, item.y];
        if (coordinates.length == 1) {
          lineString = new ol.geom.Point(point);
        } else {
          lineString.appendCoordinate(point)
        }
      }
    }
    ……
```

### 监控点状态更新

**获取点标记的 id 映射表**，地图矢量图层加载完成后，会抛出一个名为 `load-vector-success` 的事件，此时矢量数据加载完成，通过监听该事件可以拿到矢量图层的全部信息。通过`Vue`的组件通信传递信息，最终得到一个巡检点的 Map 映射集合`trailPoint`

- map 组件

```javascript
  let event = this.appMap.on('vector-load-feature-success', event => {
    // 过滤巡检监控点
      if (this.isInspect && event && event.param && event.param.code === 'spms_static_camera') {
        event.param.features = event.param.features.filter(val => val.N.use === 'trail')
        this.$emit('inspect', event.param.features)
      }
    ...
  }
```

- 父组件

```javascript
  handleInspect(features) {
      if (features) {
        this.trailPoint = new Map(features.map(val => [val.N.id, val.N.fid]));
      }
  }
```

2. **计算出符合状态更新方法的数据。**当调用地图的状态更新方法时，需要根据`featureId`按图索骥找到对应的点标记，进而更新矢量图层。查询结果`trailData`中的`elementId`和`trailPoint`的`id`是同一个值，而`id`与`featureId`又存在映射关系，不难得出`statefulPoint`

```javascript
  computed： {
    statefulPoint() {
      const len = this.trailData.length;
      if (len) {
        // 根据当前操作类型映射，1 添加 2 替换 3 清除
        operate = operateMap.get(this.currentOperate)
        let obj = {
          layerCode: this.layerCode,  //图层编码
          layerWorkspace: this.layerWorkspace,  // 图层所在工作空间编码
          operate
        };

        return this.trailData.map(v => {
          const featureId = this.trailPoint.get(v.elementId); // 获取点标记的 featureId
          const status = v.type
          return {
            ...obj,
            featureId,
            status
          };
        });
      } else {
        return [];
      }
    },
    ......
  }
```

3. **更新监控点状态**，当监控点状态数据发生变更时，执行`updateStatus`方法，更新监控点状态

```javascript
  statefulPoint: {
      handler(val, oldVal) {
        if (val && val.length) {
          this.appMap.updateStatus(val);
        }
      },
      deep: true
  }
```

`updateStatus`方法传入的五个参数中，`featureId`、`layerCode`、`layerWorkspace`用于定位到要素，`status`对应 gis 引擎服务中对应的图标样式、`operate`会涉及到五种状态变更（对应传入的 operate 参数），分别为**添加（add）**、**追加（append）**、**替换（replace）**、**删除（delete）**、**清除（clear）**。由于此处的监控点一开始都是无状态的，所以在这里进行**添加（add）**

### 用户修改监控点状态

点击地图上的监控点后，右侧会出现当前监控点的详情页面，当用户手动更改监控点的经过状态时，只需要更新`trailData`中对应监控点的`type`类型，`orbitData` 和`statefulPoint`就会随之更新，进而触发侦听器，重新绘制轨迹和更新监控点状态。此时监控点已有状态，会进行**替换（replace）**

```javascript
  setStatus() {
    let _loading = this._getLoading();
    const params = {
      cameraIndexCode: this.point.cameraIndexCode,
      type: this.pointType === 1 ? 2 : 1,
      reviseTime: this.formatDate(this.form.timeScope).slice(0, 10)
    };
    Service.amendTrajectory(params)
      .then(res => {
        this._closeLoading(_loading);
        const { result, data } = res;
        if (result) {
          this.point.modifyPerson = data.personName;
          const index = this.trailData.findIndex(
            item => item.cameraIndexCode === this.point.cameraIndexCode
          );
          this.trailData[index].type = params.type;
          this.point.status = params.type;
          this.$message({
            type: 'success',
            message: '设置成功！'
          });
        }
      })
      .catch(err => this.$message.error(err.resultMsg));
  },
```

Tips: 由于用户可能进行多次不同的查询操作，因此每次查询时要先清除之前的轨迹和监控点状态，再进行之前的操作：

1. 清除监控点状态依然使用`updateStatus`。对状态进行**清除（clear）**
1. 清除轨迹的实现方法如下：

```javascript
MapConfig.prototype.clearOrbit = function() {
  if (this.orbitLayer) {
    this.orbitLayer.getSource().clear();
  }
};
```

## 最终效果

### 第一次查询有结果

![未全部经过](https://upload-images.jianshu.io/upload_images/4260383-82a7cb175be543ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 用户手动设置监控点状态

![点击监控点显示详情](https://upload-images.jianshu.io/upload_images/4260383-c0fc94eb3f2c11e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 状态修改完成

![全部经过](https://upload-images.jianshu.io/upload_images/4260383-fdd0c78aa4638feb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 完整流程

![完整流程.png](https://upload-images.jianshu.io/upload_images/4260383-9f3a208f095ee650.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考资料

- [https://vuejs.org/index.html](https://vuejs.org/index.html)
- [https://openlayers.org/](https://openlayers.org/)
- [https://developer.mozilla.org](https://developer.mozilla.org/)
