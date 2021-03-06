---
layout: post
title: Java使用poi将office文件转为html
category: ['Java']
tags: ['Java']
author: 陈鑫
email: chenxin6@asiainfo.com
description: Java使用poi将office文件转为html
---

#Java使用poi将office文件转为html
##一、前言
功能需求：上传office文档，并提供文件在线预览。

解决方案：
- 使用Aspose.cells.jar包，将文档转换为pdf格式；
- 使用libreOffice，将文档转换为pdf格式；
- 使用poi将文档转换为html格式。

方案一，通过Aspose的方式，该功能是付费版，需要破解，所以是能抛弃。
方案二，使用libreOffice,需要安装使用libreOffice，linux还需要装unoconv，需要使用commons-io的pom依赖，之前maven官方库查询不到这个pom依赖所以放弃了这个方案，刚才准备查询资料时发现这个依赖已经可以使用，估计是前段时间maven官方库出现问题。
方案三，只需要添加所需的依赖就可以使用，但是转换出的html会有一些格式问题，等下会再下面讲到。

## 二、添加依赖
    <dependencies>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.12</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-scratchpad</artifactId>
            <version>3.12</version>
        </dependency>
        <dependency>
            <groupId>fr.opensagres.xdocreport</groupId>
            <artifactId>fr.opensagres.xdocreport.document</artifactId>
            <version>1.0.5</version>
        </dependency>
        <dependency>
            <groupId>fr.opensagres.xdocreport</groupId>
            <artifactId>org.apache.poi.xwpf.converter.xhtml</artifactId>
            <version>1.0.4</version>
        </dependency>
        <dependency>
            <groupId>fr.opensagres.xdocreport</groupId>
            <artifactId>fr.opensagres.xdocreport.core</artifactId>
            <version>2.0.1</version>
        </dependency>
    </dependencies>
    
## 三、word文档转html
一般word文件后缀有doc、docx两种。docx是word2007以及以后版本文档的扩展名，doc是word2003文档保存的扩展名。对于这两种格式的word转换成html需要使用不同的方法。

1、转换问题：
    转换后，对于2003版本word,自动生成的目录会显示有错误；2003版本和2007版本对于特殊字符都有可能显示不出来，不过问题不是很明显；文章中的图片一定要是常用的图片格式（jpeg,jpg,png等），不然无法显示。

2、2003版本word转换成html（.doc）

    public static boolean word2003ToHtml(Map params) {
  
        logger.debug("***** word2003ToHtml start params:{}", params);
        try {
    
        //图片存放路径
        String fileImg = params.get("fileImg").toString();
      
        //转换html后，html中图片的url前缀
        String viewImgPath = params.get("viewImgPath").toString();
        
        //html文件
        File htmlFile = new File(params.get("htmlFile").toString());
        
        File file = new File(params.get("filePath").toString() + params.get("FILE_NAME").toString());
        
        // 1) 加载word文档生成 HWPFDocument对象  
        InputStream inputStream = new FileInputStream(file);
        HWPFDocument wordDocument = new HWPFDocument(inputStream);
        WordToHtmlConverter wordToHtmlConverter =
            new WordToHtmlConverter(DocumentBuilderFactory.newInstance().newDocumentBuilder().newDocument());
            
        //设置图片存放的位置
        wordToHtmlConverter.setPicturesManager(new PicturesManager() {
            public String savePicture(byte[] content, PictureType pictureType, String suggestedName, float widthInches, float heightInches) {
                File imgPath = new File(fileImg);
                if (!imgPath.exists()) {//图片目录不存在则创建
                imgPath.mkdirs();
             }
            File file = new File(fileImg + suggestedName);
            try {
                OutputStream os = new FileOutputStream(file);
                os.write(content);
                os.close();
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
            //这里可以指定word文档中图片的路径。
            return viewImgPath + "/" + suggestedName;
            }
        });
        
        //解析word文档
        wordToHtmlConverter.processDocument(wordDocument);
        Document htmlDocument = wordToHtmlConverter.getDocument();
     
        OutputStream outputStream = new FileOutputStream(htmlFile);

        DOMSource domSource = new DOMSource(htmlDocument);
        StreamResult streamResult = new StreamResult(outputStream);

        TransformerFactory factory = TransformerFactory.newInstance();
        Transformer serializer = factory.newTransformer();
        serializer.setOutputProperty(OutputKeys.ENCODING, "utf-8");
        serializer.setOutputProperty(OutputKeys.INDENT, "yes");
        serializer.setOutputProperty(OutputKeys.METHOD, "html");

        serializer.transform(domSource, streamResult);
        outputStream.close();
        } catch (Exception e) {
        e.printStackTrace();
        return false;
        }
        return true;
    }
    
3、2007版本word转换成html（.docx）

    public static boolean word2007ToHtml(Map params) throws Exception {
        logger.debug("***** word2007ToHtml start params:{}", params);
        try {
            //转换html后，html中图片的url前缀
            String viewImgPath = params.get("viewImgPath").toString();
            
            //图片存放路径
            String fileImg = params.get("fileImg").toString();
            
            File file = new File(params.get("filePath").toString() + params.get("FILE_NAME").toString());
            
            // 1) 加载word文档生成 XWPFDocument对象
            InputStream inputStream = new FileInputStream(file);
            XWPFDocument document = new XWPFDocument(inputStream);
            
            // 2) 解析 XHTML配置 (URIResolver来设置图片存放的目录)
            XHTMLOptions options = XHTMLOptions.create();
            options.URIResolver(new BasicURIResolver(viewImgPath));
            FileImageExtractor extractor = new FileImageExtractor(new File(fileImg));
            options.setExtractor(extractor);
            
            // 3) 将 XWPFDocument转换成XHTML
            File htmlFile = new File(params.get("htmlFile").toString());
            OutputStream outputStream = new FileOutputStream(htmlFile);
            XHTMLConverter.getInstance().convert(document, outputStream, options);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }
    
## 四、Excel文档转Html

POI中将Excel转换为HTML方法仅能转换HSSFWorkBook类型（即03版xls），故可以先将读取的xlsx文件转换成xls文件再调用该方法统一处理

1、2003版本excel转换html(excel中不可包括图片，因为poi没提供将excel中图片转换的方法)
    
    public static boolean excelToHtml(Map params) {
        try {
            String file = params.get("FILE_NAME").toString();
            String filePath = params.get("filePath").toString() + file;


            InputStream input = new FileInputStream(filePath);

            HSSFWorkbook excelBook = new HSSFWorkbook();

            //判断Excel文件将07+版本转换为03版本
            if (file.endsWith(EXCEL_XLS)) {  //Excel 2003
                excelBook = new HSSFWorkbook(input);
            } else if (file.endsWith(EXCEL_XLSX)) {  // Excel 2007/2010
                ExcelTransFormUtil xls = new ExcelTransFormUtil();
                XSSFWorkbook workbookOld = new XSSFWorkbook(input);
                xls.transformXSSF(workbookOld, excelBook);
            }

            ExcelToHtmlConverter excelToHtmlConverter = new ExcelToHtmlConverter(DocumentBuilderFactory.newInstance().newDocumentBuilder().newDocument());
            
            //去掉Excel头行
            excelToHtmlConverter.setOutputColumnHeaders(false);
            
            //去掉Excel行号
            excelToHtmlConverter.setOutputRowNumbers(false);

            excelToHtmlConverter.processWorkbook(excelBook);

            Document htmlDocument = excelToHtmlConverter.getDocument();

            ByteArrayOutputStream outStream = new ByteArrayOutputStream();
            DOMSource domSource = new DOMSource(htmlDocument);
            StreamResult streamResult = new StreamResult(outStream);
            TransformerFactory tf = TransformerFactory.newInstance();
            Transformer serializer = tf.newTransformer();

            serializer.setOutputProperty(OutputKeys.ENCODING, "UTF-8");
            serializer.setOutputProperty(OutputKeys.INDENT, "yes");
            serializer.setOutputProperty(OutputKeys.METHOD, "html");
            serializer.transform(domSource, streamResult);
            outStream.close();

            String content = new String(outStream.toByteArray());

            FileUtils.writeStringToFile(new File(params.get("htmlFile").toString()), content, "UTF-8");
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
            return true;
    }


2、将2007版本转换成2003版本

    public void transformXSSF(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew) {
        HSSFSheet sheetNew;
        XSSFSheet sheetOld;

        workbookNew.setMissingCellPolicy(workbookOld.getMissingCellPolicy());

        for (int i = 0; i < workbookOld.getNumberOfSheets(); i++) {
            sheetOld = workbookOld.getSheetAt(i);
            sheetNew = workbookNew.getSheet(sheetOld.getSheetName());
            sheetNew = workbookNew.createSheet(sheetOld.getSheetName());
            this.transform(workbookOld, workbookNew, sheetOld, sheetNew);
        }
    }

    private void transform(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew,
                         XSSFSheet sheetOld, HSSFSheet sheetNew) {

        sheetNew.setDisplayFormulas(sheetOld.isDisplayFormulas());
        sheetNew.setDisplayGridlines(sheetOld.isDisplayGridlines());
        sheetNew.setDisplayGuts(sheetOld.getDisplayGuts());
        sheetNew.setDisplayRowColHeadings(sheetOld.isDisplayRowColHeadings());
        sheetNew.setDisplayZeros(sheetOld.isDisplayZeros());
        sheetNew.setFitToPage(sheetOld.getFitToPage());

        sheetNew.setHorizontallyCenter(sheetOld.getHorizontallyCenter());
        sheetNew.setMargin(Sheet.BottomMargin,
            sheetOld.getMargin(Sheet.BottomMargin));
        sheetNew.setMargin(Sheet.FooterMargin,
            sheetOld.getMargin(Sheet.FooterMargin));
        sheetNew.setMargin(Sheet.HeaderMargin,
            sheetOld.getMargin(Sheet.HeaderMargin));
        sheetNew.setMargin(Sheet.LeftMargin,
            sheetOld.getMargin(Sheet.LeftMargin));
        sheetNew.setMargin(Sheet.RightMargin,
            sheetOld.getMargin(Sheet.RightMargin));
        sheetNew.setMargin(Sheet.TopMargin, sheetOld.getMargin(Sheet.TopMargin));
        sheetNew.setPrintGridlines(sheetNew.isPrintGridlines());
        sheetNew.setRightToLeft(sheetNew.isRightToLeft());
        sheetNew.setRowSumsBelow(sheetNew.getRowSumsBelow());
        sheetNew.setRowSumsRight(sheetNew.getRowSumsRight());
        sheetNew.setVerticallyCenter(sheetOld.getVerticallyCenter());

        HSSFRow rowNew;
        for (Row row : sheetOld) {
            rowNew = sheetNew.createRow(row.getRowNum());
            if (rowNew != null)
                this.transform(workbookOld, workbookNew, (XSSFRow) row, rowNew);
        }

        for (int i = 0; i < this.lastColumn; i++) {
            sheetNew.setColumnWidth(i, sheetOld.getColumnWidth(i));
            sheetNew.setColumnHidden(i, sheetOld.isColumnHidden(i));
        }

        for (int i = 0; i < sheetOld.getNumMergedRegions(); i++) {
            CellRangeAddress merged = sheetOld.getMergedRegion(i);
            sheetNew.addMergedRegion(merged);
        }
    }

    private void transform(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew,
                         XSSFRow rowOld, HSSFRow rowNew) {
        HSSFCell cellNew;
        rowNew.setHeight(rowOld.getHeight());

        for (Cell cell : rowOld) {
        cellNew = rowNew.createCell(cell.getColumnIndex(),
            cell.getCellType());
        if (cellNew != null)
            this.transform(workbookOld, workbookNew, (XSSFCell) cell,
                cellNew);
        }
        this.lastColumn = Math.max(this.lastColumn, rowOld.getLastCellNum());
    }

    private void transform(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew,
                         XSSFCell cellOld, HSSFCell cellNew) {
        cellNew.setCellComment(cellOld.getCellComment());

        Integer hash = cellOld.getCellStyle().hashCode();
        if (this.styleMap != null && !this.styleMap.containsKey(hash)) {
        this.transform(workbookOld, workbookNew, hash,
            cellOld.getCellStyle(),
            (HSSFCellStyle) workbookNew.createCellStyle());
        }
        cellNew.setCellStyle(this.styleMap.get(hash));

        switch (cellOld.getCellType()) {
        case Cell.CELL_TYPE_BLANK:
            break;
        case Cell.CELL_TYPE_BOOLEAN:
            cellNew.setCellValue(cellOld.getBooleanCellValue());
            break;
        case Cell.CELL_TYPE_ERROR:
            cellNew.setCellValue(cellOld.getErrorCellValue());
            break;
        case Cell.CELL_TYPE_FORMULA:
            cellNew.setCellValue(cellOld.getCellFormula());
            break;
        case Cell.CELL_TYPE_NUMERIC:
            cellNew.setCellValue(cellOld.getNumericCellValue());
            break;
        case Cell.CELL_TYPE_STRING:
            cellNew.setCellValue(cellOld.getStringCellValue());
            break;
        default:
       
        }
    }

    private void transform(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew,
                         Integer hash, XSSFCellStyle styleOld, HSSFCellStyle styleNew) {
        styleNew.setAlignment(styleOld.getAlignment());
        styleNew.setBorderBottom(styleOld.getBorderBottom());
        styleNew.setBorderLeft(styleOld.getBorderLeft());
        styleNew.setBorderRight(styleOld.getBorderRight());
        styleNew.setBorderTop(styleOld.getBorderTop());
        styleNew.setDataFormat(this.transform(workbookOld, workbookNew,
            styleOld.getDataFormat()));
        styleNew.setFillBackgroundColor(styleOld.getFillBackgroundColor());
        styleNew.setFillForegroundColor(styleOld.getFillForegroundColor());
        styleNew.setFillPattern(styleOld.getFillPattern());
        styleNew.setFont(this.transform(workbookNew,
            (XSSFFont) styleOld.getFont()));
        styleNew.setHidden(styleOld.getHidden());
        styleNew.setIndention(styleOld.getIndention());
        styleNew.setLocked(styleOld.getLocked());
        styleNew.setVerticalAlignment(styleOld.getVerticalAlignment());
        styleNew.setWrapText(styleOld.getWrapText());
        this.styleMap.put(hash, styleNew);
    }

    private short transform(XSSFWorkbook workbookOld, HSSFWorkbook workbookNew,
                          short index) {
        DataFormat formatOld = workbookOld.createDataFormat();
        DataFormat formatNew = workbookNew.createDataFormat();
        return formatNew.getFormat(formatOld.getFormat(index));
    }

    private HSSFFont transform(HSSFWorkbook workbookNew, XSSFFont fontOld) {
        HSSFFont fontNew = workbookNew.createFont();
        fontNew.setBoldweight(fontOld.getBoldweight());
        fontNew.setCharSet(fontOld.getCharSet());
        fontNew.setColor(fontOld.getColor());
        fontNew.setFontName(fontOld.getFontName());
        fontNew.setFontHeight(fontOld.getFontHeight());
        fontNew.setItalic(fontOld.getItalic());
        fontNew.setStrikeout(fontOld.getStrikeout());
        fontNew.setTypeOffset(fontOld.getTypeOffset());
        fontNew.setUnderline(fontOld.getUnderline());
        return fontNew;
    }
手动捂脸，有点太多了。大家凑合着看。。。

## 五、PPT文档转Html 

就是将ppt转换成一张张图片再放入html。

注：
(1)ppt中文字会有中文显示问题，可以将中文文字库加入到服务器中即可。
(2)pptx2007版本文件不能显示表格
   
1、2003版本ppt转html

    public static boolean ppt2003Tohtml(Map params) {
        try {
           String imgPath = params.get("fileImg").toString();
           File file = new File(params.get("filePath").toString() + params.get("FILE_NAME").toString());
           InputStream inputStream = new FileInputStream(file);
           SlideShow ppt = new SlideShow(inputStream);
           inputStream.close();

           Dimension pgsize = ppt.getPageSize();
           org.apache.poi.hslf.model.Slide[] slide = ppt.getSlides();
           FileOutputStream out = null;
           String imghtml = "";
           String viewImgPath = params.get("viewImgPath").toString();
           for (int i = 0; i < slide.length; i++) {
               logger.debug("第" + i + "页。");

               TextRun[] truns = slide[i].getTextRuns();
               for (int k = 0; k < truns.length; k++) {
                   RichTextRun[] rtruns = truns[k].getRichTextRuns();
                   for (int l = 0; l < rtruns.length; l++) {

                   rtruns[l].setFontIndex(1);
                   rtruns[l].setFontName("宋体");
                   }
               }
               BufferedImage img = new BufferedImage(pgsize.width, pgsize.height, BufferedImage.TYPE_INT_RGB);

               Graphics2D graphics = img.createGraphics();
               graphics.setPaint(Color.BLUE);
               graphics.fill(new Rectangle2D.Float(0, 0, pgsize.width, pgsize.height));
               slide[i].draw(graphics);

               // 这里设置图片的存放路径和图片的格式(jpeg,png,bmp等等)
               out = new FileOutputStream(imgPath + (i + 1) + ".jpeg");
               javax.imageio.ImageIO.write(img, "jpeg", out);
                
               //图片在html加载路径
               String imgs = viewImgPath + "/" + (i + 1) + ".jpeg";
               imghtml += "<img src=\'" + imgs + "\' style=\'width:960px;height:530px;vertical-align:text-bottom;\'><br><br><br><br>";
               DOMSource domSource = new DOMSource();
               StreamResult streamResult = new StreamResult(out);
               TransformerFactory tf = TransformerFactory.newInstance();
               Transformer serializer = tf.newTransformer();
               serializer.setOutputProperty(OutputKeys.ENCODING, "utf-8");
               serializer.setOutputProperty(OutputKeys.INDENT, "yes");
               serializer.setOutputProperty(OutputKeys.METHOD, "html");
               serializer.transform(domSource, streamResult);
               out.close();
               String ppthtml = "<html><head><META http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"></head><body>" + imghtml + "</body></html>";
               FileUtils.writeStringToFile(new File(params.get("htmlFile").toString()), ppthtml, "utf-8");
           }
       } catch (Exception e) {
           e.printStackTrace();
           return false;
       }
       return true;
    }
    
2、2007版本ppt转换html

    public static boolean ppt2007Tohtml(Map params) {
        try {
          String imgPath = params.get("fileImg").toString();
          File file = new File(params.get("filePath").toString() + params.get("FILE_NAME").toString());
          InputStream inputStream = new FileInputStream(file);
          XMLSlideShow ppt = new XMLSlideShow(inputStream);
          inputStream.close();
    
          Dimension pgsize = ppt.getPageSize();
    
          XSLFSlide[] pptPageXSLFSLiseList = ppt.getSlides();
          FileOutputStream out = null;
          String imghtml = "";
          String viewImgPath = params.get("viewImgPath").toString();
          for (int i = 0; i < pptPageXSLFSLiseList.length; i++) {
            try {
              for (XSLFShape shape : pptPageXSLFSLiseList[i].getShapes()) {
                if (shape instanceof XSLFTextShape) {
                  XSLFTextShape tsh = (XSLFTextShape) shape;
                  for (XSLFTextParagraph p : tsh) {
                    for (XSLFTextRun r : p) {
                      r.setFontFamily("宋体");
                    }
                  }
                }
              }
              BufferedImage img = new BufferedImage(pgsize.width, pgsize.height, BufferedImage.TYPE_INT_RGB);
              Graphics2D graphics = img.createGraphics();
              // clear the drawing area
              graphics.setPaint(Color.white);
              graphics.fill(new Rectangle2D.Float(0, 0, pgsize.width, pgsize.height));
    
              // render
              pptPageXSLFSLiseList[i].draw(graphics);
    
              //
              String Imgname = imgPath + (i + 1) + ".jpeg";
              out = new FileOutputStream(Imgname);
              javax.imageio.ImageIO.write(img, "jpeg", out);
              //图片在html加载路径
              String imgs = viewImgPath + "/" + (i + 1) + ".jpeg";
              imghtml += "<img src=\'" + imgs + "\' style=\'width:960px;height:530px;vertical-align:text-bottom;\'><br><br><br><br>";
            } catch (Exception e) {
              System.out.println(e);
              System.out.println("第" + i + "张ppt转换出错");
            }
          }
    
    
          DOMSource domSource = new DOMSource();
          StreamResult streamResult = new StreamResult(out);
          TransformerFactory tf = TransformerFactory.newInstance();
          Transformer serializer = tf.newTransformer();
          serializer.setOutputProperty(OutputKeys.ENCODING, "utf-8");
          serializer.setOutputProperty(OutputKeys.INDENT, "yes");
          serializer.setOutputProperty(OutputKeys.METHOD, "html");
          serializer.transform(domSource, streamResult);
          out.close();
          String ppthtml = "<html><head><META http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"></head><body>" + imghtml + "</body></html>";
          FileUtils.writeStringToFile(new File(params.get("htmlFile").toString()), ppthtml, "utf-8");
        } catch (Exception e) {
          e.printStackTrace();
          return false;
        }
        return true;
      }
      
## 六、预览

1、每次预览将源文件转换成html后，将html文件上传nginx服务器目录下，用nginx代理访问，以防出现浏览器缓存问题。
我这里是将html文件和图片路径打包，通过sftp上传nginx服务器，然后解压。
2、使用iframe标签访问html文件。这里需要注意的是预览pdf文件，会出现下载打印的按钮，如果需求是文件只可缓存不可下载，就不可以使用iframe标签预览pdf。
