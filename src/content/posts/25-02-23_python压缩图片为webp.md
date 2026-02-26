---
title: python压缩图片为webp
published: 2025-02-23 08:00:00
description: python压缩图片为webp
tags: [python, 博客]
category: 技术博客
---

把博客下的png压下

递归批量将图片转为Webp并替换原文件

安装库
```bash
pip install pillow
```


### 完整脚本代码
```python
import os
from PIL import Image
from PIL.UnidentifiedImageError import UnidentifiedImageError

current_dir = os.getcwd()
#（默认80兼顾画质和体积）
compression_quality = 80
image_extensions = ('.jpg', '.jpeg', '.png', '.gif', '.bmp', '.tiff', '.tif')

print(f"开始递归转换 {current_dir} 下的所有图片为 webp 格式（替换原文件,压缩质量：{compression_quality}）...\n")

# 递归遍历当前目录所有文件
for dirpath, _, filenames in os.walk(current_dir):
    for filename in filenames:
        # 筛选出需要处理的图片文件
        if filename.lower().endswith(image_extensions):
            original_path = os.path.join(dirpath, filename)
            # 临时webp路径（避免转换中覆盖原文件导致损坏）
            temp_webp_path = os.path.splitext(original_path)[0] + '_temp.webp'
            
            try:
                # 打开图片并处理透明通道（适配PNG等带透明背景的图片）
                with Image.open(original_path) as img:
                    if img.mode in ("RGBA", "P"):
                        img = img.convert("RGBa")
                    # 保存为临时webp文件
                    img.save(temp_webp_path, 'webp', quality=compression_quality, optimize=True)
                
                # 计算压缩率
                original_size = os.path.getsize(original_path) / 1024  # 单位：KB
                new_size = os.path.getsize(temp_webp_path) / 1024
                compression_ratio = (1 - new_size/original_size) * 100
                
                # 删除原文件,将临时webp重命名为原文件名
                os.remove(original_path)
                os.rename(temp_webp_path, original_path)
                
                print(f"替换完成")
                print(f"压缩率: {compression_ratio:.1f}% (原{original_size:.1f}KB → 新{new_size:.1f}KB)\n")
            
            except UnidentifiedImageError:
                print(f"无法识别的图片文件: {original_path}\n")
            except Exception as e:
                print(f"处理失败 {original_path}: {str(e)}\n")
                # 清理转换失败的临时文件,避免残留垃圾
                if os.path.exists(temp_webp_path):
                    os.remove(temp_webp_path)

print("所有图片转换替换完成")
```

**替换操作不可逆转,替换完成后会自动删除原图片,只保留webp格式文件.建议先备份重要图片**
