# RISC-V 指令集手册 - 中文

## 手动构建 PDF

> 因为 PDF 不方便版本管理，所以未将其添加，需要安装[pandoc](https://github.com/jgm/pandoc)并手动构建。

### Linux

```bash
mkdir build
cd src
pandoc -f  markdown-auto_identifiers --pdf-engine=xelatex   --template=../templates/mppl.tex -s --listings ./*.md -o ../build/RISC-V指令集手册-中文
```

### Windows

```bash
md build
cd src
pandoc.exe -f  markdown-auto_identifiers --pdf-engine=xelatex   --template=../templates/mppl.tex -s --listings 1-Introduction.md 2-Overview.md  -o ../build/RISC-V指令集手册-中文
```
