# 问题说明

我正在使用机器学习模型进行土壤盐渍化遥感反演，并希望完善我的代码。现有代码如下`new_code_codex_v3.ipynb`（包含多个步骤，例如数据处理、模型训练、特征提取等），但是还有一些功能需要改进。具体来说，我希望你参考旧版代码`old_code_claude.ipynb`，使改进的代码的功能与`old_code_claude.ipynb`接近，只是在调用数据上由地下水数据改为土壤数据，土壤数据路径：`projects/ee-hanxu0223/assets/Soil_Raster`，配置参数保持不变：

# ==================== 1. 配置参数 ====================
# 时间范围 - 扩展到7-9月
START_DATE = '2022-07-01'
END_DATE = '2022-10-01'

# 空间范围
BOUNDARY_PATH = '/Users/hanxu/geemap/bdy.shp'
bdy = geemap.shp_to_ee(BOUNDARY_PATH)

# 统一的目标投影和分辨率 - 使用30m以适配所有数据源
TARGET_SCALE = 30  # 30米分辨率
TARGET_CRS = 'EPSG:4326'  # WGS84

# 输出路径
OUTPUT_DIR = '/Users/hanxu/geemap/out_plots'
os.makedirs(OUTPUT_DIR, exist_ok=True)
OUTPUT_CSV = 'multi_source_features_2022_07_09.csv'
SOIL_DATA_PATH = '/Users/hanxu/geemap/material/soil/soil_2022_08.csv'


## 文件说明

* `old_code_claude.ipynb`：旧版本代码，由Claude编写。
* `newcode_cell1.ipynb` 与 `newcode_cell2.ipynb`：部分实现新技术路线的代码片段。
* `gee data`：gee的官方数据集调用、bands、示例代码。

## 研究目标

* 建立基于2022年8月实测数据的土壤盐渍化遥感模型。
* 利用2022年7-9月遥感数据中值合成，确保研究区完整覆盖。
* 融合光学（Landsat 8）、雷达（Sentinel-1）、气象（MODIS、CHIRPS）、地形（SRTM DEM）和土壤类型数据。
* 目标精度：R² ≥ 0.60，RMSE ≤ 3.0 dS·m⁻¹。
