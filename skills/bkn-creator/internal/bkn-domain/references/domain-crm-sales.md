# CRM/销售领域提取参考

## 典型对象类（canonical）

客户(`customer`)、线索(`lead`)、商机(`opportunity`)、报价单(`quotation`)、
合同(`contract`)、销售订单(`sales_order`)、回款(`payment`)、销售人员(`sales_rep`)

## 典型关系类

- 客户 `产生` 线索（1:N）
- 线索 `转化为` 商机（1:N 或 1:1）
- 商机 `生成` 报价单（1:N）
- 报价单 `签订为` 合同（1:N）
- 合同 `履约为` 销售订单（1:N）
- 销售订单 `对应` 回款（1:N）
- 销售人员 `负责` 客户（1:N）

## 领域闭环补全

| 命中对象 | 缺失时补全 |
|---------|-----------|
| quotation | contract |
| contract | sales_order |
| sales_order | payment |
| 客户经理/归属/分配语义 | sales_rep 或 customer |

## 关系识别提示

- "跟进、转化、赢单、签约、续约、回款" → 线索到合同链路
- "客户分配、客户经理、账户归属" → 销售人员与客户关系

## 主键候选

- 编码类：customer_code、lead_code、opportunity_code
- 单据类：quotation_no、contract_no、order_no、payment_no
- 人员类：sales_rep_id、employee_id
