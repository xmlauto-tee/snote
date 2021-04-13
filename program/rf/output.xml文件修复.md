```python
pip install robotfixml

# input.xml是有问题的xml文件，output.xml是修复后输出的xml文件
python -m robotfixml input.xml output.xml

# output.xml是修复后的文件
rebot output.xml

# 代码方式处理
from robotfixml import fixml
inpath = '/directory/example.xml'
outpath = '/directory/output/example-fixed.xml'
fixml(inpath, outpath)
```

工具参考链接：[Fixml](https://bitbucket.org/robotframework/fixml/src/master/)