MR解决topN问题思路
一个MR job可解决
>1、在map端生成key->value对，然后在map端做一个合并操作(在map端做reduce)，生成key->show_cnt，然后分发到reduce端
2、在reduce端对相同的key的show_cnt做加和操作，然后将结果排序，得到topN

如果show_cnt加和的结果依然很大，在单台机器上没办法做全局排序，则再起一个MR job
>1、在map端做排序，每个map取排名前N的记录，分发到reduce端
>2、在reduce端进行全局的排序，取出topN


