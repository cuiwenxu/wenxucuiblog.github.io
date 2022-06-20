`cat query-hive-18553.csv |awk '{if (substr($1,1,3)=="tmp") print $1}'>tmp_list`

