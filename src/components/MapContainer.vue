<script setup>
import { onMounted, onUnmounted } from "vue";
import AMapLoader from "@amap/amap-jsapi-loader";
let toolBar,
  scale,
  controlBar,
  Geolocation,
  overView,
  MapType,
  ispointinring,
  polygon1,
  polygon,
  mousetool,
  dot,
  dotpath,
  selectedPolygon,
  importpolygon,
  polyEditor;
let map = null;
let drawnFeatures = [];
let drawnPolygons = [];
// var path1 = [
//   [116.475334, 39.997534],
//   [116.476627, 39.998315],
//   [116.478603, 39.99879],
//   [116.478529, 40.000296],
//   [116.475082, 40.000151],
//   [116.473421, 39.998717],
// ];
//定义函数
const createPolygon = function () {
  polyEditor.close();
  polyEditor.setTarget();
  polyEditor.open();
};
const removePolygon = function () {
  if (selectedPolygon) {
    // 在这里可以进行其他处理，例如从吸附多边形列表中移除
    console.log("Removing Polygon:", selectedPolygon.getPath());
    polyEditor.close(); // 关闭编辑器
    selectedPolygon.setMap(null); // 从地图上移除多边形
    selectedPolygon = null; // 重置选中的多边形
  } else {
    alert("请先选中一个多边形");
  }
};
const setMarker = function () {
  mousetool.marker();
  mousetool.on("draw", (event) => {
    console.log("绘制完成");
    var markerPosition = event.obj.getPosition(); // 获取绘制的点的位置
    drawnFeatures.push({
      type: "point",
      coordinates: [markerPosition.lng, markerPosition.lat],
    });
    console.log("点的位置", markerPosition);
    for (let i = 0; i < drawnPolygons.length; i++) {
      var polypath = drawnPolygons[i].getPath(); // 获取当前多边形的路径
      var isPointInside = AMap.GeometryUtil.isPointInRing(
        markerPosition,
        polypath
      );

      if (isPointInside) {
        console.log("点在多边形内");
        // 在此处执行点在多边形内的操作
        break; // 如果点在任何一个多边形内，跳出循环
      } else {
        console.log("点不在多边形内");
        // 在此处执行点不在多边形内的操作
      }
    }
  });
  // ispointinring = AMap.GeometryUtil.isPointInRing();
};
const exportData = async () => {
  const exportedData = {
    polygons: drawnFeatures
      .filter((feature) => feature.type === "polygon")
      .map((feature) => feature.coordinates),
    points: drawnFeatures
      .filter((feature) => feature.type === "point")
      .map((feature) => feature.coordinates),
  };
  const jsonData = JSON.stringify(exportedData);
  console.log(jsonData);
  console.log(drawnFeatures);
  try {
    const response = await fetch("http://localhost:3000/posts", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: jsonData,
    });

    if (!response.ok) {
      throw new Error("Network response was not ok");
    }

    // 在这里可以处理后端返回的数据
    const responseData = await response.json();
    console.log("Received data from the backend:", responseData);
  } catch (error) {
    console.error("Error exporting data:", error.message);
  }
  // TODO: 在这里可以将 jsonData 发送给后端，使用 fetch 或其他方式
};
const importData = () => {
  // 发送 GET 请求到后端获取数据
  fetch("http://localhost:3000/posts")
    .then((response) => {
      if (!response.ok) {
        throw new Error("Network response was not ok");
      }
      return response.json(); // 将响应解析为 JSON
    })
    .then((data) => {
      console.log("Received imported data from backend:", data);

      // 在这里处理从后端接收到的导入数据
      data.forEach((item) => {
        // 显示多边形
        if (item.polygons && item.polygons.length > 0) {
          item.polygons.forEach((polygonCoordinates) => {
            (function (polygonCoords) {
              // 使用立即执行函数表达式创建新的作用域
              const importpolygon = new AMap.Polygon({
                path: polygonCoords,
                // You can customize polygon options here, such as fillColor, strokeColor, etc.
              });
              polyEditor.addAdsorbPolygons(importpolygon);
              map.add(importpolygon);
              drawnPolygons.push(importpolygon);
              importpolygon.on("dblclick", () => {
                polyEditor.setTarget(importpolygon);
                selectedPolygon = importpolygon;
                polyEditor.open();
              });
            })(polygonCoordinates); // 将多边形坐标传递给立即执行函数
          });
        }
        // 显示点
        if (item.points && item.points.length > 0) {
          item.points.forEach((pointCoordinates) => {
            const marker = new AMap.Marker({
              position: new AMap.LngLat(
                pointCoordinates[0],
                pointCoordinates[1]
              ),
              // 在此处可以自定义标记选项，如icon、title等
            });
            map.add(marker);
            drawnFeatures.push({
              type: "point",
              coordinates: pointCoordinates,
            });
          });
        }
      });
      // 存储后端返回的数据
    })
    .catch((error) => {
      console.error("Error fetching imported data:", error.message);
      // 处理发生的错误
    });
};
//生命周期
onMounted(() => {
  AMapLoader.load({
    key: "8e8a107e8ac39b7d118741211cdb2b98", // 申请好的Web端开发者Key，首次调用 load 时必填
    version: "2.0", // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
    plugins: ["AMap.Scale"], // 需要使用的的插件列表，如比例尺'AMap.Scale'等
  })
    .then((AMap) => {
      map = new AMap.Map("container", {
        // 设置地图容器id
        viewMode: "3D", // 是否为3D地图模式
        zoom: 11, // 初始化地图级别
        center: [116.397428, 39.90923], // 初始化地图中心点位置
      });

      AMap.plugin([
        "AMap.ToolBar",
        "AMap.Scale",
        "AMap.ControlBar",
        "AMap.Geolocation",
        "AMap.HawkEye",
        "AMap.MapType",
        "AMap.PolygonEditor",
        "AMap.Polygon",
        "AMap.MouseTool",
      ]);
      toolBar = new AMap.ToolBar({
        position: "LB",
        offset: [30, 60],
      });
      mousetool = new AMap.MouseTool(map);
      scale = new AMap.Scale();
      controlBar = new AMap.ControlBar({
        position: "RT",
      });
      Geolocation = new AMap.Geolocation({
        position: {
          top: "180px",
          right: "30px",
        },
      });
      overView = new AMap.HawkEye({
        position: "LT",
      });
      // polygon1 = new AMap.Polygon({
      //   path: path1,
      // });
      map.addControl(scale);
      map.addControl(toolBar);
      map.addControl(controlBar);
      map.addControl(Geolocation);
      map.addControl(overView);
      scale.hide();
      toolBar.hide();
      controlBar.hide();
      overView.hide();
      // map.add(polygon1);
      map.setFitView(polygon1);

      // ----------------------------------//

      polyEditor = new AMap.PolygonEditor(map);
      // polyEditor.addAdsorbPolygons(polygon1);
      // 把自己定义的多边形加入到编辑器里
      polyEditor.on("add", function (data) {
        console.log(data);
        polygon = data.target;
        drawnPolygons.push(polygon); //多边形集合
        console.log("drawnPolygons:", drawnPolygons);
        drawnFeatures.push({
          type: "polygon",
          coordinates: polygon.getPath().map((point) => [point.lng, point.lat]),
        });
        //多边形导入。
        var polypath = polygon.getPath();
        console.log("New Polygon:", polygon.getPath());
        polyEditor.addAdsorbPolygons(polygon);
        polygon.on("dblclick", (event) => {
          var adjustedPolygon = event.target;
          selectedPolygon = adjustedPolygon; // 保存选中的多边形
          console.log("Selected Polygon:", selectedPolygon.getPath());
          polyEditor.setTarget(polygon);
          polyEditor.open();
        });
      });
      // polygon1.on("dblclick", () => {
      //   polyEditor.setTarget(polygon1);
      //   polyEditor.open();
      // });
      importData();
      map.on("rightclick", () => {
        mousetool.close();
        console.log("close is ok");
      });
      const marker = new AMap.Marker({
        position: new AMap.LngLat(116.39, 39.9),
        offset: new AMap.Pixel(-10, -10),
        icon: "//vdata.amap.com/icons/b18/1/2.png", // 添加 Icon 图标 URL
        title: "北京",
      });

      map.add(marker);
    })

    .catch((e) => {
      console.log(e);
    });
});
onUnmounted(() => {
  map?.destroy();
});
</script>
<template>
  <div id="container"></div>
  <div
    class="flex flex-col min-w-0 break-words bg-white bg-clip-border rounded-md shadow-md bottom-4 right-4 flex-1 p-3 fixed"
    style="width: 120px"
  >
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="setMarker"
      style="margin-bottom: 5px"
    >
      标记点位
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="createPolygon"
      style="margin-bottom: 5px"
    >
      新建多边形
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="polyEditor.open"
      style="margin-bottom: 5px"
    >
      多边形编辑
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="polyEditor.close"
    >
      结束编辑
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-6 00"
      @click="removePolygon"
    >
      删除多边形
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="exportData"
      style="margin-bottom: 5px"
    >
      导出数据
    </button>
    <button
      class="inline-block font-normal text-center whitespace-nowrap align-middle select-none border border-transparent bg-none text-blue-500 py-1 px-2 leading-[1.5] rounded-full appearance-none border-blue-700"
      @click="importData"
      style="margin-bottom: 5px"
    >
      导入数据
    </button>
  </div>
  <!-- <div class="input-card">
    <div class="input-item">
      <input type="checkbox" onclick="toggleScale(this)" />比例尺
    </div>

    <div class="input-item">
      <input type="checkbox" id="toolbar" onclick="toggleToolBar(this)" />工具条
    </div>

    <div class="input-item">
      <input
        type="checkbox"
        id="controlBar"
        onclick="toggleControlBar(this)"
      />工具条方向盘
    </div>

    <div class="input-item">
      <input
        type="checkbox"
        id="overview"
        onclick="toggleOverViewShow(this)"
      />显示鹰眼
    </div>
  </div> -->
</template>

<style scoped>
#container {
  width: 100%;
  height: 100vh;
}
#myPageTop {
  position: absolute;
  top: 5px;
  right: 10px;
  background: #fff none repeat scroll 0 0;
  border: 1px solid #ccc;
  margin: 10px auto;
  padding: 6px;
  font-family: "Microsoft Yahei", "微软雅黑", "Pinghei";
  font-size: 14px;
}
#myPageTop label {
  margin: 0 20px 0 0;
  color: #666666;
  font-weight: normal;
}
#myPageTop input {
  width: 170px;
}
#myPageTop .column2 {
  padding-left: 25px;
}
</style>
