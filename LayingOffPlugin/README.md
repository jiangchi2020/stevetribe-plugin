# 放样插件

施工放样：把设计图纸上工程建筑物的平面位置和高程，用一定的测量仪器和方法测设到实地上去的测量工作称为施工放样

本插件中，虽然原理和工程测量中的放样操作完全不同，但是请允许我用这个略带专业的名字以提高逼格

本插件，用于将地理要素在minecraft世界中标定出来，仅限于op执行此操作。

## 插件配置
插件的配置文件中，描述了图层与方块的映射关系

默认的，使用白色羊毛生成结构

使用该配置，可以自定义映射的颜色

## 数据准备

本项目中，插件的直接数据来源是GeoJson数据格式，因为：

1. GeoJson人类可读，便于调试
2. GeoJson是OGC通用格式，大部分地理数据shapefile，dwg，乃至postgis等数据库均可简单地转换为GeoJSON
3. 开发环境下没有查看DWG的软件，安装起来太麻烦，直接解析DWG不方便调试

简要备份一下DWG转GeoJSON方式：

1. DWG转DXF：使用java代码库aspose-cad可以方便完成
```java
CadImage cad = (CadImage)Image.load(new File("xxx.dwg"));
cad.save("XXX.dxf");
```
当然用其他专业软件也可以

2. DXF转GeoJSON：使用gdal命令行即可
```shell
sudo apt install gdal-bin
ogr2ogr -f GeoJSON ./xxx.dxf ./xxx.geojson
```

spigot貌似并没有提供从客户端上传文件供服务器解析的功能，所有需要提前将数据准备好放在spigot环境中。

## 坐标变换

并不用关心源数据的坐标系统和比例尺，只需要将数字坐标转换为Minecraft坐标即可。

通过在Minecraft世界中标定若干控制点在源数据坐标系下的坐标，建立仿射变换模型，算出变换系数矩阵。

在此基础上实现源坐标到Minecraft坐标的转换。

使用```//set-control-point```指令，后跟源数据坐标系下的坐标值，将当前玩家所在位置设为取样点

在1.0.0版本中，仅支持二维层面的映射（X，Y），使用geotools库方法实现仿射变换，需要采取三个取样点，推荐分散取点

预计在下一个版本中，会支持三维坐标（X，Y与高程值）的坐标变换，支持添加多个采样点以提供更高的精度并提供精度评估

## 施工放样

使用```//laying-off```指令，后跟源数据名（源数据应提前存放在geo-data文件夹中）

之后插件就会根据设置的取样点结算变换矩阵，读取数据源并将坐标转换为Minecraft坐标并返回，插件根据这些坐标，在对应位置设置地物方块

该指令可能比较耗时，会导致服务器的插件逻辑线程产生长时间未响应错误，具体则取决于GeoJSON文件规模。

Spigot线程模型中，使用单线程循环处理插件的逻辑，所有并不推荐使用异步线程处理与世界编辑有关的任务
（PS：貌似非主线程执行非异步的Spigot API会报错的？依稀记得是这样，懒得验证了。同样的，本插件的代码中也就不考虑线程安全问题了）

## 事务管理

放样操作会对地形方块造成不可逆的影响，为了避免此类危险，插件在放样时会保存该位置原始方块以供恢复

使用```//begin```命令开启一个事务，事务隔离级别为最简单粗暴的串行化，即同时执行执行一个放样工程

之后便可以使用上述两个指令执行操作，效果实时反馈到客户端

如果一切正常，使用命令```//commit```结束该事务，该操作将会清除GeoJSON在服务器中的缓存。

如果需要撤销，使用命令```//rollback```将在laying-off时替换的方块替换回去，然后清除缓存。

## 实际应用

如果你有了一份地理信息数据，通过确定采样点的方式，快速省时省力的将其投影到世界中

插件直接面向GeoJSON格式数据，所以任何地理数据只要能够转成GeoJSON都能操作，常见的有：DWG,DXF,SHP,SVG等

不过一般来说只有矢量地理数据会支持转成GeoJSON格式，栅格数据可以使用图片生成插件，现在社区中此类插件已经很丰富了

如果你有一张航拍图，此类的栅格数据，可以使用一些影像处理软件，将其绘制成矢量格式，然后转为GeoJSON

### 一般操作流程案例

1. 你有一份DWG格式的地形图：
    1. 选择一个建筑（或地物）在世界合适的地方勾出轮廓
    2. 选择这个建筑的若干边角点或其他方便辨认的点，找到他们在原始数据中的坐标
    3. 让玩家站在这些点上，开启事务，使用set-control-point指令后跟坐标值采样
    4. 重复这些流程知道选取三个或以上的采用点
    5. 将DWG格式转为GeoJSON，放置于geo-data文件夹中
    6. 使用laying-off指令后跟文件名，将其余部分已选取的建筑物为基准投影到世界中
    7. 提交事务或回滚事务
    
2. 你有一份航拍栅格数据
    1. 取几个标志性的点，在对应的位置放置方块，并取样
    2. 使用影像处理软件，将航摄影像中需要建造的部分编辑为矢量数据，后转为GeoJSON
    3. laying-off指令启动

以上操作中，并不强制数据有标准的空间参考，只需要他们的相对位置符合预期即可。坐标变换由程序根据采样点自动完成。