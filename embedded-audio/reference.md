# 参考资源

## 说话人识别 / 分离(speaker recognition / diarization)

- **3D-Speaker 官方仓库(modelscope):** [https://github.com/modelscope/3D-Speaker](https://github.com/modelscope/3D-Speaker)
  - 聚类实现 `speakerlab/process/cluster.py`(`CommonClustering`:spectral / eigengap 估人数、p-pruning pval=0.012、min_cluster_size=4),由 `egs/sv-cam++/infer_diarization.py` 引用
- **CAM++ 论文:** [CAM++: A Fast and Efficient Network for Speaker Verification Using Context-Aware Masking (arXiv 2303.00332)](https://arxiv.org/abs/2303.00332)
  - D-TDNN 骨干 + context-aware masking;本项目使用的 192 维 embedding 模型即其产物的 ONNX 导出
- **3D-Speaker toolkit 论文:** [An Open-Source Toolkit for Multimodal Speaker Verification and Diarization (arXiv 2403.19971)](https://arxiv.org/abs/2403.19971)
- **CAM++ 预训练模型页(ModelScope):** [damo/speech_campplus_sv_zh-cn_16k-common](https://modelscope.cn/models/damo/speech_campplus_sv_zh-cn_16k-common)
  - 中文通用 16k 说话人确认/日志模型;另有 3D-Speaker 数据集版 `iic/speech_campplus_sv_zh-cn_3dspeaker_16k`

## VAD

- **Silero VAD:** [https://github.com/snakers4/silero-vad](https://github.com/snakers4/silero-vad)
  - 轻量 ONNX VAD,本项目实时流水线用 32ms 窗 + 阈值 0.5 切段
