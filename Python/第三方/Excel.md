使用`xlrd`和`xlwt`库

### xlwt

#### 新建Excel

```python
book = xlwt.Workbook()
book.save("excel_file.xls")
```

#### 写入数据

```python
sheet = book.add_sheet(table_map[2])
sheet.write(0, 0, "text")
```

#### [格式](https://github.com/python-excel/xlwt/blob/master/examples/num_formats.py)

##### 数字

```python
percent_style = xlwt.XFStyle()
percent_style.num_format_str = "0.00%"
sheet.write(0, 0, 0.9999, percent_style)
```

#### [样式](http://blog.sina.com.cn/s/blog_5357c0af01019gjo.html)

##### 字体

```python
font = xlwt.Font() 
font.bold = True
font.height = 255
title_style = xlwt.XFStyle() 
title_style.font = font
sheet.write(0, 0, "数据标识", title_style)
```

##### 宽度

```python
sheet.col(0).width = 263 * 30
```

