你的任务是读取并深度理解一篇 .docx 格式的论文。

用户的指令是：$ARGUMENTS

## 执行步骤

### 第一步：解析文件

使用以下 Python 脚本读取用户指定的 .docx 文件。如果用户没有给出完整路径，先用 Glob 工具搜索 .docx 文件供用户确认。

将以下脚本保存为 `/tmp/read_docx.py` 并用 `python3 /tmp/read_docx.py <文件路径>` 执行：

```python
import docx
from docx.oxml.ns import qn
from docx.shared import Pt, Cm, Emu
import sys
import os
import zipfile
import json
from lxml import etree

# ========== OMML 公式解析器 ==========

OMML_NS = "http://schemas.openxmlformats.org/officeDocument/2006/math"
W_NS = "http://schemas.openxmlformats.org/wordprocessingml/2006/main"

def omml_to_latex(element):
    """递归将 OMML 元素转换为 LaTeX 表示"""
    tag = etree.QName(element.tag).localname if '}' in element.tag else element.tag
    result = ""

    if tag == "oMath" or tag == "oMathPara":
        parts = []
        for child in element:
            parts.append(omml_to_latex(child))
        result = "".join(parts)

    elif tag == "r":  # math run - 包含实际文本
        texts = element.findall(f"{{{OMML_NS}}}t")
        for t in texts:
            if t.text:
                result += t.text
        # 也检查 w:t
        wtexts = element.findall(f"{{{W_NS}}}t")
        for t in wtexts:
            if t.text:
                result += t.text

    elif tag == "f":  # 分数
        num = element.find(f"{{{OMML_NS}}}num")
        den = element.find(f"{{{OMML_NS}}}den")
        num_text = omml_to_latex(num) if num is not None else "?"
        den_text = omml_to_latex(den) if den is not None else "?"
        result = f"\\frac{{{num_text}}}{{{den_text}}}"

    elif tag == "sSup":  # 上标
        base = element.find(f"{{{OMML_NS}}}e")
        sup = element.find(f"{{{OMML_NS}}}sup")
        base_text = omml_to_latex(base) if base is not None else ""
        sup_text = omml_to_latex(sup) if sup is not None else ""
        result = f"{base_text}^{{{sup_text}}}"

    elif tag == "sSub":  # 下标
        base = element.find(f"{{{OMML_NS}}}e")
        sub = element.find(f"{{{OMML_NS}}}sub")
        base_text = omml_to_latex(base) if base is not None else ""
        sub_text = omml_to_latex(sub) if sub is not None else ""
        result = f"{base_text}_{{{sub_text}}}"

    elif tag == "sSubSup":  # 同时上下标
        base = element.find(f"{{{OMML_NS}}}e")
        sub = element.find(f"{{{OMML_NS}}}sub")
        sup = element.find(f"{{{OMML_NS}}}sup")
        base_text = omml_to_latex(base) if base is not None else ""
        sub_text = omml_to_latex(sub) if sub is not None else ""
        sup_text = omml_to_latex(sup) if sup is not None else ""
        result = f"{base_text}_{{{sub_text}}}^{{{sup_text}}}"

    elif tag == "rad":  # 根号
        deg = element.find(f"{{{OMML_NS}}}deg")
        e = element.find(f"{{{OMML_NS}}}e")
        e_text = omml_to_latex(e) if e is not None else ""
        deg_text = omml_to_latex(deg) if deg is not None else ""
        if deg_text and deg_text.strip():
            result = f"\\sqrt[{deg_text}]{{{e_text}}}"
        else:
            result = f"\\sqrt{{{e_text}}}"

    elif tag == "nary":  # 求和、积分等 n 元运算符
        nary_pr = element.find(f"{{{OMML_NS}}}naryPr")
        char = "∑"
        if nary_pr is not None:
            chr_elem = nary_pr.find(f"{{{OMML_NS}}}chr")
            if chr_elem is not None:
                char = chr_elem.get(f"{{{OMML_NS}}}val", "∑")

        sub = element.find(f"{{{OMML_NS}}}sub")
        sup = element.find(f"{{{OMML_NS}}}sup")
        e = element.find(f"{{{OMML_NS}}}e")

        char_map = {"∑": "\\sum", "∏": "\\prod", "∫": "\\int",
                     "∬": "\\iint", "∮": "\\oint", "⋃": "\\bigcup",
                     "⋂": "\\bigcap"}
        latex_char = char_map.get(char, char)

        sub_text = omml_to_latex(sub) if sub is not None else ""
        sup_text = omml_to_latex(sup) if sup is not None else ""
        e_text = omml_to_latex(e) if e is not None else ""

        result = f"{latex_char}"
        if sub_text.strip():
            result += f"_{{{sub_text}}}"
        if sup_text.strip():
            result += f"^{{{sup_text}}}"
        result += f" {e_text}"

    elif tag == "d":  # 定界符（括号）
        d_pr = element.find(f"{{{OMML_NS}}}dPr")
        beg_chr = "("
        end_chr = ")"
        if d_pr is not None:
            bc = d_pr.find(f"{{{OMML_NS}}}begChr")
            ec = d_pr.find(f"{{{OMML_NS}}}endChr")
            if bc is not None:
                beg_chr = bc.get(f"{{{OMML_NS}}}val", "(")
            if ec is not None:
                end_chr = ec.get(f"{{{OMML_NS}}}val", ")")

        parts = []
        for e in element.findall(f"{{{OMML_NS}}}e"):
            parts.append(omml_to_latex(e))
        result = f"{beg_chr}{', '.join(parts)}{end_chr}"

    elif tag == "m":  # 矩阵
        rows = []
        for mr in element.findall(f"{{{OMML_NS}}}mr"):
            cols = []
            for e in mr.findall(f"{{{OMML_NS}}}e"):
                cols.append(omml_to_latex(e))
            rows.append(" & ".join(cols))
        result = "\\begin{matrix} " + " \\\\ ".join(rows) + " \\end{matrix}"

    elif tag == "acc":  # 重音符号（hat, tilde, bar 等）
        acc_pr = element.find(f"{{{OMML_NS}}}accPr")
        char = "̂"
        if acc_pr is not None:
            chr_elem = acc_pr.find(f"{{{OMML_NS}}}chr")
            if chr_elem is not None:
                char = chr_elem.get(f"{{{OMML_NS}}}val", "̂")
        e = element.find(f"{{{OMML_NS}}}e")
        e_text = omml_to_latex(e) if e is not None else ""
        acc_map = {"̂": "\\hat", "̃": "\\tilde", "̄": "\\bar",
                    "→": "\\vec", "̇": "\\dot", "̈": "\\ddot"}
        latex_acc = acc_map.get(char, f"\\accent{{{char}}}")
        result = f"{latex_acc}{{{e_text}}}"

    elif tag == "bar":  # 上划线/下划线
        e = element.find(f"{{{OMML_NS}}}e")
        e_text = omml_to_latex(e) if e is not None else ""
        result = f"\\overline{{{e_text}}}"

    elif tag == "limLow":  # 下极限
        e = element.find(f"{{{OMML_NS}}}e")
        lim = element.find(f"{{{OMML_NS}}}lim")
        e_text = omml_to_latex(e) if e is not None else ""
        lim_text = omml_to_latex(lim) if lim is not None else ""
        result = f"{e_text}_{{{lim_text}}}"

    elif tag == "limUpp":  # 上极限
        e = element.find(f"{{{OMML_NS}}}e")
        lim = element.find(f"{{{OMML_NS}}}lim")
        e_text = omml_to_latex(e) if e is not None else ""
        lim_text = omml_to_latex(lim) if lim is not None else ""
        result = f"{e_text}^{{{lim_text}}}"

    elif tag == "func":  # 函数（sin, cos, log 等）
        fname = element.find(f"{{{OMML_NS}}}fName")
        e = element.find(f"{{{OMML_NS}}}e")
        fname_text = omml_to_latex(fname) if fname is not None else ""
        e_text = omml_to_latex(e) if e is not None else ""
        result = f"{fname_text}({e_text})"

    elif tag == "eqArr":  # 方程组
        parts = []
        for e in element.findall(f"{{{OMML_NS}}}e"):
            parts.append(omml_to_latex(e))
        result = "\\begin{aligned} " + " \\\\ ".join(parts) + " \\end{aligned}"

    elif tag == "box" or tag == "borderBox":
        e = element.find(f"{{{OMML_NS}}}e")
        result = omml_to_latex(e) if e is not None else ""

    elif tag in ("num", "den", "e", "sub", "sup", "deg", "lim", "fName"):
        parts = []
        for child in element:
            parts.append(omml_to_latex(child))
        result = "".join(parts)

    else:
        # 递归处理未知元素
        parts = []
        for child in element:
            parts.append(omml_to_latex(child))
        if parts:
            result = "".join(parts)

    return result


def extract_formulas_from_para(para_element):
    """从段落 XML 中提取所有公式"""
    formulas = []
    # 查找 oMathPara（独立公式行）和 oMath（内联公式）
    for math_para in para_element.findall(f".//{{{OMML_NS}}}oMathPara"):
        latex = omml_to_latex(math_para)
        if latex.strip():
            formulas.append(("display", latex.strip()))
    for math_elem in para_element.findall(f".//{{{OMML_NS}}}oMath"):
        # 跳过已经在 oMathPara 中处理过的
        parent = math_elem.getparent()
        if parent is not None and etree.QName(parent.tag).localname == "oMathPara":
            continue
        latex = omml_to_latex(math_elem)
        if latex.strip():
            formulas.append(("inline", latex.strip()))
    return formulas


# ========== 格式信息提取 ==========

def emu_to_cm(emu):
    """EMU 转厘米"""
    if emu is None:
        return None
    return round(int(emu) / 914400 * 2.54, 2)

def emu_to_pt(emu):
    """EMU 转磅"""
    if emu is None:
        return None
    return round(int(emu) / 12700, 1)

def half_pt_to_pt(val):
    """半磅转磅"""
    if val is None:
        return None
    return round(int(val) / 2, 1)

def extract_page_layout(doc):
    """提取页面布局信息"""
    layout = {}
    for section in doc.sections:
        layout["页面宽度(cm)"] = round(section.page_width.cm, 2) if section.page_width else None
        layout["页面高度(cm)"] = round(section.page_height.cm, 2) if section.page_height else None
        layout["上边距(cm)"] = round(section.top_margin.cm, 2) if section.top_margin else None
        layout["下边距(cm)"] = round(section.bottom_margin.cm, 2) if section.bottom_margin else None
        layout["左边距(cm)"] = round(section.left_margin.cm, 2) if section.left_margin else None
        layout["右边距(cm)"] = round(section.right_margin.cm, 2) if section.right_margin else None
        layout["页眉距离(cm)"] = round(section.header_distance.cm, 2) if section.header_distance else None
        layout["页脚距离(cm)"] = round(section.footer_distance.cm, 2) if section.footer_distance else None
        # 只取第一个 section
        break
    return layout

def extract_para_format(para):
    """提取段落格式信息"""
    fmt = {}
    pf = para.paragraph_format

    # 对齐方式
    align_map = {0: "左对齐", 1: "居中", 2: "右对齐", 3: "两端对齐", 4: "分散对齐"}
    if pf.alignment is not None:
        fmt["对齐"] = align_map.get(pf.alignment, str(pf.alignment))

    # 行距
    if pf.line_spacing is not None:
        if pf.line_spacing_rule is not None and pf.line_spacing_rule == 4:  # MULTIPLE
            fmt["行距"] = f"{pf.line_spacing}倍"
        elif isinstance(pf.line_spacing, (int, float)) and pf.line_spacing > 50:
            fmt["行距"] = f"{pf.line_spacing}磅(固定值)"
        else:
            fmt["行距"] = f"{pf.line_spacing}倍"

    # 首行缩进
    if pf.first_line_indent is not None:
        fmt["首行缩进(cm)"] = round(pf.first_line_indent.cm, 2)

    # 段前段后间距
    if pf.space_before is not None:
        fmt["段前间距(磅)"] = round(pf.space_before.pt, 1)
    if pf.space_after is not None:
        fmt["段后间距(磅)"] = round(pf.space_after.pt, 1)

    return fmt

def extract_run_format(run):
    """提取文字格式信息"""
    fmt = {}
    if run.font.name:
        fmt["西文字体"] = run.font.name
    if run.font.size:
        fmt["字号(磅)"] = round(run.font.size.pt, 1)

    # 中文字体
    rPr = run._element.find(qn('w:rPr'))
    if rPr is not None:
        rFonts = rPr.find(qn('w:rFonts'))
        if rFonts is not None:
            eastAsia = rFonts.get(qn('w:eastAsia'))
            if eastAsia:
                fmt["中文字体"] = eastAsia

    if run.bold:
        fmt["加粗"] = True
    if run.italic:
        fmt["斜体"] = True
    if run.underline:
        fmt["下划线"] = True
    if run.font.color and run.font.color.rgb:
        fmt["颜色"] = str(run.font.color.rgb)
    return fmt


# ========== 主函数 ==========

def read_docx(filepath):
    doc = docx.Document(filepath)
    output_dir = "/tmp/docx_images"
    os.makedirs(output_dir, exist_ok=True)

    print(f"=== 文件信息 ===")
    print(f"路径: {os.path.abspath(filepath)}")

    props = doc.core_properties
    if props.title:
        print(f"标题: {props.title}")
    if props.author:
        print(f"作者: {props.author}")
    if props.created:
        print(f"创建时间: {props.created}")
    if props.modified:
        print(f"修改时间: {props.modified}")

    # ========== 页面布局 ==========
    print(f"\n=== 页面布局 ===\n")
    layout = extract_page_layout(doc)
    for k, v in layout.items():
        print(f"  {k}: {v}")

    # ========== 提取图片 ==========
    print(f"\n=== 图片提取 ===\n")
    image_paths = []
    with zipfile.ZipFile(filepath, 'r') as z:
        media_files = [f for f in z.namelist() if f.startswith('word/media/')]
        if media_files:
            for img_file in media_files:
                img_data = z.read(img_file)
                ext = os.path.splitext(img_file)[1]
                img_name = os.path.basename(img_file)
                out_path = os.path.join(output_dir, img_name)
                with open(out_path, 'wb') as f:
                    f.write(img_data)
                image_paths.append(out_path)
                print(f"  [图片] {img_name} -> {out_path}")
            print(f"\n  共提取 {len(image_paths)} 张图片")
        else:
            print("  文档中未发现图片。")

    # ========== 格式样式汇总 ==========
    print(f"\n=== 格式样式汇总 ===\n")
    style_formats = {}  # style_name -> {font_info}
    for para in doc.paragraphs:
        style_name = para.style.name if para.style else "Normal"
        if style_name not in style_formats and para.text.strip():
            para_fmt = extract_para_format(para)
            run_fmt = {}
            if para.runs:
                run_fmt = extract_run_format(para.runs[0])
            combined = {**para_fmt, **run_fmt}
            if combined:
                style_formats[style_name] = combined

    for style_name, fmt in sorted(style_formats.items()):
        print(f"  [{style_name}]")
        for k, v in fmt.items():
            print(f"    {k}: {v}")
        print()

    # ========== 按文档顺序读取内容 ==========
    print(f"\n=== 正文内容（含公式） ===\n")

    body = doc.element.body
    paragraphs = doc.paragraphs
    tables = doc.tables
    para_idx = 0
    table_idx = 0
    formula_count = 0

    def get_inline_images(para):
        images = []
        for run in para.runs:
            drawing_elements = run._element.findall(qn('w:drawing'))
            for drawing in drawing_elements:
                blips = drawing.findall('.//' + qn('a:blip'))
                for blip in blips:
                    embed = blip.get(qn('r:embed'))
                    if embed:
                        images.append(embed)
        return images

    for child in body:
        if child.tag == qn('w:p'):
            if para_idx < len(paragraphs):
                para = paragraphs[para_idx]
                para_idx += 1

                style = para.style.name if para.style else "Normal"
                text = para.text.strip()

                # 提取公式
                formulas = extract_formulas_from_para(child)

                # 检查内嵌图片
                inline_imgs = get_inline_images(para)
                if inline_imgs:
                    print(f"\n[📊 此处有图片，见提取的图片文件]")

                # 如果有独立公式
                if formulas:
                    for f_type, f_latex in formulas:
                        formula_count += 1
                        if f_type == "display":
                            print(f"\n[公式 {formula_count}] $$ {f_latex} $$")
                        else:
                            # 内联公式嵌入文本中
                            if text:
                                print(f"{text}  [内联公式: $ {f_latex} $]")
                                text = ""  # 已输出
                            else:
                                print(f"[内联公式: $ {f_latex} $]")

                if not text:
                    if not formulas:
                        pass
                    continue

                # 标题
                if "Heading" in style or "heading" in style or "标题" in style:
                    level = ""
                    for ch in style:
                        if ch.isdigit():
                            level = ch
                            break
                    if level:
                        print(f"\n{'#' * int(level)} {text}")
                    else:
                        print(f"\n# {text}")
                elif "Caption" in style or "题注" in style:
                    print(f"[图表标题] {text}")
                elif "TOC" in style or "目录" in style:
                    print(f"[目录] {text}")
                else:
                    print(text)

        elif child.tag == qn('w:tbl'):
            if table_idx < len(tables):
                table = tables[table_idx]
                table_idx += 1

                print(f"\n--- 表格 ---")
                rows_data = []
                max_cols = 0
                for row in table.rows:
                    row_cells = []
                    for cell in row.cells:
                        cell_text = cell.text.strip().replace('\n', ' ')
                        if row_cells and row_cells[-1] == cell_text:
                            continue
                        row_cells.append(cell_text)
                    max_cols = max(max_cols, len(row_cells))
                    rows_data.append(row_cells)

                if rows_data:
                    for row in rows_data:
                        while len(row) < max_cols:
                            row.append("")
                    print("| " + " | ".join(rows_data[0]) + " |")
                    print("| " + " | ".join(["---"] * max_cols) + " |")
                    for row in rows_data[1:]:
                        print("| " + " | ".join(row) + " |")
                print()

    # ========== 页眉页脚 ==========
    print(f"\n=== 页眉页脚 ===\n")
    for i, section in enumerate(doc.sections):
        header = section.header
        footer = section.footer
        if header and header.paragraphs:
            h_text = " ".join([p.text for p in header.paragraphs if p.text.strip()])
            if h_text:
                print(f"  页眉: {h_text}")
        if footer and footer.paragraphs:
            f_text = " ".join([p.text for p in footer.paragraphs if p.text.strip()])
            if f_text:
                print(f"  页脚: {f_text}")
    if not any(s.header.paragraphs for s in doc.sections if s.header):
        print("  未设置页眉页脚")

    # ========== 脚注 ==========
    footnotes_part = None
    for rel in doc.part.rels.values():
        if "footnotes" in rel.reltype:
            footnotes_part = rel.target_part

    if footnotes_part:
        print(f"\n=== 脚注 ===\n")
        fn_root = footnotes_part._element
        for fn in fn_root.findall(qn('w:footnote')):
            fn_id = fn.get(qn('w:id'))
            if fn_id in ('-1', '0'):
                continue
            texts = [t.text for t in fn.findall('.//' + qn('w:t')) if t.text]
            if texts:
                print(f"  [{fn_id}] {''.join(texts)}")

    # ========== 统计信息 ==========
    all_text = " ".join([p.text for p in doc.paragraphs])
    char_count = len(all_text.replace(" ", ""))
    cn_chars = sum(1 for c in all_text if '\u4e00' <= c <= '\u9fff')
    en_words = len([w for w in all_text.split() if w.isascii() and w.isalpha()])

    print(f"\n=== 统计信息 ===")
    print(f"  总段落数: {len(doc.paragraphs)}")
    print(f"  总表格数: {len(doc.tables)}")
    print(f"  总图片数: {len(image_paths)}")
    print(f"  总公式数: {formula_count}")
    print(f"  总字符数（不含空格）: {char_count}")
    print(f"  中文字符数: {cn_chars}")
    print(f"  英文单词数: {en_words}")

    if image_paths:
        print(f"\n=== 图片文件清单（可用 Read 工具查看） ===")
        for p in image_paths:
            print(f"  {p}")

filepath = sys.argv[1]
read_docx(filepath)
```

### 第二步：查看图片

脚本会将 .docx 中的所有图片提取到 `/tmp/docx_images/` 目录。**你必须使用 Read 工具逐一查看每张图片**，理解图片内容（架构图、实验结果图、流程图等），这对论文理解至关重要。

查看图片时注意：
- 识别图片类型：模型架构图、实验曲线、混淆矩阵、注意力可视化、t-SNE图 等
- 理解图中的文字标注、坐标轴含义、图例
- 将图片内容与正文中的引用对应起来

### 第三步：深度理解

读取完所有文字内容、公式和图片后，输出一份**论文理解报告**：

1. **基本信息**：标题、作者、总字数、章节结构概览

2. **研究概要**：
   - 研究问题是什么？
   - 研究动机和背景是什么？
   - 提出了什么方法？核心创新点是什么？
   - 使用了什么数据集和实验设置？
   - 主要实验结果和结论是什么？

3. **章节结构梳理**：
   - 列出所有章节和子章节标题
   - 每个章节的核心内容（1-2 句话概括）
   - 章节间的逻辑关系

4. **关键要素提取**：
   - 核心方法/模型的名称和架构
   - 使用的数据集
   - 对比的基线方法
   - 关键实验结果（数值）
   - 提到的局限性和未来工作

5. **公式解读**：
   - 列出所有关键公式及其含义
   - 公式编号是否连续
   - 公式中的符号是否在正文中有定义说明
   - 公式推导逻辑是否清晰

6. **图表解读**：
   - 逐一描述每张图和每个表的内容及作用
   - 图表是否清晰、专业、信息量充足
   - 图表与正文的对应关系是否合理

7. **排版格式评估**：
   - 页面布局是否合理（页边距、纸张大小）
   - 各级标题的字体字号是否规范统一
   - 正文字体字号是否符合要求
   - 行距、段间距是否合理
   - 格式一致性是否存在问题

8. **论文脉络图**：
   用文字描述论文的逻辑脉络：
   问题 → 动机 → 方法 → 实验 → 结论

## 注意事项

- 如果文件路径不存在或格式错误，给出明确的错误提示
- 如果论文内容很长，确保完整读取，不要截断
- **必须查看所有提取的图片**，不能跳过图片只读文字
- **必须解读所有提取的公式**，检查 LaTeX 转换是否合理
- 理解报告要忠实于原文，不要添加原文没有的信息
- 如果论文是英文的，理解报告用中文输出
- 读取完成后提醒用户可以使用 /review-paper 进行评审
