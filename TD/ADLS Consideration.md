- Catalog name: `demoworkspace`
- External locations:
	- name: bronze_location
		- url: abfss://bronze@storageaccountdemoapp.dfs.core.windows.net/
	- name: demoworkspace
		- url: abfss://unity-catalog-storage@dbstoragezqobcco2cprsy.dfs.core.windows.net/1166586406547797
- Schema: `bronze`
- 

----------------- 
  
  
  - 
  - ==write into Delta tables using JAva spark API== to work with Databricks delta
		  -  in next stage we will add the bi-temporality 
  - ==Logging==
		  - DLQ 
		  - Retry 
  - ==Auth== 
		  - absorb all details in side the enterprise-kafka-starter, just ask them for basics like clientid, secret, clusterPoolId 
  - ==OpenTelemetrySDK ==
	- ==push the logs down with traceId to dynatrace ==and splunk 