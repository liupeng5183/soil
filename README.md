以下是整合后的README文件，包含了目标描述和数据集的使用信息，以供Codex更清晰地理解目标。

---

# 土壤盐渍化遥感反演项目

本项目基于Geemap平台，通过遥感技术实现银川平原土壤盐渍化反演，包含旧版代码和基于新技术路线的新代码。

## 项目目标

基于多源遥感数据，包括Landsat 8光学影像、Sentinel-1合成孔径雷达数据、MODIS植被指数、CHIRPS降水、数字高程模型（DEM）和土壤类型等，提取了多种与土壤盐渍化相关的光谱指数和环境因子，并结合野外采集的0–20 cm表层土壤盐分含量样本，构建并比较了多种机器学习模型（如随机森林RF、极端梯度提升树XGBoost、支持向量回归SVR以及多元线性回归MLR）的精度。

结果表明：融合多源数据可显著提高土壤盐渍化反演精度，相较于单一光学数据的模型，测试集R²提高约（）%；其中XGBoost模型精度最高，测试集决定系数R²约（），均方根误差RMSE约为（）dS·m⁻¹，明显优于RF模型和传统线性回归。特征贡献分析显示，（）因子是影响盐渍化反演精度的主导变量。

## 文件说明

* `old_code_claude.ipynb`：旧版本代码，由Claude编写。
* `newcode_cell1.ipynb` 与 `newcode_cell2.ipynb`：部分实现新技术路线的代码片段。

## 研究目标

* 建立基于2022年8月实测数据的土壤盐渍化遥感模型。
* 利用2022年7-9月遥感数据中值合成，确保研究区完整覆盖。
* 融合光学（Landsat 8）、雷达（Sentinel-1）、气象（MODIS、CHIRPS）、地形（SRTM DEM）和土壤类型数据。
* 目标精度：R² ≥ 0.60，RMSE ≤ 3.0 dS·m⁻¹。

## 数据源

### 1. **Landsat 8 Level 2, Collection 2, Tier 1**

数据集：`ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")`

* **像素大小**：30米
* **波段信息**：

| 波段名称   | 单位   | 最小值 | 最大值   | 缩放因子     | 偏移量  | 波长范围           | 描述                                                       |
| ------ | ---- | --- | ----- | -------- | ---- | -------------- | -------------------------------------------------------- |
| SR\_B1 | None | 1   | 65455 | 2.75e-05 | -0.2 | 0.435-0.451 μm | Band 1 (ultra blue, coastal aerosol) surface reflectance |
| SR\_B2 | None | 1   | 65455 | 2.75e-05 | -0.2 | 0.452-0.512 μm | Band 2 (blue) surface reflectance                        |
| SR\_B3 | None | 1   | 65455 | 2.75e-05 | -0.2 | 0.533-0.590 μm | Band 3 (green) surface reflectance                       |
| SR\_B4 | None | 1   | 65455 | 2.75e-05 | -0.2 | 0.636-0.673 μm | Band 4 (red) surface reflectance                         |
| SR\_B5 | None | 1   | 65455 | 2.75e-05 | -0.2 | 0.851-0.879 μm | Band 5 (near infrared) surface reflectance               |
| SR\_B6 | None | 1   | 65455 | 2.75e-05 | -0.2 | 1.566-1.651 μm | Band 6 (shortwave infrared 1) surface reflectance        |
| SR\_B7 | None | 1   | 65455 | 2.75e-05 | -0.2 | 2.107-2.294 μm | Band 7 (shortwave infrared 2) surface reflectance        |

**示例代码：**

```python
dataset = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2').filterDate(
    '2021-05-01', '2021-06-01'
)

# Apply scaling factors
def apply_scale_factors(image):
  optical_bands = image.select('SR_B.').multiply(0.0000275).add(-0.2)
  thermal_bands = image.select('ST_B.*').multiply(0.00341802).add(149.0)
  return image.addBands(optical_bands, None, True).addBands(thermal_bands, None, True)

dataset = dataset.map(apply_scale_factors)

visualization = {
    'bands': ['SR_B4', 'SR_B3', 'SR_B2'],
    'min': 0.0,
    'max': 0.3,
}

m = geemap.Map()
m.set_center(-114.2579, 38.9275, 8)
m.add_layer(dataset, visualization, 'True Color (432)')
m
```

### 2. **Sentinel-1 SAR GRD**

数据集：`ee.ImageCollection("COPERNICUS/S1_GRD")`

* **波段信息**：

| 波段名称 | 单位 | 最小值   | 最大值 | 像素大小 | 描述                                                                 |
| ---- | -- | ----- | --- | ---- | ------------------------------------------------------------------ |
| HH   | dB | -50\* | 1\* |      | Single co-polarization, horizontal transmit/horizontal receive     |
| HV   | dB | -50\* | 1\* |      | Dual-band cross-polarization, horizontal transmit/vertical receive |
| VV   | dB | -50\* | 1\* |      | Single co-polarization, vertical transmit/vertical receive         |
| VH   | dB | -50\* | 1\* |      | Dual-band cross-polarization, vertical transmit/horizontal receive |

**示例代码：**

```python
def mask_edge(image):
  edge = image.lt(-30.0)
  masked_image = image.mask().And(edge.Not())
  return image.updateMask(masked_image)

img_vv = (
    ee.ImageCollection('COPERNICUS/S1_GRD')
    .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
    .filter(ee.Filter.eq('instrumentMode', 'IW'))
    .select('VV')
    .map(mask_edge)
)

desc = img_vv.filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
asc = img_vv.filter(ee.Filter.eq('orbitProperties_pass', 'ASCENDING'))

spring = ee.Filter.date('2015-03-01', '2015-04-20')
late_spring = ee.Filter.date('2015-04-21', '2015-06-10')
summer = ee.Filter.date('2015-06-11', '2015-08-31')

desc_change = ee.Image.cat(
    desc.filter(spring).mean(),
    desc.filter(late_spring).mean(),
    desc.filter(summer).mean(),
)

asc_change = ee.Image.cat(
    asc.filter(spring).mean(),
    asc.filter(late_spring).mean(),
    asc.filter(summer).mean(),
)

m = geemap.Map()
m.set_center(5.2013, 47.3277, 12)
m.add_layer(asc_change, {'min': -25, 'max': 5}, 'Multi-T Mean ASC', True)
m.add_layer(desc_change, {'min': -25, 'max': 5}, 'Multi-T Mean DESC', True)
m
```

### 3. **MODIS Net Evapotranspiration (MOD16A2.061)**

数据集：`ee.ImageCollection("MODIS/061/MOD16A2")`

* **像素大小**：500米
* **波段信息**：

| 波段名称 | 单位         | 最小值    | 最大值   | 缩放因子  | 描述                                 |
| ---- | ---------- | ------ | ----- | ----- | ---------------------------------- |
| ET   | kg/m²/8day | -32767 | 32700 | 0.1   | Total evapotranspiration           |
| LE   | J/m²/day   | -32767 | 32700 | 10000 | Average latent heat flux           |
| PET  | kg/m²/8day | -32767 | 32700 | 0.1   | Total potential evapotranspiration |

**示例代码：**

```javascript
var dataset = ee.ImageCollection('MODIS/061/MOD16A2')
                  .filter(ee.Filter.date('2022-01-01', '2022-05-01'));
var evapotranspiration = dataset.select('ET');
var evapotranspirationVis = {
  min: 0,
  max: 300,
  palette: [
    'ffffff', 'fcd163', '99b718', '66a000', '3e8601', '207401', '056201',
    '004c00', '011301'
  ],
};
Map.setCenter(0, 0, 2);
Map.addLayer(evapotranspiration, evapotranspirationVis, 'Evapotranspiration');
```

### 4. **CHIRPS Daily Precipitation**

数据集：`ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY")`

* **像素大小**：5566米
* **波段信息**：

| 波段名称          | 单位   | 最小值 | 最大值       | 描述            |
| ------------- | ---- | --- | --------- | ------------- |
| precipitation | mm/d | 0\* | 1444.34\* | Precipitation |

**示例代码：**

```python
dataset = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY').filter(
    ee.Filter.date('2018-05-01', '2018-05-03')
)
precipitation = dataset.select('precipitation')
precipitation_vis = {
    'min': 1,
    'max': 17,
    'palette': ['001137', '0aab1e', 'e7eb05', 'ff4a2d', 'e90000'],
}

m = geemap.Map()
m.set_center(17.93, 7.71, 2)
m.add_layer(precipitation, precipitation_vis, 'Precipitation')
m
```

### 5. **SRTM Digital Elevation**

数据集：`ee.Image("USGS/SRTMGL1_003")`

* **像素大小**：30米
* **波段信息**：

| 波段名称      | 单位 | 描述        |
| --------- | -- | --------- |
| elevation | m  | Elevation |

**示例代码：**

```javascript
var dataset = ee.Image('USGS/SRTMGL1_003');
var elevation = dataset.select('elevation');
var slope = ee.Terrain.slope(elevation);
Map.setCenter(-112.8598, 36.2841, 10);
Map.addLayer(slope, {min: 0, max: 60}, 'slope');
```

### 6. **ESA WorldCover**

数据集：`ee.ImageCollection("ESA/WorldCover/v200")`

* **像素大小**：10米
* **波段信息**：

| 波段名称 | 描述              |
| ---- | --------------- |
| Map  | Landcover class |

**示例代码：**

```javascript
var dataset = ee.ImageCollection('ESA/WorldCover/v200').first();

var visualization = {
  bands: ['Map'],
};

Map.centerObject(dataset);
Map.addLayer(dataset, visualization, 'Landcover');
```
