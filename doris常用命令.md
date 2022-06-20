
`改表字段长度`
>ALTER TABLE db.tb MODIFY COLUMN col varchar(8192) NULL COMMENT ""


`删除字段`
>ALTER TABLE db.tb DROP COLUMN col


`添加字段`
>ALTER TABLE db.tb
  ADD COLUMN `col` int(11) DEFAULT '0' comment '' AFTER `other_col`;

`doris 新增指标字段`
>ALTER TABLE table1 ADD COLUMN uv BIGINT SUM DEFAULT '0' after pv;
ALTER TABLE db.tb ADD COLUMN `col` bitmap BITMAP_UNION after other_col;


`查看alter命令进度`
>SHOW ALTER table column


`查看分区`
>SHOW PARTITIONS FROM db.tb

`drop分区`
>alter table db.tb DROP PARTITION p20220208

