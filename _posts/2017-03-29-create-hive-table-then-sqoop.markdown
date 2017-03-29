---
layout: post
category: "read"
title:  "create hive table from oracle then sqoop"
tags: [spark,hive]
---

## 需求
通过jdbc的getmetadata方法得到表结构建立相应的表,之前
两个文件 CreateHiveTableSQL.java  create_hive_table.sh
CreateHiveTableSQL.java
```
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileWriter;
import java.sql.*;


//create at 2016-12-1 15:40:45
//根据指定的数据库连接地址和schema名称得到 hive 的建表语句
//gp

public class CreateHiveTableSQL {

    private static Connection connection=null;

    private static final String url_18="jdbc:oracle:thin:@10.111.111.11:1521:report";
    private static final String username_18="username";
    private static final String password_18="password";
    private static final String schema_18="schema";




    //默认获取18的链接
    public static Connection getConnection(){
        try {
            Class.forName("oracle.jdbc.driver.OracleDriver");
            connection = DriverManager.getConnection(url_18,username_18,password_18);
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return connection;
    }


    public static Connection getConnection(String url,String username,String password){
        try {
            Class.forName("oracle.jdbc.driver.OracleDriver");
            connection = DriverManager.getConnection(url,username,password);
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return connection;
    }

    public static String getSchema() throws Exception{
        String schema = getConnection().getMetaData().getUserName();
        System.out.println(schema);
        return schema.toUpperCase();
    }

    public static void getMetaData() throws SQLException {
        DatabaseMetaData metaData = getConnection().getMetaData();
        ResultSet tables = metaData.getTables(null, "IDMDATA", "%", new String[]{"TABLE"});
        while (tables.next()){
            String table_name = tables.getString("TABLE_NAME");
            System.out.println(table_name);
            if("M_UEC_POWER_DATA_POWER_B".equals(table_name)) {
                ResultSet columns =
                        getConnection().getMetaData().getColumns(null, "IDMDATA", "M_UEC_POWER_DATA_POWER_B", null);
                while (columns.next()){
                    String column_name = columns.getString("COLUMN_NAME");
                    String type_name = columns.getString("TYPE_NAME");
                    String column_size = columns.getString("COLUMN_SIZE");
                    System.out.println(column_name+"-----------------------------"+type_name+"             "+column_size);
                }
            }
        }
    }

    public static String getTableMetaDate2HiveStatement(Connection conn,String schema,String tableName,String systemID) throws SQLException{
        StringBuffer sql_buff= new StringBuffer();
        sql_buff.append("create table `sourcedata.");
	sql_buff.append(systemID).append("_");
        sql_buff.append(tableName.toUpperCase());
        sql_buff.append("`(").append("\n");

        DatabaseMetaData metaData = conn.getMetaData();
//        DatabaseMetaData metaData = getConnection().getMetaData();
        ResultSet columns = metaData.getColumns("null", schema.toUpperCase(), tableName.toUpperCase(), null);
        int count=0;
        while (columns.next()){
            String table_name = columns.getString("TABLE_NAME");
            //由于是模式匹配,如果表名不一样,那么跳过
            if(!table_name.equalsIgnoreCase(tableName)){
                continue;
            }
            String column_name = columns.getString("COLUMN_NAME");
            String type_name = columns.getString("TYPE_NAME");
            String data_type = columns.getString("DATA_TYPE");
            String column_size = columns.getString("COLUMN_SIZE");
            int decimal_digits = columns.getInt("DECIMAL_DIGITS");
            int num_prec_radix = columns.getInt("NUM_PREC_RADIX");
            String remark = columns.getString("REMARKS");

            System.out.println(column_name+" "+type_name+" "+data_type+" "+column_size+" "+remark+"         "+decimal_digits+","+num_prec_radix);
            if(count!=0){
                sql_buff.append(", \n");
            }else{
                sql_buff.append("\n");
            }
            count++;

            if(type_name.equalsIgnoreCase("varchar2") || type_name.equalsIgnoreCase("nvarchar2")){
                sql_buff.append("\t").append(column_name).append("\t").append("string");
            }else if(type_name.equalsIgnoreCase("number")){
                if(decimal_digits==0){
		    if(Integer.parseInt(column_size) > 9){
                    	sql_buff.append("\t").append(column_name).append("\t").append("bigint ");
		    }else{
                    	sql_buff.append("\t").append(column_name).append("\t").append("int ");
		    }
                }else{
                    //如果有小数点,那么转换为double类型
                    sql_buff.append("\t").append(column_name).append("\t").append("double");
                }
            }else if(type_name.equalsIgnoreCase("long")){
                sql_buff.append("\t").append(column_name).append("\t").append("bigint ");
            }else if(type_name.equalsIgnoreCase("date")){
                sql_buff.append("\t").append(column_name).append("\t").append("timestamp ");
            }else if(type_name.toUpperCase().contains("TIMESTAMP")){
                sql_buff.append("\t").append(column_name).append("\t").append("timestamp ");
            }else if(type_name.equalsIgnoreCase("char")){
                sql_buff.append("\t").append(column_name).append("\t").append("string");
            }else if(type_name.equalsIgnoreCase("clob")){
                sql_buff.append("\t").append(column_name).append("\t").append("string");
            }
            else{
                sql_buff.append("\t").append(column_name).append("\t").append("未识别类型");
                throw new RuntimeException("有需要特殊处理的数据类型"+type_name );
            }
        }
        sql_buff.append("\n) partitioned by (`dt` string )\n");

        return sql_buff.toString();

    }

    public static void main(String[] args ) throws Exception {
        if(args.length!=6){
            System.out.println("需要五个参数 url, username ,password ,schema ,table_name, system_id ");
            System.out.println("demoinput: jdbc:oracle:thin:@10.201.48.18:1521:report idmdata bigdata915 idmdata test_1 s01");

            System.exit(0);
        }
        String url=args[0];
        String username=args[1];
        String password=args[2];
        String schema=args[3];
        String table_name=args[4];
	String system_id=args[5];

        System.out.println("-----start_----------");
        String hive_sql = getTableMetaDate2HiveStatement(getConnection(url,username,password), schema, table_name,system_id);
//        String hive_sql = getTableMetaDate2HiveStatement(getConnection(), "IDMDATA", "M_UEC_POWER_DATA_POWER_B");
        System.out.println("-----end_----------");
        System.out.println(hive_sql);
        File file = new File("1.txt");
        //如果存在那么删除
        if(file.exists()){
            file.delete();
        }
        file.createNewFile();
        FileWriter fileWriter = new FileWriter(file, true);
        BufferedWriter bw = new BufferedWriter(fileWriter);
        bw.write(hive_sql);
        bw.flush();
        bw.close();
        fileWriter.close();
    }
}

```
create_hive_table.sh
```
#!/bin/bash
#echo "jdbc:oracle:thin:@10.201.1.1:1521:promodg querydata bigdata MEMBER UMBRELLA_STORE_STOCK"
#生成的 建表语句位于 1.txt  ' sudo -u hive hive -f 1.txt ' 执行建表
home=$(cd "`dirname "\$0"`" ; pwd)
log_file=$home/create.log
time=`date +"%F %T"`
echo $time $@ >> $log_file

#echo "jdbc:oracle:thin:@10.201.1.1:1521:promodg querydata bigdata campaign flash_store"

java -cp .:ojdbc6-11.2.0.1.0.jar CreateHiveTableSQL $@

cat 1.txt >> 2.txt
schema=$(echo $4 | tr '[a-z]' '[A-Z]')
table_name=$(echo $5 | tr '[a-z]' '[A-Z]')

echo "--connect $1 --username $2 --password $3 --verbose --table $schema.$table_name --split-by TC_ID --target-dir /user/hive/output/$6_$5 --hive-import --hive-overwrite --hive-database sourcedata --hive-table $6_$5 --hive-partition-value \${task_date} --hive-partition-key dt"

```
后续会写一些针对sqoop的操作