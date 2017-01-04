https://dev.mysql.com/doc/refman/5.7/en/mysql-nutshell.html\#mysql-nutshell-deprecations

不再支持4.1之前的旧密码认证模式。mysql\_old\_password认证插件被移除，同时与之相关的选项与函数不再被支持，不支持--skip-secure-auth选项

2、不再支持YEAR\(2\)数据类型。升级最新版本之前转换为YEAR\(4\)类型

3、移除部分参数。innodb\_mirrored\_log\_groups，storage\_engine\(使用新的default\_storage\_engine替代\)，thread\_concurrency，timed\_mutexes，innodb\_use\_sys\_malloc，innodb\_additional\_mem\_pool\_size等

4、alter tables 语句不再支持ignore子句

5、移除insert delay特性，但保留该语法。与之相关的选项或特性被移除，如mysqldump的--delayed-insert选项

6、移除mysql\_upgrade的几个选项--basedir, --datadir, --tmpdir

7、移除SHOW ENGINE INNODB MUTEX，现在可以通过查询performance\_schema中的相关视图获取信息

8、取消对选项前缀的支持，现在需要完整的选项名

9、移除innodb Monitor特性。添加两个动态参数innodb\_status\_output、innodb\_status\_output\_locks，相关信息现在被包含在INFORMATION\_SCHEMA

10、移除一些工具。msql2mysql, mysql\_convert\_table\_format, mysql\_find\_rows, mysql\_fix\_extensions, mysql\_setpermission, mysql\_waitpid, mysql\_zap, mysqlaccess,  mysqlbug, mysqlhotcopy

