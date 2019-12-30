# 使用cesium加载离线地图和terrain数据

我们需要完成以下工作：
1. 搭建离线地图服务
2. 搭建离线terrain服务
3. 在cesium中加载自己搭建的web服务

## 搭建离线地图服务
参见https://github.com/liwb27/tileserver

## 搭建离线terrain服务
### 下载terrain数据
全球高程数据可以从http://srtm.csi.cgiar.org/strmdata/下载。格式选择Geo TIFF，网站提供5*5°和30*30°两种大小的文件下载。

30*30°的数据文件中不包含坐标系信息，在后续处理过程中需要先添加坐标系信息。

### terrain数据切片
我们使用ctb-tile工具对下载好的高程数据进行处理。最简便的方法是通过doker运行，下载镜像tumgis/ctb-quantized-mesh，其中包含了完整的ctb-tile工具。

1. 启动docker镜像：`docker run -it -name ctb -v "D:/CesiumTerrain":"/data" tumgis/ctb-quantized-mesh`，其中`D:/CesiumTerrain`是本地存放terrain数据的目录。

2. 给tif添加坐标系信息。执行`gdal_translate -a_srs "WGS84" input.tif output.tif`可以给`input.tif`文件添加坐标系信息，并另存为`output.tif`，对所有的tif文件执行上述命令。

3. 合并tif。将所有处理好的tif文件放在同一个目录下，并在该目录下执行`gdalbuildvrt tiles.vrt *.tif`，该命令会生成一个名为`tiles.vrt`的索引文件，其中包含了所有的tif文件的信息。

4. 生成terrain数据。执行命令`ctb-tile -f Mesh -C -N -o terrain tiles.vrt`会在当前目录下创建terrain文件夹，并将处理好的terrain数据放在该文件夹下。处理完成后再执行`ctb-tile -f Mesh -C -N -l -o terrain tiles.vrt`生成分层描述信息`layer.json`

5. 当需要处理大量高程数据时（例如全国数据），可能会出现process被终止的情况，这是由于生成低层(level3,level4..)terrain文件时，会同时使用多个tif，这需要消耗大量内存，当机器性能不足时就会出现进程被终止的情况，这种情况官方文档给出了解释：

    ctb-tile will resample data from the source dataset when generating tilesets for the various zoom levels. This can lead to performance issues and datatype overflows at lower zoom levels(e.g. level0) when the source dataset is very large. To overcome this the tool can be used on the original dataset to only create the tile set at the highest zoom level(e.g. level 18) using the --start-zoom and --end-zoom options. Once this tileset is generated it can be turned into a GDAL Virtual Raster dataset for creating the next zoom level down(e.g. level17). Repeating this process until the lowest zoom level is created means that the resampling is much more efficient(e.g. level0 would be created from a VRT representation of level1). Because terrain tiles are not a format supported by VRT dataset you will need to perform this process in order to create tiles in a GDAL DEM fromat as an intermediate step. VRT representation of these intermediate tilesets can then be used to created the final terrain tile output.

5. 简言之就是我们需要根据高层(level5)数据生成一个中间文件，在根据这个中间文件去生成低层(level4)数据，这个中间文件会比原始文件小很多（相当于降低采样了），因此就可以解决性能问题。以level5到level4为例，具体步骤如下：

    1. 生成level5数据: `ctb-tile --output-dir ./tmp --start-level 5 --end-level 5 tiles.vrt`
    2. 生成level5的GDAL tileset: `ctb-tile --outpu-format GTiff --output-dir ./tiff --start-level 5 --end-level 5 tiles.vrt`
    3. 将所有的tiff合并成vrt文件: `gdalbuildvrt level5.vrt ./tiff/5/*/*.tif`
    4. 生成level4的terrain: `ctb-tile --output-dir ./terrain --start-level 4 --end-level 4 level5.vrt`
    5. 对于更低层的数据也可以此方法类推


### 搭建web服务
将生成好的terrain文件保存在`static/terrain/`下，已经切片好的terrain数据文件按照`/z/x/y`的目录形式组织，处理好的文件后缀名为`.terrain`。这个文件实际上是一个gzip压缩包，因此直接将文件夹作为静态资源提供web服务时，cesiumjs会报错无法读取terrain文件。因此我们需要创建一个web服务，修改response中的content_encoding，来让cesiumjs认识到这是一个gzip文件。

这里我们使用flask来搭建这个web服务，代码如下，也可以使用其他的webserver，主要是修改返回文件的content_encoding。
```python
@route('/terrain/<level>/<x>/<y>.terrain')
def terrain_tile(level,x,y):
    resp = send_from_directory('static/terrain/'+str(level)+'/'+str(x)+'/', str(y)+'.terrain')
    resp = make_response(resp)
    resp.content_encoding = 'gzip'
    return resp

@route('/terrain/layer.json')
def terrain():
    return send_from_directory('static/terrain/', 'layer.json')
```

## 在cesium中加载
在cesium中加载terrain可以使用`CesiumTerrainProvider`，代码如下，其中本地地图服务搭建在`localhost:8080`，本地terrain服务搭建在`localhost:5000`

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <script src="https://cesium.com/downloads/cesiumjs/releases/1.64/Build/Cesium/Cesium.js"></script>
    <link href="https://cesium.com/downloads/cesiumjs/releases/1.64/Build/Cesium/Widgets/widgets.css" rel="stylesheet">
</head>

<body>
    <div id="cesiumContainer" style="width: 1000px; height:600px"></div>

</body>
<script>
    var viewer = new Cesium.Viewer('cesiumContainer', {
        'imageryProvider': new Cesium.UrlTemplateImageryProvider({
            url: 'http://localhost:8080'/styles/satellite/{z}/{x}/{y}.png,
            format: "image/png",
        }),
    });
    var terrainLayer = new Cesium.CesiumTerrainProvider({
        url: 'http://localhost:5000/terrain',
    });
    viewer.scene.terrainProvicer = terrainLayer;
</script>
</html>
```
