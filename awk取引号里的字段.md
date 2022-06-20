双引号：`awk -F "[\"]" '{print $2}'`
单引号：`awk -F "[\']" '{print $2}'`
比如文件aaa.txt格式如下
```
'ZHDC_foshan_hyc_20200628_clusterid_10007',
 'ZHDC_foshan_hyc_20200628_clusterid_10008',
 'ZHDC_foshan_hyc_20200628_clusterid_10009',
 'ZHDC_foshan_hyc_20200628_clusterid_10010',
 'ZHDC_foshan_hyc_20200628_clusterid_10012',
 'ZHDC_foshan_hyc_20200628_clusterid_10013',
 'ZHDC_foshan_hyc_20200628_clusterid_10014',
 'ZHDC_foshan_hyc_20200628_clusterid_10016',
```
要取两个单引号之间的字符
`cat aaa.txt|awk -F "[\']" '{print $2}'`



