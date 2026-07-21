# 供应链领域提取参考

## 典型对象类（canonical）

产品(`product`)、产品BOM(`bom`)、物料(`material`)、需求预测(`forecast`)、
产品需求计划(`pp`)、物料需求计划(`mrp`)、工厂生产计划(`mps`)、
采购申请(`pr`)、采购订单(`po`)、供应商(`supplier`)、库存(`inventory`)、销售订单(`sales_order`)

## 典型关系类

- 产品 `包含` BOM（1:N）
- BOM `引用` 物料（N:1）
- 物料 `形成` 采购申请（1:N）
- 采购申请 `转换为` 采购订单（1:N）
- 采购订单 `下达给` 供应商（N:1）
- 物料 `记录于` 库存（1:N）
- 需求预测 `驱动` 产品需求计划（N:1）
- 产品需求计划 `展开为` 物料需求计划（1:N）
- 产品需求计划 `驱动` 工厂生产计划（1:N）
- 销售订单 `关联` 产品（N:1）

## 领域闭环补全

| 命中对象 | 缺失时补全 |
|---------|-----------|
| pr 或 po | supplier |
| mrp | material |
| pp | mrp 或 mps |
| 到货/入库/库存语义 | inventory |

## 关系识别提示

- "转单、下达、审批、到货、入库、领料、齐套" → PR/PO/Inventory 相关
- "预测、计划、排程、缺口、净需求" → Forecast/PP/MRP/MPS 相关

## 主键候选

- 编码类：material_code、supplier_code、product_code
- 单据类：entry_id、billno、contract_number
- 流水类：seq_no
