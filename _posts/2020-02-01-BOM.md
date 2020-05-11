---
title: 基于 Python 和 SolidWorks API 自动生成 BOM
description: 本文研究了基于 Python 和 SolidWorks API 遍历产品结构并获取零部件数据的方法，同时通过将数据写入 Excel 与 xml 文件实现了 BOM 的自动生成。这不仅有助于简化工程师的相关工作，而且对于产品信息的标准化、BOM文件的规范化有着重要意义。
categories:
- SolidWorks 二次开发
tags:
- SolidWorks
- Python
- BOM
typora-copy-images-to: ../_posts
---

BOM（Bill of Material，即物料清单）是一种以数据格式描述产品结构的文件，包含了产品所有零部件的型号、名称、原材料种类和数量等相关信息，是企业销售、计划、设计、生产、供应、核算、产品维护等工作环节的重要依据。

## BOM 信息框架

在产品设计阶段，工程师通过 `SolidWorks` 以自定义属性的形式为 CAD 模型赋予了一些基础信息，通过它们可以建立起基本的 BOM。为了存储这些零部件基础信息，在程序中创建了 `BomInfo()` 类：

```python
class BomInfo():
    'To store the information of components.'
    def __init__(self):
        self.dwg_no = ''
        self.dwg_name = ''
        self.material = ''
        self.designer = ''
        self.drawer = ''
        self.date_design = ''
        self.remark = ''
```

## 软件 UI 与功能

使用 `QT Designer` 设计了 BOM 生成软件的用户界面，并使用 `Python` 编写了业务逻辑。

![bom_creator_ui](http://raynnmc.github.io/img/bom_creator_ui.jpeg)

如图所示，用户可指定装配体文件路径、是否保存缩略图、缩略图保存路径、产品总套数等信息。界面右上方的 `运行` 按钮用于触发遍历装配体特征树及获取缩略图的功能模块，`导出` 按钮用于触发将 BOM 信息保存为 `.xlsx` 或 `.xml` 格式文件的功能模块。同时，在软件的运行过程中，运用多线程编程在软件的主窗口中动态实时显示当前运行状态等信息。

## 遍历 SolidWorks 装配体的特征树

BOM 是基于产品结构及其零部件所包含的信息建立的，因此需要基于产品总装配体的 CAD 模型来获取产品的结构以及零部件的基础信息。

本文以遍历 `SolidWorks` 装配体模型的特征树的方法来获取这些信息。该方法无需预先绘制产品总装配体的工程图，能够保留特征树的顺序与层级，并且支持直接对零部件模型进行操作，更为灵活和自由。

模块核心代码如下：

```python
part = sw_app.ActiveDoc
bom_dic = {}    # {编号:BomInfo对象}，用于存储组件编号及其对应的BomInfo对象
count_dic = {}    # {(父级编号, 文件名): 数量}，用于统计在父级编号下该组件的数量

def traverse_feature(part, bom_dic, count_dic):
    sw_feat = part.FirstFeature
    no_in_currlevel = 1
    no_parent = '0'

    while sw_feat != None:
        # 判断当前特征是否为组件类型
        if sw_feat.GetTypeName == 'Reference' or sw_feat.GetTypeName == 'ReferencePattern':
            curr_comp = sw_feat.GetSpecificFeature2
            # 如果当前组件“不包含于材料明细表中”或者被压缩，则直接跳过
            if curr_comp.ExcludeFromBOM or curr_comp.GetSuppression == 0:
                sw_feat = sw_feat.GetNextFeature
                continue
            # 如果当前组件被轻化，则将其还原
            elif curr_comp.GetSuppression == 1 or curr_comp.GetSuppression == 3 or curr_comp.GetSuppression == 4:
                curr_comp.SetSuppression2(2)
            model_doc = curr_comp.GetModelDoc2
            file_name = model_doc.GetTitle
            self.progress_info.emit(time.strftime('%Y-%m-%d %H:%M:%S  ', time.localtime()) + '读取组件：' + file_name)
            if (no_parent, file_name) in count_dic:
                count_dic[(no_parent, file_name)] += 1
            else:
                count_dic[(no_parent, file_name)] = 1
                # 获取模型文档的自定义属性，并存储于BomInfo()对象中
                bi = BomInfo()
                bi.dwg_no = model_doc.GetCustomInfoValue('', '代号')
                bi.dwg_name = model_doc.GetCustomInfoValue('', '名称')
                bi.material = model_doc.GetCustomInfoValue('', '材料')
                bi.designer = model_doc.GetCustomInfoValue('', '设计')
                bi.drawer = model_doc.GetCustomInfoValue('', '绘图')
                bi.date_design = model_doc.GetCustomInfoValue('', '日期')
                bi.remark = model_doc.GetCustomInfoValue('', '备注')
                bi.file_name = file_name
                bi.file_path = model_doc.GetPathName
                bom_dic[str(no_in_currlevel)] = bi
                # 如果当前组件为装配体，且非外购件及标准件，则调用 traverse_comp() 遍历子装配体
                if model_doc.GetType == constants.swDocASSEMBLY and bi.remark != '外购件' and bi.remark != '标准件':
                    traverse_comp(curr_comp, no_in_currlevel)
                no_in_currlevel += 1
            sw_feat = sw_feat.GetNextFeature
            continue
        else:
            sw_feat = sw_feat.GetNextFeature
            continue

def traverse_comp(component, no_in_parent_level):
    sw_feat = component.FirstFeature
    no_in_currlevel = 1
    no_parent = str(no_in_parent_level)

    while sw_feat != None:
        if sw_feat.GetTypeName == 'Reference' or sw_feat.GetTypeName == 'ReferencePattern':
            curr_comp = sw_feat.GetSpecificFeature2
            if curr_comp.ExcludeFromBOM or curr_comp.GetSuppression == 0:
                sw_feat = sw_feat.GetNextFeature
                continue
            elif curr_comp.GetSuppression == 1 or curr_comp.GetSuppression == 3 or curr_comp.GetSuppression == 4:
                curr_comp.SetSuppression2(2)
            model_doc = curr_comp.GetModelDoc2
            file_name = model_doc.GetTitle
            if (no_parent, file_name) in count_dic:
                count_dic[(no_parent, file_name)] += 1
            else:
                count_dic[(no_parent, file_name)] = 1
                bi = BomInfo()
                bi.dwg_no = model_doc.GetCustomInfoValue('', '代号')
                bi.dwg_name = model_doc.GetCustomInfoValue('', '名称')
                bi.material = model_doc.GetCustomInfoValue('', '材料')
                bi.designer = model_doc.GetCustomInfoValue('', '设计')
                bi.drawer = model_doc.GetCustomInfoValue('', '绘图')
                bi.date_design = model_doc.GetCustomInfoValue('', '日期')
                bi.remark = model_doc.GetCustomInfoValue('', '备注')
                bi.file_name = file_name
                bi.file_path = model_doc.GetPathName
                bom_dic[str(no_parent) + '-' + str(no_in_currlevel)] = bi
                # 如果当前组件为装配体，且非外购件及标准件，则进行迭代
                if model_doc.GetType == constants.swDocASSEMBLY and bi.remark != '外购件' and bi.remark != '标准件':
                    traverse_comp(curr_comp, str(no_parent) + '-' + str(no_in_currlevel))
                no_in_currlevel += 1
            sw_feat = sw_feat.GetNextFeature
            continue
        else:
            sw_feat = sw_feat.GetNextFeature
            continue
```

## 获取零部件的缩略图

BOM 中通常只包含文字信息而不具备预览零部件模型的功能，这导致了在成本估算、坯料预投、物料清点等工作环节中经常需要将 BOM 对照 CAD 模型逐个查找，甚至于“望文猜图”，给相关工作带来了诸多不便。

因此，有必要获取零部件的缩略图并将其加入 BOM 表格中。

虽然 `SolidWorks API` 提供了 `SaveBMP()` 函数用于保存模型的位图，但该方法获取的缩略图色彩失真严重、清晰度低，无法满足需求。因此通过将模型另存为 `png` 文件的方法获取其缩略图。核心代码如下：

```python
def save_preview_png(png_dir):
    for key in bom_dic:
        model_name = bom_dic[key].file_name
        model_path = bom_dic[key].file_path

        if model_path[-7:] == '.SLDASM' or model_path[-7:] == '.sldasm':
            sw_app.OpenDocSilent(model_path, 2, win32com.client.VARIANT(pythoncom.VT_BYREF | pythoncom.VT_I4, 3))
        elif model_path[-7:] == '.SLDPRT' or model_path[-7:] == '.sldprt':
            sw_app.OpenDocSilent(model_path, 1, win32com.client.VARIANT(pythoncom.VT_BYREF | pythoncom.VT_I4, 3))
        part = sw_app.ActiveDoc
        # 调整为等轴测视图，并缩放至适应窗口的大小
        part.ShowNamedView2('*等轴测', 7)
        part.ViewZoomtofit2()
        part.SaveAs3(r'{}\\'.format(png_dir) + model_name[:-7] + '.PNG', 0, 0)
        sw_app.CloseDoc(part.GetTitle)
```

值得一提的是，为了获得最好的效果，需要适当调整 SolidWorks 软件窗口的长宽比例，并在 `系统选项 - TIF/PSD/JPG/PNG` 中勾选 `移除背景` 和 `屏幕捕获` 。

## 将 BOM 信息写入 Excel 表格

使用 `openpyxl` 模块将 BOM 信息写入了 `.xlsx` 文件，并根据零部件的信息分别赋予了其对应的样式。

当用户选择保存缩略图时，自动导出的 `Excel` 表格效果图如下：

![bom_creator_excel_with_png](http://raynnmc.github.io/img/bom_creator_excel_with_png.jpeg)

当用户选择不保存缩略图时，自动导出的 `Excel` 表格效果图如下：

![bom_creator_excel_no_png](http://raynnmc.github.io/img/bom_creator_excel_no_png.jpeg)

以上两图均未作任何手动处理。

模块核心代码如下：

```python
from openpyxl import Workbook
from openpyxl.styles import colors, Color, Font, PatternFill, Border, Alignment, Side
from openpyxl.drawing.image import Image


def xlsx_write(bom_dic, count_dic, set_no, export_dir, png_dir):
    wb = Workbook()
    ws = wb.active
    ws.title = 'BOM'

    col_name = ['层级', '编号', '物料代码', '代号', '名称', '材质/品牌', '预览', '件数', '套数', '总数', '备注', '设计', '绘图', '设计日期']
    ws.row_dimensions[3].height = 20
    for i in range(1, len(col_name) + 1):
        c_t = ws.cell(3, i, col_name[i - 1])
        c_t.font = Font(name='Times New Roman', bold=True, color='FFFFFF')
        c_t.fill = PatternFill(fill_type='solid', start_color='585858')
        c_t.border = Border(left=Side(border_style='thin'), right=Side(border_style='thin'), top=Side(border_style='thin'), bottom=Side(border_style='thin'))
        c_t.alignment = Alignment(horizontal='center', vertical='center')

    def cell_style(key_sent, cell):

        key = str(key_sent)

        def asm_or_not(key, cell):
            global bom_dic
            if bom_dic[key].remark == '装配体':
                cell.font = Font(name='Times New Roman', bold=True)
            else:
                cell.font = Font(name='Times New Roman')

        cell.border = Border(left=Side(border_style='thin'), right=Side(border_style='thin'), top=Side(border_style='thin'), bottom=Side(border_style='thin'))
        cell.alignment = Alignment(horizontal='center', vertical='center')

        if key == '0':
            cell.font = Font(name='Times New Roman', bold=True)
            cell.fill = PatternFill(fill_type='solid', start_color='00BFFF')
        elif key.count('-') == 0:
            cell.fill = PatternFill(fill_type='solid', start_color='DEECB9')
            asm_or_not(key, cell)
        elif key.count('-') == 1:
            cell.fill = PatternFill(fill_type='solid', start_color='F5D0A9')
            asm_or_not(key, cell)
        elif key.count('-') == 2:
            cell.fill = PatternFill(fill_type='solid', start_color='F6E3CE')
            asm_or_not(key, cell)
        elif key.count('-') == 3:
            cell.fill = PatternFill(fill_type='solid', start_color='F8ECE0')
            asm_or_not(key, cell)
        else:
            asm_or_not(key, cell)
            
    curr_row = 4

    for key in bom_dic:
        bi = bom_dic[key]
        if '-' in str(key):
            no_parent = key[::-1].split('-', 1)[1][::-1]
        else:
            no_parent = '0'
        no_in_currlvl = count_dic[(no_parent, bom_dic[key].file_name)]
        if png_dir != '' and os.path.exists(png_dir + '/' + bi.file_name[:-7] + '.PNG'):
            ws.row_dimensions[curr_row].height = 61
        else:
            ws.row_dimensions[curr_row].height = 20
        
        if key == '0':
            c1 = ws.cell(curr_row, 1, 0)
        else:
            c1 = ws.cell(curr_row, 1, key.count('-') + 1)
        c2 = ws.cell(curr_row, 2, str(key))
        c3 = ws.cell(curr_row, 3, '')
        c4 = ws.cell(curr_row, 4, bi.dwg_no)
        c5 = ws.cell(curr_row, 5, bi.dwg_name)
        c6 = ws.cell(curr_row, 6, bi.material)
        
        c7 = ws.cell(curr_row, 7, '')
        if png_dir != '' and os.path.exists(png_dir + '/' + bi.file_name[:-7] + '.PNG'):
            img = Image(png_dir + '/' + bi.file_name[:-7] + '.PNG')
            newsize = (180, 80)
            img.width, img.height = newsize
            ws.add_image(img, 'G' + str(curr_row))

        if key == '0':
            c8 = ws.cell(curr_row, 8, 1)
        else:
            c8 = ws.cell(curr_row, 8, count_dic[(no_parent, bi.file_name)])

        temp = 1

        if '-' in str(key) == False:
            c9 = ws.cell(curr_row, 9, set_no)
            c10 = ws.cell(curr_row, 10, count_dic[(no_parent, bi.file_name)]*set_no)
        else:
            _key = key
            while '-' in str(_key):
                if '-' in no_parent:
                    no_grandparent = no_parent[::-1].split('-', 1)[1][::-1]
                else:
                    no_grandparent = '0'
                temp *= count_dic[(no_grandparent, bom_dic[no_parent].file_name)]
                _key = _key[::-1].split('-', 1)[1][::-1]
                if '-' in str(_key):
                    no_parent = _key[::-1].split('-', 1)[1][::-1]

            set_count = temp*set_no
            total_count = set_count*no_in_currlvl
            c9 = ws.cell(curr_row, 9, set_count)
            c10 = ws.cell(curr_row, 10, total_count)
    
        c11 = ws.cell(curr_row, 11, bi.remark)
        c12 = ws.cell(curr_row, 12, bi.designer)
        c13 = ws.cell(curr_row, 13, bi.drawer)
        c14 = ws.cell(curr_row, 14, bi.date_design)

        cell_list = [c1, c2, c3, c4, c5, c6, c7, c8, c9, c10, c11, c12, c13, c14]
        for i in range(len(cell_list)):
            cell_style(key, cell_list[i])
        
        curr_row += 1
    
    large_width = 23
    medium_width = 14
    ws.column_dimensions['C'].width = large_width
    ws.column_dimensions['D'].width = large_width
    ws.column_dimensions['E'].width = large_width
    ws.column_dimensions['F'].width = large_width
    ws.column_dimensions['G'].width = large_width
    ws.column_dimensions['N'].width = medium_width
    
    if png_dir == '':
        ws.column_dimensions['G'].hidden = True
    
    try:
        wb.save(export_dir)
        self.progress_info.emit('xlsx文档导出完毕!路径为：' + self.export_dir)
    except:
        self.warning_info.emit('Excel写入失败，请检查文件是否被占用！')
```

## 将 BOM 信息写入 XML 文件

使用 `xml` 模块将 BOM 信息写入 `.xml` 文件。文件内容示例如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<bom_info>
    <comp_info>
        <comp_no>0</comp_no>
        <dwg_no>A20121-000</dwg_no>
        <dwg_name>Worm_screw_gear.sldasm</dwg_name>
        <material>装配体</material>
        <designer>Michael</designer>
        <drawer>Jackson</drawer>
        <date_design>2020-02-21</date_design>
        <remark>装配体</remark>
        <count_current_level>1</count_current_level>
        <total_count>20</total_count>
    </comp_info>
    ......
```

模块核心代码如下：

```python
from xml.dom.minidom import Document

def xml_write(bom_dic, count_dic, set_no, export_dir):
    doc = Document()
    bom_info = doc.createElement('bom_info')
    doc.appendChild(bom_info)

    for key in bom_dic:
        bi = bom_dic[key]
        if '-' in str(key):
            no_parent = key[::-1].split('-', 1)[1][::-1]
        else:
            no_parent = '0'

        comp_info = doc.createElement('comp_info')
        bom_info.appendChild(comp_info)

        comp_no = doc.createElement('comp_no')
        comp_no_text = doc.createTextNode(str(key))
        comp_no.appendChild(comp_no_text)
        comp_info.appendChild(comp_no)

        # 略

    try:
        with open(export_dir, 'wb') as f:
            f.write(doc.toprettyxml(indent='\t', encoding='utf-8'))
        self.progress_info.emit('xml文档导出完毕！路径为：' + self.export_dir)
    except:
        self.warning_info.emit('xml文档导出失败，请检查文件是否被占用！')
```

