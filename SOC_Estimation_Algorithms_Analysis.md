# 🔋 BMS SOC 估算算法对标分析

## 概述

您的项目关注于SOC估算，以下是从GitHub开源项目中收集的不同SOC估算方法实现。

---

## 1. SOC 估算方法对比

### 1.1 库伦计数法（Coulomb Counting）
**特点**: 简单、快速、易实现

**应用项目**:
- **LibreSolar/bms-firmware** - 主要方法
- **foxBMS/foxbms-2** - 辅助方法

**优点**:
- ✅ 计算快速
- ✅ 实时性好
- ✅ 易于硬件集成

**缺点**:
- ❌ 漂移问题
- ❌ 需要精确电流测量
- ❌ 不能自校正

**MATLAB实现核心**:
```matlab
SOC = SOC_init + (I_measured * dt) / (Q_nominal)
```

---

### 1.2 Kalman 滤波法 (推荐 ⭐⭐⭐⭐⭐)
**特点**: 融合多个信息源、自适应、抗干扰

**应用项目**:
- **geshan99/BMS-Project** ⭐ **最佳参考** - Kalman滤波完整实现
- **foxBMS/foxbms-2** - EKF（扩展Kalman滤波）实现

**优点**:
- ✅ 精度高
- ✅ 自适应
- ✅ 能融合多个传感器
- ✅ 能处理非线性问题（EKF）

**缺点**:
- ❌ 算法复杂
- ❌ 需要系统建模
- ❌ 调参困难

**geshan99 的实现细节**:
```
文件: kalman_filter.py
方法: 标准Kalman滤波
融合: 电压 + 电流 + 温度
输出: SOC估计值
特性: 200ms、400ms两种采样率版本
```

**MATLAB框架**:
```matlab
% 预测步
SOC_pred = SOC_k + (I/Q_cap)*Ts;

% 更新步 - 用实际电压修正
Z = V_measured;
H = dV/dSOC;  % 放电曲线斜率
K = P * H / (H*P*H + R);  % Kalman增益
SOC = SOC_pred + K * (Z - Z_pred);
```

---

### 1.3 扩展Kalman滤波（EKF）
**特点**: 处理非线性系统

**应用项目**:
- **foxBMS/foxbms-2** ⭐⭐ **最佳参考** - 企业级实现

**优点**:
- ✅ 可处理非线性电池特性
- ✅ 精度最高
- ✅ 能融合多个非线性观测值

**缺点**:
- ❌ 最复杂
- ❌ 计算量大
- ❌ 需要建立详细的电池模型

**foxBMS 的实现** (参考自文档):
```
系统模型：
  SOC(k+1) = SOC(k) - I(k)*Ts / (3600*Q_cap)
  
观测模型：
  V(k) = OCV(SOC) + R_int*I(k) + R_diffusion*I_filtered(k)
  
关键参数：
  - OCV: 开路电压-SOC曲线（分段函数）
  - R_int: 内阻
  - R_diffusion: 扩散电阻（与放电历史相关）
```

---

### 1.4 放电曲线法（Open Circuit Voltage）
**特点**: 简单但需要静置

**应用项目**:
- **foxBMS/foxbms-2** - 辅助方法
- **多数商业BMS** - 广泛应用

**优点**:
- ✅ 原理简单
- ✅ 精度可控
- ✅ 不需要参数优化

**缺点**:
- ❌ 需要长时间静置
- ❌ 受温度影响大
- ❌ 实时性差

---

## 2. 项目实现对标

### 2.1 geshan99/BMS-Project 详解

#### 文件结构
```
bms-project/
├── kalman_filter.py          ⭐ 核心算法
├── kalman_data.csv           - 测试数据
├── Kalman_Filter.csv         - 输出结果
├── voltage_sensor_calibration.py
├── current_sensor_calibration.py
└── data_visualization.py
```

#### 关键代码分析
```python
# kalman_filter.py - 标准Kalman实现

class KalmanFilter:
    def __init__(self, Q_cap=100):
        self.Q_cap = Q_cap  # 电池容量
        self.SOC = 0.5      # 初始SOC
        self.P = 1.0        # 误差协方差
        
    def predict(self, I, dt):
        # 预测步：库伦计数
        dSOC = (I * dt) / (3600 * self.Q_cap)
        self.SOC = self.SOC + dSOC
        
    def update(self, V_measured, V_est, R):
        # 更新步：用电压修正
        Z = V_measured
        H = 1.0  # 观测矩阵（线性化）
        Q = 0.001  # 过程噪声
        R = 0.01   # 测量噪声
        
        # Kalman增益
        self.P = self.P + Q
        K = self.P / (self.P + R)
        
        # 状态更新
        self.SOC = self.SOC + K * (Z - V_est)
        self.P = (1 - K) * self.P
        
        return self.SOC
```

#### 采样率对比
- **kalman_200ms.csv** - 更新频率高，响应快
- **kalman_400ms.csv** - 更新频率低，更稳定

#### 实际应用建议
- 初始化：需要准确的Q_cap和初始SOC
- 参数：Q和R的选择影响跟踪性能
- 校准：需要定期用静置电压校准

---

### 2.2 foxBMS/foxbms-2 详解

#### 架构
```
foxBMS-2/
├── src/
│   ├── module/
│   │   ├── state_estimation/    ⭐ SOC/SOH估算
│   │   ├── bms/                 ⭐ BMS核心
│   │   ├── battery/             - 电池参数
│   │   └── ...
│   └── driver/                  - 底层驱动
├── doc/
│   ├── html/
│   │   ├── user_guide.html
│   │   └── api_reference.html
```

#### SOC 估算实现
**三种方法融合**:
```
1. 库伦计数 (Coulomb Counting)
   - 快速计算
   - 易漂移
   
2. Kalman滤波
   - 融合电压信息
   - 自动校正
   
3. Open Circuit Voltage
   - 长期参考值
   - 静置获取
```

**融合策略**:
```
SOC_final = w1*SOC_coulomb + w2*SOC_kalman + w3*SOC_ocv
其中权重根据系统状态动态调整
```

#### 关键参数文件
- `cell_params.c` - 电芯特性参数
- `ocv_soc_table.c` - OCV-SOC查表
- `system_config.h` - 系统配置

#### 温度补偿
```c
// 温度对SOC精度的影响
SOC_corrected = SOC_base + f_temp(T, SOC)
// 其中 f_temp() 是温度校正函数
```

---

## 3. 推荐实现方案 for MATLAB

### 方案 A: 基础Kalman（推荐入门）

```matlab
% 基于 geshan99 改进
function SOC_est = estimate_SOC_kalman_basic(I, V, T_sample)
    persistent SOC P Q_cap
    if isempty(SOC)
        SOC = 0.5;      % 初始SOC
        P = 0.001;      % 初始误差协方差
        Q_cap = 100;    % 容量Ah
    end
    
    % 1. 预测 - 库伦计数
    dSOC = (I * T_sample) / (3600 * Q_cap);
    SOC_pred = SOC + dSOC;
    
    % 2. 更新 - 用OCV修正
    V_ocv_pred = ocv_lookup(SOC_pred);
    V_error = V - V_ocv_pred;
    
    % 3. Kalman增益
    Q = 0.0001;  % 过程噪声
    R = 0.01;    % 测量噪声
    P = P + Q;
    K = P / (P + R);
    
    % 4. 状态和协方差更新
    dSOC_correction = K * V_error / 0.5;  % 假设OCV斜率0.5V/SOC
    SOC = SOC_pred + dSOC_correction;
    P = (1 - K) * P;
    
    SOC_est = SOC;
end

function V = ocv_lookup(SOC)
    % OCV-SOC查表（示例）
    ocv_table = [2.5, 3.0, 3.5, 4.0, 4.2];  % 电压
    soc_table = [0, 0.25, 0.5, 0.75, 1.0];  % SOC
    V = interp1(soc_table, ocv_table, SOC);
end
```

### 方案 B: 扩展Kalman（推荐高精度）

```matlab
% 基于 foxBMS 改进
function [SOC_est, SOH_est] = estimate_SOC_EKF(I, V, T, T_sample)
    persistent x P Q R
    
    if isempty(x)
        x = [0.5; 1.0];  % 状态: [SOC, SOH]
        P = eye(2) * 0.01;
        Q = diag([0.0001, 0.00001]);
        R = 0.01;
    end
    
    % 非线性系统模型
    % 状态方程: x(k+1) = f(x(k), u(k))
    % 观测方程: y(k) = h(x(k), u(k))
    
    % 1. 预测
    F = [ 1 - I*T_sample/(3600*100*x(2)), -I*T_sample/(3600*100*x(1))^2;
          0, 1 ];
    x_pred = [ x(1) - I*T_sample/(3600*100*x(2));
               x(2) ];
    
    P = F*P*F' + Q;
    
    % 2. 观测
    V_ocv = ocv_lookup_T(x(1), T);
    V_pred = V_ocv + 0.01 * I;  % 简化，忽略阶跃
    
    % 3. 更新
    H = [docv_dSOC(x(1), T), 0];
    y = V - V_pred;
    S = H*P*H' + R;
    K = P*H' / S;
    
    x = x_pred + K * y;
    P = (eye(2) - K*H) * P;
    
    SOC_est = x(1);
    SOH_est = x(2);
end
```

---

## 4. 参数优化指南

### 4.1 噪声协方差选择

| 参数 | 范围 | 含义 | 调参建议 |
|------|------|------|---------|
| Q (过程噪声) | 1e-6 ~ 1e-4 | 模型不确定性 | 模型准度差→增大 |
| R (测量噪声) | 0.001 ~ 0.1 | 传感器噪声 | 传感器差→增大 |
| Q/R 比 | 0.001 ~ 10 | 相对权重 | 偏信任模型↑比值 |

### 4.2 OCV表的重要性

**完整的OCV-SOC表**:
```matlab
% 25°C 下的LiFePO4
SOC    = [0,    0.1,   0.2,   0.5,   0.8,   0.9,   1.0];
OCV_25 = [2.5,  2.8,   3.0,   3.2,   3.35,  3.38,  3.4];

% 温度补偿系数（简化）
T_coef = -0.002;  % -2mV/°C
```

### 4.3 温度补偿

```matlab
function V_ocv = ocv_lookup_T(SOC, T)
    % 基准温度 (25°C)
    V_ocv_25 = ocv_lookup(SOC);
    
    % 温度补偿
    T_ref = 25;
    T_coef = -0.002;  % V/°C
    V_ocv = V_ocv_25 + T_coef * (T - T_ref);
end
```

---

## 5. 验证和优化

### 5.1 性能指标

```matlab
% 与实际容量对比
errors = SOC_estimated - SOC_actual;
MAE = mean(abs(errors));           % 平均绝对误差
RMSE = sqrt(mean(errors.^2));      % 均方根误差
max_error = max(abs(errors));      % 最大误差

% 显示结果
fprintf('SOC估算精度:\n');
fprintf('  MAE:  %.2f%%\n', MAE*100);
fprintf('  RMSE: %.2f%%\n', RMSE*100);
fprintf('  Max:  %.2f%%\n', max_error*100);
```

### 5.2 调试建议

1. **先用库伦计数baseline**
   - 检查电流测量准确性
   - 验证容量参数

2. **加入OCV修正**
   - 构建精确的OCV表
   - 检查温度影响

3. **集成Kalman滤波**
   - 从高Q值开始（更信任测量）
   - 逐步调整Q和R

4. **验证稳定性**
   - 长时间充放电测试
   - 温度变化测试
   - 老化循环测试

---

## 6. 项目推荐

### 直接对标
- **参考 geshan99** 学习Kalman实现细节
- **参考 foxBMS-2** 学习工程实践

### 快速开始
```bash
# 下载参考实现
git clone https://github.com/geshan99/BMS-Project
cd BMS-Project

# 查看关键文件
cat kalman_filter.py
cat Kalman_Filter.csv

# 用MATLAB导入并分析
data = readtable('Kalman_Filter.csv');
figure;
plot(data.Time, data.SOC_Estimated, 'b-', ...
     data.Time, data.SOC_Actual, 'r--');
legend('估计', '实际');
```

---

## 7. 总结

| 方法 | 精度 | 速度 | 复杂度 | 推荐度 |
|------|------|------|--------|--------|
| 库伦计数 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ baseline |
| Kalman | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| EKF | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ 高端 |
| OCV | ⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐ 参考值 |

**建议实现路线**:
1. 从库伦计数开始验证基础参数
2. 加入OCV修正改进精度
3. 集成Kalman滤波提高稳定性
4. 可选：升级到EKF获得最高精度

---

*参考项目: geshan99/BMS-Project, foxBMS/foxbms-2*
*更新日期: 2026-06-05*
