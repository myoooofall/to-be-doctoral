# H2O
reference:Learning Human-to-Humanoid Real-Time Whole-Body Teleoperation
![H2O_pipeline ](./humanoid/H2O_pipeline.png)  
感觉还是AMP那套

# maskmimic
说白了就是先用一个teacher训一个给全部命令和约束的  然后蒸馏一个输入部分命令或约束的 student policy 起到一个最终只有一点约束或命令 就能生成全部动作的效果  
