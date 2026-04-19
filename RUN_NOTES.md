# 本地运行记录

## 1. 环境与运行方式
- 项目路径：`F:\sn-gamestate`
- 主要运行命令：`uv run tracklab -cn soccernet`
- 本次最终成功跑通使用的是 **`.venv + CPU`**
- 机器虽然有 NVIDIA GPU，但当前项目依赖栈与本机显卡环境兼容性不理想，因此这次没有采用 CUDA 路线，而是先用 CPU 跑通完整流程

## 2. 数据集准备
- 自动下载过程中出现过任务名不一致、下载中断、代理/SSL 报错等问题
- 后续改为手动下载 `gamestate-2024`
- 下载完成后，手动解压为如下目录结构：

    data/
      SoccerNetGS/
        train/
        valid/
        test/
        challenge/

完成后，项目可以正常识别四个 split，并顺利加载数据集

## 3. 遇到的问题

### 问题一：数据集自动下载异常

**现象：**
- 自动下载过程中出现中断
- 任务名与本仓库 README 中使用的版本不一致
- 下载完成后还可能因为额外统计请求报代理/SSL 错误

**处理方法：**
- 不再依赖自动下载
- 改为手动下载 `gamestate-2024`
- 手动解压 `train` / `valid` / `test` / `challenge`

### 问题二：缺少 mmcv

**现象：**
- 运行到球员号码识别相关模块时报错：`No module named 'mmcv'`

**处理方法：**
- 在项目环境中补装 `mmcv==2.0.1`
- 安装完成后，`mmocr` 相关模块可以继续运行

### 问题三：评测阶段路径不一致

**现象：**
- 跟踪和可视化已经完成
- 但 evaluation 阶段报错，找不到评测目录

**具体表现：**
- 预测结果保存到了：`eval\pred\SoccerNetGameState-valid\tracklab`
- 评测阶段却去读取：`eval\pred\SoccerNetGS-valid`

**原因：**
- 保存预测结果时使用的数据集名称
- 和 TrackEval 构造 benchmark 名称时使用的数据集名称
- 两者不一致，导致最后找不到路径

## 4. 本地修复

**修复文件位置：**

    .venv\Lib\site-packages\tracklab\wrappers\eval\trackeval_evaluator.py

**原始代码：**

    self.trackeval_dataset_name = type(self.tracking_dataset).__name__
    self.trackeval_dataset_class = getattr(trackeval.datasets, cfg.dataset.dataset_class)

**修改后代码：**

    self.trackeval_dataset_class = getattr(trackeval.datasets, cfg.dataset.dataset_class)
    self.trackeval_dataset_name = self.trackeval_dataset_class.__name__

**修复效果：**
- 统一了预测结果保存目录名称
- 统一了 GT 保存目录名称
- 统一了 TrackEval 使用的 benchmark 名称
- 解决了 evaluation 阶段路径不一致的问题

## 5. 复用中间结果，避免重跑

第一次完整 tracking 耗时较长，因此在修复评测问题后，没有重新从头跑完整 pipeline，而是直接复用之前保存的 TrackerState

**使用的状态文件：**

    F:\sn-gamestate\outputs\sn-gamestate\2026-04-19\13-43-54\states\sn-gamestate.pklz

**恢复命令：**

    uv run tracklab -cn soccernet "state.load_file=F:/sn-gamestate/outputs/sn-gamestate/2026-04-19/13-43-54/states/sn-gamestate.pklz" pipeline=[]

**说明：**
- `state.load_file=...` 用于加载之前已经保存好的中间结果
- `pipeline=[]` 用于跳过原本耗时很长的检测、跟踪和识别流程
- 这样可以快速重新执行 visualization 和 evaluation，而不需要再跑几小时

## 6. 最终结果

本地最终已成功完成：
- 数据集读取
- 检测 / 跟踪 / 识别流程
- 可视化输出
- 最终评测

**最终关键结果：**

    GS-HOTA = 38.377%

成功运行后，新输出目录位于：

    F:\sn-gamestate\outputs\sn-gamestate\2026-04-19\17-18-14

重点内容包括：
- `visualization`：可视化结果
- `states`：保存的 TrackerState

## 7. 备注

- 这次最关键的修复是在 `.venv` 里的第三方包文件中完成的
- 因此该修改不会被 git 自动跟踪
- 直接 `git push` 不会把这个修复一起上传到 GitHub
- 如果以后重建环境，可能需要重新手动修改这一处
- **因此建议将本说明文件提交到仓库中，方便后续复现与排查问题**