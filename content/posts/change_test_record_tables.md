---
date: '2026-05-28T13:34:56+08:00'
draft: false
title: '修改docx文档中表格的指定单元格'
tags: ["java","apache poi","office"]
categories: ["gq"]
---

这段代码的主要有3个作用：

1. 将指定的单元格替换为一个xlsx文件中的对应某列值，或者说List中的值
2. 将指定的单元格替换为空行
3. 将指定的单元格替换为一个xlsx文件中的对应某列值，或者说Map中的值

```java
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.apache.poi.xssf.usermodel.XSSFSheet;
import org.apache.poi.xssf.usermodel.XSSFRow;
import org.apache.poi.xssf.usermodel.XSSFCell;
import org.apache.poi.xwpf.usermodel.*;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * @Description 修改docx文档中从第9个表格开始的所有表格的第4行第2列为空行
 * @Time 2026/5/27 10:36
 * @Author Ren Siyu
 */
public class modifyTableCellValue {
    public static void main(String[] args) {
        String filePath = "D:\\AllFiles\\IVCPS\\改测试报告\\IVCPS典型参考系统原型记录-20250525-原始.docx";
        String xlsxPath = "D:\\AllFiles\\IVCPS\\改测试报告\\测试用例编号大全.xlsx";
        String expectedOutcomeXlsxPath = "D:\\AllFiles\\IVCPS\\改测试报告\\场景预期结果.xlsx";
        String outputPath = new SimpleDateFormat("yyyy-MM-dd HH-mm-ss").format(new java.util.Date()) + "output.docx";

        try {
            modifyTableCells(filePath, outputPath, xlsxPath, expectedOutcomeXlsxPath);
            System.out.println("表格修改完成！");
        } catch (IOException e) {
            System.err.println("处理文档时出错: " + e.getMessage());
            e.printStackTrace();
        }
    }

    /**
     * 修改docx文档中从第9个表格开始的所有表格的第4行第2列
     *
     * @param inputPath               输入文件路径
     * @param outputPath              输出文件路径
     * @param xlsxPath                xlsx文件路径（编号）
     * @param expectedOutcomeXlsxPath xlsx文件路径（预期结果）
     * @throws IOException IO异常
     */
    public static void modifyTableCells(String inputPath, String outputPath, String xlsxPath, String expectedOutcomeXlsxPath) throws IOException {
        // 读取xlsx文件的B列值（编号）
        List<String> testIds = readColumnBFromXlsx(xlsxPath);

        // 读取预期结果xlsx文件（A列=用例编号，B列=描述）
        Map<String, String> expectedOutcomeMap = readExpectedOutcomeFromXlsx(expectedOutcomeXlsxPath);

        try (FileInputStream fis = new FileInputStream(inputPath);
             XWPFDocument document = new XWPFDocument(fis)) {

            List<XWPFTable> tables = document.getTables();

            // 从第9个表格开始（索引为8）
            for (int i = 8; i < tables.size(); i++) {
                XWPFTable table = tables.get(i);

                // 填充编号到第1行第1列
                int dataIndex = i - 8; // 对应testIds的索引
                if (dataIndex < testIds.size()) {
                    modifyCellTestId(table, 0, 1, testIds.get(dataIndex));
                }
                // 原有的修改空行逻辑
                modifyCell(table, 3, 1);
                // 原有的修改空行逻辑
                modifyCell(table, 4, 1);
                // 修改预期结果：检查第1行第2列是否以用例编号起始，如果是则替换第2行第2列
                modifyCellExpectedOutcome(table, 1, 1, expectedOutcomeMap);

            }

            try (FileOutputStream fos = new FileOutputStream(outputPath)) {
                document.write(fos);
            }
        }
    }

    /**
     * 修改指定表格单元格的内容为2个空行
     *
     * @param table    表格对象
     * @param rowIndex 行索引（从0开始）
     * @param colIndex 列索引（从0开始）
     */
    private static void modifyCell(XWPFTable table, int rowIndex, int colIndex) {
        List<XWPFTableRow> rows = table.getRows();

        if (rowIndex < rows.size()) {
            XWPFTableRow row = rows.get(rowIndex);
            List<XWPFTableCell> cells = row.getTableCells();

            if (colIndex < cells.size()) {
                XWPFTableCell cell = cells.get(colIndex);

                // 清空段落内容（保留段落格式和边框）
                XWPFParagraph paragraph = cell.getParagraphs().get(0);
                clearParagraphRuns(paragraph);

                // 添加换行
                XWPFRun run = paragraph.createRun();
                run.setFontSize(10.5);
                run.addBreak();

            }
        }
    }

    /**
     * 从xlsx文件读取B列的值（从第2行开始）
     *
     * @param xlsxPath xlsx文件路径
     * @return B列值的列表
     * @throws IOException IO异常
     */
    private static List<String> readColumnBFromXlsx(String xlsxPath) throws IOException {
        List<String> values = new ArrayList<>();

        try (FileInputStream fis = new FileInputStream(xlsxPath);
             XSSFWorkbook workbook = new XSSFWorkbook(fis)) {

            XSSFSheet sheet = workbook.getSheetAt(0); // 获取第一个sheet

            // 从第2行开始遍历（索引1，跳过表头）
            for (int i = 1; i <= sheet.getLastRowNum(); i++) {
                XSSFRow row = sheet.getRow(i);
                if (row != null) {
                    XSSFCell cell = row.getCell(1); // B列索引为1
                    if (cell != null) {
                        // 根据单元格类型获取值
                        switch (cell.getCellType()) {
                            case STRING:
                                values.add(cell.getStringCellValue());
                                break;
                            case NUMERIC:
                                // 如果是数字，转换为整数字符串
                                values.add(String.valueOf((int) cell.getNumericCellValue()));
                                break;
                            default:
                                values.add("");
                        }
                    } else {
                        values.add("");
                    }
                }
            }
        }

        return values;
    }

    /**
     * 修改表格单元格内容为指定值
     *
     * @param table    表格对象
     * @param rowIndex 行索引（从0开始）
     * @param colIndex 列索引（从0开始）
     * @param value    要填入的值
     */
    private static void modifyCellTestId(XWPFTable table, int rowIndex, int colIndex, String value) {
        List<XWPFTableRow> rows = table.getRows();

        if (rowIndex < rows.size()) {
            XWPFTableRow row = rows.get(rowIndex);
            List<XWPFTableCell> cells = row.getTableCells();

            if (colIndex < cells.size()) {
                XWPFTableCell cell = cells.get(colIndex);

                // 清空段落内容（保留段落格式和边框）
                XWPFParagraph paragraph = cell.getParagraphs().get(0);
                clearParagraphRuns(paragraph);

                // 设置新值
                XWPFRun run = paragraph.createRun();
                run.setText(value);
                run.setFontFamily("等线");
            }
        }
    }

    /**
     * 从预期结果xlsx文件读取数据（A列=用例编号，B列=描述，无表头）
     *
     * @param xlsxPath xlsx文件路径
     * @return 用例编号到描述的映射
     * @throws IOException IO异常
     */
    private static Map<String, String> readExpectedOutcomeFromXlsx(String xlsxPath) throws IOException {
        Map<String, String> map = new HashMap<>();

        try (FileInputStream fis = new FileInputStream(xlsxPath);
             XSSFWorkbook workbook = new XSSFWorkbook(fis)) {
            XSSFSheet sheet = workbook.getSheetAt(0); // 获取第一个sheet

            // 遍历所有行（无表头，从第0行开始）
            for (int i = 0; i <= sheet.getLastRowNum(); i++) {
                XSSFRow row = sheet.getRow(i);
                if (row != null) {
                    XSSFCell cellA = row.getCell(0); // A列=用例编号
                    XSSFCell cellB = row.getCell(1); // B列=描述

                    if (cellA != null && cellB != null) {
                        String key = getCellStringValue(cellA);
                        String value = getCellStringValue(cellB);
                        if (!key.isEmpty() && !value.isEmpty()) {
                            map.put(key, value);
                        }
                    }
                }
            }
        }

        return map;
    }

    /**
     * 获取单元格的字符串值
     *
     * @param cell 单元格对象
     * @return 字符串值
     */
    private static String getCellStringValue(XSSFCell cell) {
        switch (cell.getCellType()) {
            case STRING:
                return cell.getStringCellValue();
            case NUMERIC:
                return String.valueOf((int) cell.getNumericCellValue());
            default:
                return "";
        }
    }

    /**
     * 修改预期结果：检查指定单元格是否以用例编号起始，如果是则替换第2行第2列为对应描述
     *
     * @param table              表格对象
     * @param rowIndex           行索引（从0开始）
     * @param colIndex           列索引（从0开始）
     * @param expectedOutcomeMap 用例编号到描述的映射
     */
    private static void modifyCellExpectedOutcome(XWPFTable table, int rowIndex, int colIndex, Map<String, String> expectedOutcomeMap) {
        List<XWPFTableRow> rows = table.getRows();

        // 获取用例编号单元格内容
        XWPFTableRow row = rows.get(0);
        List<XWPFTableCell> cells = row.getTableCells();
        XWPFTableCell cell = cells.get(1);
        String cellValue = getTableCellStringValue(cell);

        // 检查是否以某个用例编号起始
        for (Map.Entry<String, String> entry : expectedOutcomeMap.entrySet()) {
            if (cellValue.startsWith(entry.getKey())) {
                // 找到匹配的用例编号，替换第2行第2列（索引2,1）
                if (rows.size() > 2) {
                    XWPFTableRow targetRow = rows.get(rowIndex);
                    List<XWPFTableCell> targetCells = targetRow.getTableCells();
                    if (targetCells.size() > 1) {
                        XWPFTableCell targetCell = targetCells.get(colIndex);

                        // 清空段落内容（保留段落格式和边框）
                        XWPFParagraph paragraph = targetCell.getParagraphs().get(0);
                        clearParagraphRuns(paragraph);

                        // 填充新内容
                        XWPFRun run = paragraph.createRun();
                        run.setText(entry.getValue());
                        // 设置字体为等线
                        run.setFontFamily("等线");
                        run.setFontSize(10.5);
                    }
                }
                break; // 找到后退出循环
            }
        }
    }


    /**
     * 获取表格单元格的字符串值
     *
     * @param cell 表格单元格对象
     * @return 字符串值
     */
    private static String getTableCellStringValue(XWPFTableCell cell) {
        StringBuilder sb = new StringBuilder();
        for (XWPFParagraph paragraph : cell.getParagraphs()) {
            for (XWPFRun run : paragraph.getRuns()) {
                sb.append(run.getText(0));
            }
        }
        return sb.toString();
    }

    /**
     * 清空段落内容（保留段落格式和边框）
     *
     * @param paragraph 段落对象
     */
    private static void clearParagraphRuns(XWPFParagraph paragraph) {
        while (!paragraph.getRuns().isEmpty()) {
            paragraph.removeRun(0);
        }
    }
}
```

