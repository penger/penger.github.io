
---
layout: post
category: "read"
title:  "根据HTML格式的字符串生成excel"
tags: [python]
---

## 场景
各种不同格式的html(主要是跨行跨列),需要按照原有的格式导出为Excel
## 思路
按照html中的 tr,td,th 作为标识符
1.首先确定 n 行(统计 tr 数量) m 列(统计第一个 tr 中td的数量),遇到 rowspan, colspan 则加上具体的值
2.构造一个 n 行 m 列的二维数组
3.设置全局的 x,y 初始值为 0,0
4.如果根据读取的 col ,row 来累加 x,y 并以此作为下一个单元格的初始值
5.为 数字,表头 等设置单独的样式
##
示例
![install steps ](../img/export_excel/excel.png)
###  
```
package com.bailian.utils;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URLEncoder;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.regex.Pattern;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.poi.hssf.usermodel.HSSFDataFormat;
import org.apache.poi.ss.usermodel.Cell;
import org.apache.poi.ss.usermodel.CellStyle;
import org.apache.poi.ss.usermodel.Font;
import org.apache.poi.ss.usermodel.IndexedColors;
import org.apache.poi.ss.usermodel.Row;
import org.apache.poi.ss.usermodel.Sheet;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.usermodel.XSSFSheet;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.dom4j.Attribute;
import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.DocumentHelper;
import org.dom4j.Element;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.FileCopyUtils;

/**
 * @ClassName: ExportUtil
 * @Description: 导出excel文件工具类
 * @author gongpeng
 * @date 2016-1-22 14:22:45
 */
public class ExportUtil {

	private static final Logger logger = LoggerFactory.getLogger(HttpClientUtils.class);

	private static final String DOT = ".";
	private static final String EXCEL_SUFFIX = "xlsx";
	private static final String XLSX_SUFFIX = DOT + EXCEL_SUFFIX;

	private static final String STYLE_TITLE = "title";
	private static final String STYLE_NUM = "num";
	private static final String STYLE_NUM_WITH_DOT = "numWithDot";
	private static final String STYLE_PERCENT = "percent";

	/**
	 * createTempFile
	 * 
	 * @param content
	 * @return
	 * @throws Exception
	 */
	public String getExcelFileName(String content) throws Exception {
		return getExcelFile(content).getName();
	}

	/**
	 * 
	 * @param content
	 * @return
	 */
	public File getExcelFile(String content) {
		XSSFWorkbook workBook = createWorkBook(content);
		return createTempFile(workBook);
	}

	public void downloadFile(HttpServletRequest request, HttpServletResponse response, String tempFilePath,
			String filename) throws Exception {
		response.setCharacterEncoding("UTF-8");
		response.setContentType("application/Json;charset=UTF-8");
		File file = new File(tempFilePath);
		InputStream is = null;
		if (filename == null || filename.equals("")) {
			filename = "data" + XLSX_SUFFIX;
		}
		if (!filename.contains(EXCEL_SUFFIX)) {
			filename += XLSX_SUFFIX;
		}
		try {
			response.reset();
			response.setContentType("application/vnd.ms-excel");
			response.setHeader("Content-Disposition", "attachment;filename="
					+ URLEncoder.encode(new String(filename.getBytes("ISO8859-1"), "utf-8"), "UTF-8"));
			is = new FileInputStream(file);
			FileCopyUtils.copy(is, response.getOutputStream());
		} catch (IOException e) {
			logger.error(e.getMessage(), e);
		} finally {
			if (is != null) {
				is.close();
			}
			// 删除临时文件
			file.delete();
		}
	}

	/**
	 * create workbook
	 * 
	 * @param content
	 * @return
	 * @throws Exception
	 */
	private XSSFWorkbook createWorkBook(String content) {
		// 格式化内容
		// content="<table id='ta' border='1'><tr><td rowspan='3'>1</td><td
		// colspan='2'>程序员</td><td>天津</td><td>admin@kali.com</td></tr></table>";
		if (!content.startsWith("<table")) {
			content = "<table>" + content + "</table>";
			content = content.replace("\"", "'");
		}
		content = content.replace("<br>", "");

		XSSFWorkbook workbook = new XSSFWorkbook();
		Map<String, CellStyle> styles = createStyles(workbook);
		XSSFSheet sheet = workbook.createSheet();
		try {
			Document doc = DocumentHelper.parseText(content);
			Element rootElement = doc.getRootElement();
			createWorkBook(rootElement, sheet, styles);
		} catch (DocumentException e) {
			logger.error(e.getMessage(), e);
		}
		return workbook;
	}

	private File createTempFile(XSSFWorkbook workBook) {
		String tempFileName = UUID.randomUUID().toString() + XLSX_SUFFIX;
		File tempFile = new File(tempFileName);
		FileOutputStream out = null;
		try {
			out = new FileOutputStream(tempFile);
			workBook.write(out);
		} catch (IOException e) {
			logger.error(e.getMessage(), e);
		} finally {
			try {
				if (out != null) {
					out.close();
				}
			} catch (IOException e) {
				logger.error(e.getMessage(), e);
			}
		}
		return tempFile;
	}

	private void createWorkBook(Element rootElement, Sheet sheet, Map<String, CellStyle> styles) {
		// 创建二维数组
		Boolean[][] booleanArray = getDoubleBooleanArray(rootElement);
		@SuppressWarnings("unchecked")
		List<Element> elements = rootElement.selectNodes("//td|//th");

		for (int i = 0; i < elements.size(); i++) {
			int itemRows = 1;
			int itemCols = 1;
			Element ele = elements.get(i);
			Attribute colAttr = ele.attribute("colspan");
			Attribute rowAttr = ele.attribute("rowspan");
			if (colAttr != null) {
				itemCols = Integer.parseInt(colAttr.getStringValue());
			}
			if (rowAttr != null) {
				itemRows = Integer.parseInt(rowAttr.getStringValue());
			}
			String text = ele.getText();
			//insertBlankGetFirstBlank(sheet, booleanArray, itemRows, itemCols, text, ele, styles);
			insertBlankGetFirstBlank2(sheet, booleanArray, itemRows, itemCols, text, ele, styles,i);
		//	printbooleanArray(booleanArray);
		}
	}

	/**
	 * for debug 
	 * 2016-3-31 14:05:36 gongpeng
	 * @param booleanArray
	 */
	
	@SuppressWarnings("unused")
	private void printbooleanArray(Boolean[][] booleanArray) {
		System.out.println("start");
		for(int i=0;i<booleanArray.length;i++){
			Boolean[] booleans = booleanArray[i];
			for(int j=0;j<booleans.length;j++){
				System.out.print(booleans[j]);
			}
			System.out.println();
		}
		System.out.println("end ========================");
		
	}

	/**
	 * 创建excel模板数组
	 * 
	 * @param rootElement
	 * @return
	 */
	@SuppressWarnings("unchecked")
	private Boolean[][] getDoubleBooleanArray(Element rootElement) {
		int rows = 0;
		int cols = 0;

		List<Element> element = null;
		List<Element> selectNodes = rootElement.selectNodes("//tr");
		if (selectNodes != null) {
			Element trele = selectNodes.get(0);
			if ((element = trele.elements("td")) == null || (trele.elements("td").size() == 0)) {
				element = trele.elements("th");
			}
		}
		for (Element tde : element) {
			Attribute attribute = tde.attribute("colspan");
			if (attribute == null) {
				cols += 1;
			} else {
				cols += Integer.parseInt(attribute.getStringValue());
			}
		}
		rows = selectNodes.size();
		return new Boolean[rows][cols];
	}

	private void insertBlankGetFirstBlank(Sheet sheet, Boolean[][] b, int itemRows, int itemCols, String content,
			Element ele, Map<String, CellStyle> styles) {
		boolean shouldBreak = false;
		for (int i = 0; i < b.length; i++) {
			if (!shouldBreak) {
				for (int j = 0; j < b[i].length; j++) {
					if (b[i][j] == null) {
						// 填充值
						for (int t = 0; t < itemRows; t++) {
							for (int tt = 0; tt < itemCols; tt++) {
								b[i + t][j + tt] = true;
							}
						}
						// 行列起始 终止值
						Row row = null;
						// 定位行
						row = sheet.getRow(i);
						if (row == null) {
							row = sheet.createRow(i);
						}
						// 定位列
						Cell cell = row.createCell(j);
						// 设置单元格格式
						String name = ele.getName();
						if (name.equals("th")) {
							cell.setCellStyle(styles.get(STYLE_TITLE));
							cell.setCellValue(content);
						} else {
							boolean matches = Pattern.matches("(\\d|\\.|,|%|-)+", content);
							if (matches && !content.equals("-")) {
								if (content.contains("%")) {
									cell.setCellStyle(styles.get(STYLE_PERCENT));
									cell.setCellValue(content);
								}else{
									content = content.replaceAll(",", "");
									double doubleContent = Double.parseDouble(content);
									if (content.contains(".")) {
										cell.setCellStyle(styles.get(STYLE_NUM_WITH_DOT));
										cell.setCellValue(doubleContent);
									} else {
										// 编号之类的不以数字处理
										if (content.length() > 4 && !content.contains(",")) {
											cell.setCellValue(content);
										} else {
											cell.setCellStyle(styles.get(STYLE_NUM));
											cell.setCellValue(doubleContent);
										}
										
									}
								}
							} else {
								// 设置值
								cell.setCellValue(content);
							}
						}
						// 合并单元格
						sheet.addMergedRegion(new CellRangeAddress(i, i + itemRows - 1, j, j + itemCols - 1));
						shouldBreak = true;
						break;
					}
				}
			} else {
				break;
			}
		}
	}
	
	
	
	private void insertBlankGetFirstBlank2(Sheet sheet, Boolean[][] b, int itemRows, int itemCols, String content,
			Element ele, Map<String, CellStyle> styles,int index) {
			// 填充值
			//通过index获取行和列的值
			System.out.println(b.length);
			System.out.println(b[0].length);
			int i=(index/b[0].length);
//			int j=((index+1)%b[0].length);
//			if(j==0){
//				j=b[0].length;
//			}
//			j=j-1;
			int j=index%b[0].length;
			System.out.println("index :"+index +" i: "+i +" j: "+j);
			// 行列起始 终止值
			Row row = null;
			// 定位行
			row = sheet.getRow(i);
			if (row == null) {
				row = sheet.createRow(i);
			}
			// 定位列
			Cell cell = row.createCell(j);
			// 设置单元格格式
			String name = ele.getName();
			if (name.equals("th")) {
				cell.setCellStyle(styles.get(STYLE_TITLE));
				cell.setCellValue(content);
			} else {
				boolean matches = Pattern.matches("(\\d|\\.|,|%|-)+", content);
				if (matches && !content.equals("-")) {
					if (content.contains("%")) {
						cell.setCellStyle(styles.get(STYLE_PERCENT));
						cell.setCellValue(content);
					}else{
						content = content.replaceAll(",", "");
						double doubleContent = Double.parseDouble(content);
						if (content.contains(".")) {
							cell.setCellStyle(styles.get(STYLE_NUM_WITH_DOT));
							cell.setCellValue(doubleContent);
						} else {
							// 编号之类的不以数字处理
							if (content.length() > 4 && !content.contains(",")) {
								cell.setCellValue(content);
							} else {
								cell.setCellStyle(styles.get(STYLE_NUM));
								cell.setCellValue(doubleContent);
							}
							
						}
					}
				} else {
					cell.setCellValue(content);
				}
		}
	}

	/**
	 * 创建表格样式
	 * 
	 * @param wb
	 *            工作薄对象
	 * @return 样式列表
	 */
	private Map<String, CellStyle> createStyles(Workbook wb) {
		Map<String, CellStyle> styles = new HashMap<String, CellStyle>();

		CellStyle style = wb.createCellStyle();
		CellStyle numStyle = wb.createCellStyle();
		CellStyle numWithDotStyle = wb.createCellStyle();
		CellStyle percentStyle = wb.createCellStyle();
		numWithDotStyle.setDataFormat(HSSFDataFormat.getBuiltinFormat("#,##0.00"));
		numStyle.setDataFormat(HSSFDataFormat.getBuiltinFormat("#,##0"));
		percentStyle.setDataFormat(HSSFDataFormat.getBuiltinFormat("0.00%"));
		styles.put(STYLE_NUM, numStyle);
		styles.put(STYLE_NUM_WITH_DOT, numWithDotStyle);
		styles.put(STYLE_PERCENT, percentStyle);

		style.setAlignment(CellStyle.ALIGN_CENTER);
		style.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
		Font titleFont = wb.createFont();
		// titleFont.setFontName("Arial");
		// titleFont.setFontHeightInPoints((short) 10);
		titleFont.setBoldweight(Font.BOLDWEIGHT_BOLD);
		style.setFont(titleFont);
		styles.put(STYLE_TITLE, style);

		style = wb.createCellStyle();
		style.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
		style.setBorderRight(CellStyle.BORDER_THIN);
		style.setRightBorderColor(IndexedColors.GREY_50_PERCENT.getIndex());
		style.setBorderLeft(CellStyle.BORDER_THIN);
		style.setLeftBorderColor(IndexedColors.GREY_50_PERCENT.getIndex());
		style.setBorderTop(CellStyle.BORDER_THIN);
		style.setTopBorderColor(IndexedColors.GREY_50_PERCENT.getIndex());
		style.setBorderBottom(CellStyle.BORDER_THIN);
		style.setBottomBorderColor(IndexedColors.GREY_50_PERCENT.getIndex());
		Font dataFont = wb.createFont();
		dataFont.setFontName("Arial");
		dataFont.setFontHeightInPoints((short) 10);
		style.setFont(dataFont);
		styles.put("data", style);

		style = wb.createCellStyle();
		style.cloneStyleFrom(styles.get("data"));
		style.setAlignment(CellStyle.ALIGN_LEFT);
		styles.put("data1", style);

		style = wb.createCellStyle();
		style.cloneStyleFrom(styles.get("data"));
		style.setAlignment(CellStyle.ALIGN_CENTER);
		styles.put("data2", style);

		style = wb.createCellStyle();
		style.cloneStyleFrom(styles.get("data"));
		style.setAlignment(CellStyle.ALIGN_RIGHT);
		styles.put("data3", style);

		style = wb.createCellStyle();
		style.cloneStyleFrom(styles.get("data"));
		// style.setWrapText(true);
		style.setAlignment(CellStyle.ALIGN_CENTER);
		style.setFillForegroundColor(IndexedColors.GREY_50_PERCENT.getIndex());
		style.setFillPattern(CellStyle.SOLID_FOREGROUND);
		Font headerFont = wb.createFont();
		headerFont.setFontName("Arial");
		headerFont.setFontHeightInPoints((short) 10);
		headerFont.setBoldweight(Font.BOLDWEIGHT_BOLD);
		headerFont.setColor(IndexedColors.WHITE.getIndex());
		style.setFont(headerFont);
		styles.put("header", style);

		return styles;
	}

	public static void main(String[] args) throws Exception {
		ExportUtil exportUtil = new ExportUtil();
		 String content="<table id='ta' border='1'><tr><td rowspan='3'>1</td><td colspan='2'>程序员</td><td>天津</td><td>admin@kali.com</td></tr><tr><td>23</td><td	 rowspan='2'>测试员</td><td>北京</td><td rowspan='2'>guest@kali.com</td></tr><tr><td>23</td><td	 rowspan='2'>北京</td></tr><tr><td>1</td><td>23</td><td>程序员</td><td>admin@kali.com</td></tr></table>";
//		String content = "<table id='ta' border='1'><thead><tr><th >1</th><th >程序员</th><th >程序员</th><th>天津</th></tr></thead><tbody><tr><td>23,456,343.00</td><td>23,456,343.00</td><td>北京</td><td >guest@kali.com</td></tr></tbody></table>";
		XSSFWorkbook workbook = exportUtil.createWorkBook(content);
		File file = new File("d://test.xlsx");
		if (file.exists()) {
			System.out.println("已经存在删除");
			file.delete();
		}
		FileOutputStream fileOut = new FileOutputStream(file);
		workbook.write(fileOut);
		fileOut.close();
	}

}

```
