+++
date = '2025-08-06T16:56:12+08:00'
draft = false
title = '删除Prometheus中的旧数据'
tags = ['prometheus']
+++

## DCGM Exporter: 清除某Hostname下数据

是的，我又忘记给container设置hostname了。

看看现在都有什么序列：

```bash
curl http://localhost:9400/metrics > metrics.txt
```

删除非"hn"开头的数据：

```bash
http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_GPU_TEMP{Hostname!~"hn.."}
```

删除hostname为"37526f1fd2fd"的数据：

```bash
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_DEC_UTIL{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_ENC_UTIL{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_FB_FREE{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_FB_USED{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_GPU_TEMP{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_GPU_TEMP{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_GPU_UTIL{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_MEMORY_TEMP{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_MEM_CLOCK{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_MEM_COPY_UTIL{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_PCIE_REPLAY_COUNTER{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_POWER_USAGE{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_SM_CLOCK{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_VGPU_LICENSE_STATUS{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_DEV_XID_ERRORS{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_PROF_DRAM_ACTIVE{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_PROF_GR_ENGINE_ACTIVE{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_PROF_PCIE_RX_BYTES{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_PROF_PCIE_TX_BYTES{Hostname="37526f1fd2fd"}'
curl -X POST -g 'http://127.1:9090/api/v1/admin/tsdb/delete_series?match[]=DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{Hostname="37526f1fd2fd"}'
```

