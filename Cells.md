# Cells ����(����)
���濪ʼ��Cells���ֿ�ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

Cells���ܴ��ϸ�������˵�м���������ȽϹؼ�����4����������չ��5��
-----------------------------------
* rep bbs��auctions�����Ҫ����ͨ�ţ�ͬʱ����executor������������garden�Ĵ���
* garden ����ǰ��
* executor ִ����
* garden-linux-backend �������
* garden-windows(��չ)

### rep executor�������
�ȿ���������������

		/var/vcap/packages/rep/bin/rep ${etcd_sec_flags} \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17008 \
		-listenAddr=0.0.0.0:1800 \
		-preloadedRootFS cflinuxfs2:/var/vcap/packages/rootfs_cflinuxfs2/rootfs \
		-rootFSProvider docker \
		-cellID=cell_z1-0 \
		-zone=z1 \
		-pollingInterval=30s \
		-evacuationPollingInterval=10s \
		-evacuationTimeout=60s \
		-skipCertVerify=true \
		-gardenNetwork=tcp \
		-gardenAddr=127.0.0.1:7777 \
		-memoryMB=auto \
		-diskMB=auto \
		-containerInodeLimit=200000 \
		-containerMaxCpuShares=1024 \
		-cachePath=$CACHE_DIR \
		-maxCacheSizeInBytes=10000000000\
		-exportNetworkEnvVars=true\
		-healthyMonitoringInterval=30s \
		-unhealthyMonitoringInterval=0.5s \
		-createWorkPoolSize=32 \
		-deleteWorkPoolSize=32 \
		-readWorkPoolSize=64 \
		-metricsWorkPoolSize=8 \
		-healthCheckWorkPoolSize=64 \
		-tempDir=$TMP_DIR \
		-logLevel=debug
		
* rep��
��Ϊ����cells�Ĵ��ţ�rep������˺ܹؼ������ã�</br>
1.����������cell������������BBS�����ͨ�ţ���Ҫ��ȷ��BBS�����Tasks��actuallLrps�������е�ͬ����Ȼ�������֣������Converger������Ѿ��������ˣ���Ҫ������ʵ��Ǩ��</br>
2.����auctions��Tasks��LRPs������</br>
3.�ڲ���ѯ��Executor�����ͨ����������tasks��lrps���ܹ������Ĵ�����������������������������</br>

* Executor��
��������һ������������ִ�е�ʵ�֣�����������Tasks��LRPs���������Ὣcell�е���־ת����</br>

�ص��ȿ�һ��rep:��������һ��Ѳ��� Ҳ�Ǻ�����Ҫ����ʵ��Ӧ�û������е������ط������������Ǻ������Ƚ���ص�����</br>
https://github.com/cloudfoundry-incubator/rep/blob/6676dde8901ba3c483602e48ff8f88058f285f76/cmd/rep/executor.go</br>

		func executorConfig() executorinit.Configuration {
			return executorinit.Configuration{
				GardenNetwork:               *gardenNetwork, //tcp
				GardenAddr:                  *gardenAddr,//garden�˵ĵ�ַ 127.0.0.1:7777
				ContainerOwnerName:          *containerOwnerName,//������ӵ����
				TempDir:                     *tempDir,
				CachePath:                   *cachePath,
				MaxCacheSizeInBytes:         *maxCacheSizeInBytes,//10000000000
				SkipCertVerify:              *skipCertVerify, //true
				ExportNetworkEnvVars:        *exportNetworkEnvVars, //�Ƿ���Ҫ���绷������ true
				ContainerMaxCpuShares:       *containerMaxCpuShares, //1024
				ContainerInodeLimit:         *containerInodeLimit, //inode����Ŀ200000
				HealthyMonitoringInterval:   *healthyMonitoringInterval, //�������30s
				UnhealthyMonitoringInterval: *unhealthyMonitoringInterval,
				HealthCheckWorkPoolSize:     *healthCheckWorkPoolSize,
				CreateWorkPoolSize:          *createWorkPoolSize,
				DeleteWorkPoolSize:          *deleteWorkPoolSize,
				ReadWorkPoolSize:            *readWorkPoolSize,
				MetricsWorkPoolSize:         *metricsWorkPoolSize,
				RegistryPruningInterval:     *registryPruningInterval,
				MemoryMB:                    *memoryMBFlag, //ʣ��������������auto
				DiskMB:                      *diskMBFlag,
			}
		}

����������rep��������һЩ����
https://github.com/cloudfoundry-incubator/rep/blob/6676dde8901ba3c483602e48ff8f88058f285f76/cmd/rep/main.go</br>

��û������������֮ǰ������֮ǰ��������ÿ��tasks��LRPs������Ӧ��auctionѡ��cells��ִ��</br>
���뵽auction_cell_rep���֣�</br>

		type AuctionCellRep struct {
			cellID               string
			stackPathMap         rep.StackPathMap
			rootFSProviders      auctiontypes.RootFSProviders
			stack                string
			zone                 string
			generateInstanceGuid func() (string, error)
			bbs                  Bbs.RepBBS
			client               executor.Client
			evacuationReporter   evacuation_context.EvacuationReporter
			logger               lager.Logger
		}
		
������cellID��stack��rootFSproviders��zone����һЩ��Ҫ�õĿͻ��˱���bbs��executor.Client</br>

		func rootFSProviders(preloaded rep.StackPathMap, arbitrary []string) auctiontypes.RootFSProviders {
			rootFSProviders := auctiontypes.RootFSProviders{}
			for _, scheme := range arbitrary {
				rootFSProviders[scheme] = auctiontypes.ArbitraryRootFSProvider{}
			}

			stacks := make([]string, 0, len(preloaded))
			for stack, _ := range preloaded {
				stacks = append(stacks, stack)
			}
			rootFSProviders["preloaded"] = auctiontypes.NewFixedSetRootFSProvider(stacks...)

			return rootFSProviders
		}

��һ����ʵ�����Ѿ����úõģ�</br>

		-preloadedRootFS cflinuxfs2:/var/vcap/packages/rootfs_cflinuxfs2/rootfs
		-rootFSProvider docker

ִ��һ��cf stacks��</br>

		cflinuxfs2      Cloud Foundry Linux-based filesystem   
		windows2012R2   Windows Server 2012 R2
		
��Щ����preloadedRootFS����Ԥ���ڲ����ʱ��Ԥװ��rootfs</br>

--->Ȼ�����func (a *AuctionCellRep) Perform(work auctiontypes.Work) (auctiontypes.Work, error) ����</br>
* �ȿ�LRPs���֣�</br>

		if len(work.LRPs) > 0 {
			lrpLogger := logger.Session("lrp-allocate-instances")

			containers, lrpAuctionMap, untranslatedLRPs := a.lrpsToContainers(work.LRPs)
			
			//lrpsToContainers(work.LRPs)�������Ի�ȡ�������ã�auction���ϣ�δ�����LRPs
			
			if len(untranslatedLRPs) > 0 {
				lrpLogger.Info("failed-to-translate-lrps-to-containers", lager.Data{"num-failed-to-translate": len(untranslatedLRPs)})
				failedWork.LRPs = untranslatedLRPs
			}

			lrpLogger.Info("requesting-container-allocation", lager.Data{"num-requesting-allocation": len(containers)})
			
			//���￪ʼ��������
			errMessageMap, err := a.client.AllocateContainers(containers)
			
			if err != nil {
				lrpLogger.Error("failed-requesting-container-allocation", err)
				failedWork.LRPs = work.LRPs
			} else {
				lrpLogger.Info("succeeded-requesting-container-allocation", lager.Data{"num-failed-to-allocate": len(errMessageMap)})
				for guid, lrpStart := range lrpAuctionMap {
					if _, found := errMessageMap[guid]; found {
						failedWork.LRPs = append(failedWork.LRPs, lrpStart)
					}
				}
			}
		}

�������������rpsToContainers</br>
��������������е�LRPs,Ȼ�����lrp����Ϣ�����ÿһ����������Դ����Ϣ</br>

		containerGuid := rep.LRPContainerGuid(lrpStart.DesiredLRP.ProcessGuid, instanceGuid)
		lrpAuctionMap[containerGuid] = lrpStart

���忴һ�����container����ι���ģ�</br>

		container := executor.Container{
			Guid: containerGuid,

			Tags: executor.Tags{
				rep.LifecycleTag:    rep.LRPLifecycle,
				rep.DomainTag:       lrpStart.DesiredLRP.Domain,
				rep.ProcessGuidTag:  lrpStart.DesiredLRP.ProcessGuid,
				rep.InstanceGuidTag: instanceGuid,
				rep.ProcessIndexTag: strconv.Itoa(lrpStart.Index),
			},

			MemoryMB:     int(lrpStart.DesiredLRP.MemoryMb),
			DiskMB:       int(lrpStart.DesiredLRP.DiskMb),
			DiskScope:    diskScope,
			CPUWeight:    uint(lrpStart.DesiredLRP.CpuWeight),
			RootFSPath:   rootFSPath,
			Privileged:   lrpStart.DesiredLRP.Privileged,
			Ports:        a.convertPortMappings(lrpStart.DesiredLRP.Ports),
			StartTimeout: uint(lrpStart.DesiredLRP.StartTimeout),

			LogConfig: executor.LogConfig{
				Guid:       lrpStart.DesiredLRP.LogGuid,
				Index:      lrpStart.Index,
				SourceName: lrpStart.DesiredLRP.LogSource,
			},

			MetricsConfig: executor.MetricsConfig{
				Guid:  lrpStart.DesiredLRP.MetricsGuid,
				Index: lrpStart.Index,
			},

			Setup:   lrpStart.DesiredLRP.Setup,
			Action:  lrpStart.DesiredLRP.Action,
			Monitor: lrpStart.DesiredLRP.Monitor,

			Env: append([]executor.EnvironmentVariable{
				{Name: "INSTANCE_GUID", Value: instanceGuid},
				{Name: "INSTANCE_INDEX", Value: strconv.Itoa(lrpStart.Index)},
				{Name: "CF_INSTANCE_GUID", Value: instanceGuid},
				{Name: "CF_INSTANCE_INDEX", Value: strconv.Itoa(lrpStart.Index)},
			}, executor.EnvironmentVariablesFromModel(lrpStart.DesiredLRP.EnvironmentVariables)...),
			EgressRules: lrpStart.DesiredLRP.EgressRules,
		}

��ʵ����DesiredLRPs�����һЩ�嵥,��Ϊִ������rep��������صģ��������ǿ��Կ���������ô������������ģ�</br>
���������������</br>

		errMessageMap, err := a.client.AllocateContainers(containers)

* ֱ������https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/depot.go#L110</br>

		//��������ǰ��Ԥ������
		for _, executorContainer := range executorContainers {
			if executorContainer.CPUWeight > 100 || executorContainer.CPUWeight < 0 {
				logger.Debug("invalid-cpu-weight", lager.Data{
					"guid":      executorContainer.Guid,
					"cpuweight": executorContainer.CPUWeight,
				})
				errMessageMap[executorContainer.Guid] = executor.ErrLimitsInvalid.Error()
				continue
			} else if executorContainer.CPUWeight == 0 {
				//�����0��������Ȩ��100
				executorContainer.CPUWeight = 100
			}

			if executorContainer.Guid == "" {
				logger.Debug("empty-guid")
				errMessageMap[executorContainer.Guid] = executor.ErrGuidNotSpecified.Error()
				continue
			}

			eligibleContainers = append(eligibleContainers, executorContainer)
		}
		
��һ���Ƕ�container���������cpu���ȼ�Ȩֵ�����жϣ��ܺ����</br>

Ȼ�󿴵������ȼ���һ����Դ����</br>

		c.resourcesLock.Lock()

Ȼ��Կɷ������������������Դ��</br>

		for _, allocatableContainer := range allocatableContainers {
			if _, err := c.allocationStore.Allocate(logger, allocatableContainer); err != nil {
				logger.Debug(
					"failed-to-allocate-container",
					lager.Data{
						"guid":  allocatableContainer.Guid,
						"error": err.Error(),
					},
				)
				errMessageMap[allocatableContainer.Guid] = err.Error()
			}
		}
		
���ǿ��Լ����������������c.allocationStore.Allocate(logger, allocatableContainer)</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/allocationstore/allocationstore.go#L49</br>

		func (a *AllocationStore) Allocate(logger lager.Logger, container executor.Container) (executor.Container, error) {
			a.lock.Lock()
			defer a.lock.Unlock()

			if _, err := a.lookup(container.Guid); err == nil {
				logger.Error("failed-allocating-container", err)
				return executor.Container{}, executor.ErrContainerGuidNotAvailable
			}
			logger.Debug("allocating-container", lager.Data{"container": container})

			container.State = executor.StateReserved
			container.AllocatedAt = a.clock.Now().UnixNano()
			a.allocated[container.Guid] = container

			a.eventEmitter.Emit(executor.NewContainerReservedEvent(container))

			return container, nil
		}

���������꣬��������һ�ݴ����嵥��</br>

		{
		  "timestamp": "1441087134.999691010", 
		  "source": "rep", 
		  "message": "rep.depot-client.allocate-containers.allocating-container", 
		  "log_level": 0, 
		  "data": {
			"container": {
			  "guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-b4dcdbe9337046cc91410d126d4cf861", 
			  "state": "", 
			  "privileged": true, 
			  "memory_mb": 1024, 
			  "disk_mb": 6144, 
			  "cpu_weight": 100, 
			  "tags": {
				"domain": "cf-app-staging", 
				"lifecycle": "task", 
				"result-file": "/tmp/docker-result/result.json"
			  }, 
			  "allocated_at": 0, 
			  "rootfs": "/var/vcap/packages/rootfs_cflinuxfs2/rootfs", 
			  "external_ip": "", 
			  "ports": null, 
			  "log_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0, 
				"source_name": "STG"
			  }, 
			  "metrics_config": {
				"guid": "", 
				"index": 0
			  }, 
			  "start_timeout": 0, 
			  "setup": null, 
			  "run": {
				"timeout": {
				  "action": {
					"serial": {
					  "actions": [
						{
						  "emit_progress": {
							"action": {
							  "download": {
								"from": "http://file-server.service.cf.internal:8080/v1/static/docker_app_lifecycle/docker_app_lifecycle.tgz", 
								"to": "/tmp/docker_app_lifecycle", 
								"cache_key": "docker-lifecycle", 
								"user": "vcap"
							  }
							}, 
							"start_message": "", 
							"success_message": "", 
							"failure_message_prefix": "Failed to set up docker environment"
						  }
						}, 
						{
						  "emit_progress": {
							"action": {
							  "run": {
								"path": "/tmp/docker_app_lifecycle/builder", 
								"args": [
								  "-outputMetadataJSONFilename", 
								  "/tmp/docker-result/result.json", 
								  "-dockerRef", 
								  "tutum/tomcat:8.0"
								], 
								"env": [
								  {
									"name": "VCAP_APPLICATION", 
									"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
								  }, 
								  {
									"name": "VCAP_SERVICES", 
									"value": "{}"
								  }, 
								  {
									"name": "MEMORY_LIMIT", 
									"value": "256m"
								  }, 
								  {
									"name": "CF_STACK", 
									"value": "cflinuxfs2"
								  }
								], 
								"resource_limits": {
								  "nofile": 16384
								}, 
								"user": "vcap"
							  }
							}, 
							"start_message": "Staging...", 
							"success_message": "Staging Complete", 
							"failure_message_prefix": "Staging Failed"
						  }
						}
					  ]
					}
				  }, 
				  "timeout": 900000000000
				}
			  }, 
			  "monitor": null, 
			  "run_result": {
				"failed": false, 
				"failure_reason": "", 
				"stopped": false
			  }, 
			  "egress_rules": [
				{
				  "protocol": "all", 
				  "destinations": [
					"0.0.0.0-9.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"11.0.0.0-169.253.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"169.255.0.0-172.15.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"172.32.0.0-192.167.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"192.169.0.0-255.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "tcp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "udp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}
			  ]
			}, 
			"session": "4.2079"
		  }
		}

��ֻ��һ��**task**��Ϊ�������պö�Ӧ������֮ǰ������stager����������dockerֻ������һ��task������Ԥ�����������鿴docker image�Ƿ񱻻���ȵȡ�</br>

* Ȼ���ֱ���͵���LRP��ȥ����ʱ���Ѿ���һ��ʵ����</br>

		{
		  "timestamp": "1441087199.705446482", 
		  "source": "rep", 
		  "message": "rep.depot-client.allocate-containers.allocating-container", 
		  "log_level": 0, 
		  "data": {
			"container": {
			  "guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6-fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
			  "state": "", 
			  "privileged": false, 
			  "memory_mb": 256, 
			  "disk_mb": 1024, 
			  "cpu_weight": 1, 
			  "tags": {
				"domain": "cf-apps", 
				"instance-guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
				"lifecycle": "lrp", 
				"process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
				"process-index": "0"
			  }, 
			  "allocated_at": 0, 
			  "rootfs": "docker:///tutum/tomcat#8.0", 
			  "external_ip": "", 
			  "ports": [
				{
				  "container_port": 8080
				}, 
				{
				  "container_port": 2222
				}
			  ], 
			  "log_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0, 
				"source_name": "CELL"
			  }, 
			  "metrics_config": {
				"guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
				"index": 0
			  }, 
			  "start_timeout": 60, 
			  "setup": {
				"serial": {
				  "actions": [
					{
					  "download": {
						"from": "http://file-server.service.cf.internal:8080/v1/static/docker_app_lifecycle/docker_app_lifecycle.tgz", 
						"to": "/tmp/lifecycle", 
						"cache_key": "docker-lifecycle", 
						"user": "root"
					  }
					}
				  ]
				}
			  }, 
			  "run": {
				"codependent": {
				  "actions": [
					{
					  "run": {
						"path": "/tmp/lifecycle/launcher", 
						"args": [
						  "app", 
						  "", 
						  "{"cmd":["/run.sh"],"ports":[{"Port":8080,"Protocol":"tcp"}]}"
						], 
						"env": [
						  {
							"name": "VCAP_APPLICATION", 
							"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
						  }, 
						  {
							"name": "VCAP_SERVICES", 
							"value": "{}"
						  }, 
						  {
							"name": "MEMORY_LIMIT", 
							"value": "256m"
						  }, 
						  {
							"name": "CF_STACK", 
							"value": "cflinuxfs2"
						  }, 
						  {
							"name": "PORT", 
							"value": "8080"
						  }
						], 
						"resource_limits": {
						  "nofile": 16384
						}, 
						"user": "root", 
						"log_source": "APP"
					  }
					}, 
					{
					  "run": {
						"path": "/tmp/lifecycle/diego-sshd", 
						"args": [
						  "-address=0.0.0.0:2222", 
						  "-hostKey=", 
						  "-authorizedKey=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQC3zd/eSXS2peF10a0AHX60XDuCtb/kOtOdY/OE9/4IkJexT3pqMbQvf6Te8PxeGJklSOzvI7gP6x2wCWcO2JezKRXBxh2eMAyTvH4CC/gXnBl50vjmG6prbQEujv92+i9nN4K357AQPMY6WHQt4ZuGJcT87U3Iji6zAjGP/U9M5Q==
		", 
						  "-inheritDaemonEnv", 
						  "-logLevel=fatal"
						], 
						"env": [
						  {
							"name": "VCAP_APPLICATION", 
							"value": "{"limits":{"mem":256,"disk":1024,"fds":16384},"application_id":"c181be6b-d89c-469c-b79c-4cab1e99b8de","application_version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","application_name":"helloDocker","version":"6eac776b-f051-4cb0-9614-5a3954d1bfd6","name":"helloDocker","space_name":"cloud-space","space_id":"e73d49d7-e590-42fb-a046-50a9f5c259d5"}"
						  }, 
						  {
							"name": "VCAP_SERVICES", 
							"value": "{}"
						  }, 
						  {
							"name": "MEMORY_LIMIT", 
							"value": "256m"
						  }, 
						  {
							"name": "CF_STACK", 
							"value": "cflinuxfs2"
						  }, 
						  {
							"name": "PORT", 
							"value": "8080"
						  }
						], 
						"resource_limits": {
						  "nofile": 16384
						}, 
						"user": "root"
					  }
					}
				  ]
				}
			  }, 
			  "monitor": {
				"timeout": {
				  "action": {
					"run": {
					  "path": "/tmp/lifecycle/healthcheck", 
					  "args": [
						"-port=8080"
					  ], 
					  "resource_limits": {
						"nofile": 1024
					  }, 
					  "user": "root", 
					  "log_source": "HEALTH"
					}
				  }, 
				  "timeout": 30000000000
				}
			  }, 
			  "env": [
				{
				  "name": "INSTANCE_GUID", 
				  "value": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
				}, 
				{
				  "name": "INSTANCE_INDEX", 
				  "value": "0"
				}, 
				{
				  "name": "CF_INSTANCE_GUID", 
				  "value": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
				}, 
				{
				  "name": "CF_INSTANCE_INDEX", 
				  "value": "0"
				}
			  ], 
			  "run_result": {
				"failed": false, 
				"failure_reason": "", 
				"stopped": false
			  }, 
			  "egress_rules": [
				{
				  "protocol": "tcp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "udp", 
				  "destinations": [
					"0.0.0.0/0"
				  ], 
				  "ports": [
					53
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"0.0.0.0-9.255.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"11.0.0.0-169.253.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"169.255.0.0-172.15.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"172.32.0.0-192.167.255.255"
				  ], 
				  "log": false
				}, 
				{
				  "protocol": "all", 
				  "destinations": [
					"192.169.0.0-255.255.255.255"
				  ], 
				  "log": false
				}
			  ]
			}, 
			"session": "4.2092"
		  }
		}

ע�⿴���������</br>

> lifecycle��һ����task�����������lrp��
> rootfs: һ����rootfs_cflinuxfs2/rootfs��һ����docker:///tutum/tomcat#8.0

�����Ĳ����ˣ����ǴӸղŴ��stager���Կ���һЩdocker������ļ���</br>

��docker��</br>

> Client version: 1.6.2
> Client API version: 1.18
> Go version (client): go1.4.2
> Git commit (client): 7c8fca2
> OS/Arch (client): linux/amd64

˵����1.6.2�İ汾</br>
��������һ��֪ʶ�㣬�������������˽����ô�죬�������Ҳ���ò���diegoҲ������������</br>
��builder�����̫ˬ�ˣ�</br> �˴���stager��Ҳ�ܿ�����ֻ���������������</br>

		-cacheDockerImage=false: Caches Docker images to private docker registry
		-dockerDaemonExecutablePath="/tmp/docker_app_lifecycle/docker": path to the 'docker' executable
		-dockerEmail="": Email for pulling from docker registry
		-dockerImageURL="": docker image uri in docker://[registry/][scope/]repository[#tag] format
		-dockerLoginServer="https://index.docker.io/v1/": Docker Login server address
		-dockerPassword="": Password for pulling from docker registry
		-dockerRef="": docker image reference in standard docker string format
		-dockerRegistryHost="": Docker Registry host
		-dockerRegistryIPs=[]: Docker Registry IPs
		-dockerRegistryPort=8080: Docker Registry port
		-dockerUser="": User for pulling from docker registry
		-insecureDockerRegistries=[]: insecure Docker Registry addresses (host:port)
		-outputMetadataJSONFilename="/tmp/result/result.json": filename in which to write the app metadata

���и�**launcher**��һ������ִ��docker�ﶨ��Ľű�ʹ��</br>

����һ������sshd��diego-sshd���ߣ�</br>

		diego-sshd
		-address=0.0.0.0:2222��
		-hostKey="",
		-authorizedKey="rsa ---"
		-inheritDaemonEnv
		-logLevel=fatal

���һ��healthcheck</br>

		�����ʵ�ڷ���֮ǰ��Դ��ʱ�򣬿�������һ�£���������м��ַ�ʽ������Ǹ���port�����м��ģ�
		-port=8080
		���ǿ����ڱ�����ִ��һ�£�
		vagrant@agent-id-bosh-0:~$ ./healthcheck -port 22
		healthcheck passed

* ˵�괴������������Ҫ�˽�һ��rep����ΰ����������ģ�</br>
https://github.com/cloudfoundry-incubator/rep/blob/ab1835570afb992ff756f5d0b3dcb7af1978b639/generator/internal/container_delegate.go#L49</br>

		func (d *containerDelegate) RunContainer(logger lager.Logger, guid string) bool {
			logger.Info("running-container")
			err := d.client.RunContainer(guid)
			if err != nil {
				logInfoOrError(logger, "failed-running-container", err)
				d.DeleteContainer(logger, guid)
				return false
			}
			logger.Info("succeeded-running-container")
			return true
		}

��run������û�д�������ɾ��������Ȼ��������runContainer����������ô�����ģ�</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/depot.go#L209</br>

		func (c *client) RunContainer(guid string) error {
			logger := c.logger.Session("run-container", lager.Data{
				"guid": guid,
			})

			logger.Debug("initializing-container")
			//��ʼ������
			err := c.allocationStore.Initialize(logger, guid)
			if err != nil {
				logger.Error("failed-initializing-container", err)
				return err
			}
			logger.Debug("succeeded-initializing-container")

			c.creationWorkPool.Submit(func() {
				c.containerLockManager.Lock(guid)
				defer c.containerLockManager.Unlock(guid)

				logger.Debug("looking-up-in-allocation-store")
				container, err := c.allocationStore.Lookup(guid)
				if err != nil {
					logger.Error("failed-looking-up-in-allocation-store", err)
					return
				}
				logger.Debug("succeeded-looking-up-in-allocation-store")

				if container.State != executor.StateInitializing {
					logger.Error("container-state-invalid", err, lager.Data{"state": container.State})
					return
				}

				logger.Info("creating-container-in-garden")
				//��garden�д�������
				container, err = c.gardenStore.Create(logger, container)
				if err != nil {
					logger.Error("failed-creating-container-in-garden", err)
					c.allocationStore.Fail(logger, guid, ContainerInitializationFailedMessage)
					return
				}
				logger.Info("succeeded-creating-container-in-garden")

				if !c.allocationStore.Deallocate(logger, guid) {
					//���ʧ����������������������
					logger.Info("container-deallocated-during-initialization")

					err = c.gardenStore.Destroy(logger, guid)
					if err != nil {
						logger.Error("failed-to-destroy", err)
					}

					return
				}

				logger.Info("running-container-in-garden")
				//ִ����������������
				err = c.gardenStore.Run(logger, container)
				if err != nil {
					logger.Error("failed-running-container-in-garden", err)
				}
				logger.Info("succeeded-running-container-in-garden")
			})

			return nil
		}

--->ǰ��һֱ���ǳ�ʼ��������������������ô��garden�����ģ�container, err = c.gardenStore.Create(logger, container)</br>
https://github.com/cloudfoundry-incubator/executor/blob/6c38b2fe1a175d8074e0e0164bc9841fd023161c/depot/gardenstore/garden_store.go#L145</br>

		func (store *GardenStore) Create(logger lager.Logger, container executor.Container) (executor.Container, error) {
			if container.State != executor.StateInitializing {
				return executor.Container{}, executor.ErrInvalidTransition
			}
			container.State = executor.StateCreated

			logStreamer := log_streamer.New(
				container.LogConfig.Guid,
				container.LogConfig.SourceName,
				container.LogConfig.Index,
			)

			fmt.Fprintf(logStreamer.Stdout(), "Creating container\n")
			
			//�ؼ����������garden�������������
			container, err := store.exchanger.CreateInGarden(logger, store.gardenClient, container)
			if err != nil {
				fmt.Fprintf(logStreamer.Stderr(), "Failed to create container\n")
				return executor.Container{}, err
			}

			fmt.Fprintf(logStreamer.Stdout(), "Successfully created container\n")

			return container, nil
		}

* ����������������ڣ�store.exchanger.CreateInGarden(logger, store.gardenClient, container)</br>
��ϧһ�аѴ���ճ������</br>

		func (exchanger exchanger) CreateInGarden(logger lager.Logger, gardenClient GardenClient, executorContainer executor.Container) (executor.Container, error) {
			logger = logger.Session("create-in-garden", lager.Data{"container-guid": executorContainer.Guid})
			
			//����������Ϣ
			containerSpec := garden.ContainerSpec{
				Handle:     executorContainer.Guid,
				Privileged: executorContainer.Privileged,
				RootFSPath: executorContainer.RootFSPath,
			}

			//���������������ڴ�ת����bytes
			if executorContainer.MemoryMB != 0 {
				logger.Debug("setting-up-memory-limits")
				containerSpec.Limits.Memory.LimitInBytes = uint64(executorContainer.MemoryMB * 1024 * 1024)
			}

			logger.Debug("setting-up-disk-limits")
			
			//const DiskLimitScopeExclusive DiskLimitScope = 1
			//����������� �������һ�£�����ά�Ŀ����ӹ���
			//dd if=/dev/hda of=/root/image count=1 bs=512 �����ǽ�512byte�����ݿ���/root/image��ȥ�����������ν��mbr����
			//if �����ʲô�ط���of��д�����bs���������ÿ����ֽ�����count������Ҫд�Ŀ���
			//���֮ǰ���˶�ĳ���ļ���������ƣ����ڳ�����ʱ�򣬽��ᱨ����������д������
			gardenScope := garden.DiskLimitScopeExclusive
			
			//DiskLimitScopeTotal DiskLimitScope = 0
			
			if executorContainer.DiskScope == executor.TotalDiskLimit {
				gardenScope = garden.DiskLimitScopeTotal
			}
			
			//һ��docker��Ĭ�ϵ�DiskMB����1024M
			//ContainerInodeLimit: 200000 inodeĬ��Ϊ200000��
			containerSpec.Limits.Disk = garden.DiskLimits{
				ByteHard:  uint64(executorContainer.DiskMB * 1024 * 1024),
				InodeHard: exchanger.containerInodeLimit,
				Scope:     gardenScope,
			}

			logger.Debug("setting-up-cpu-limits")
			
			//ContainerMaxCpuShares: 0 �������Ӧ�ú���Ϥ������cgroup��һ����ϵͳ 0Ϊ������
			//containerMaxCpuShares=1024 ��������������rep��ʱ���Ѿ��趨�˸�ֵ��1024
			//����Ȩֵ�������ڷ���������ʱ���Ѿ�������Ĭ�ϵ�100 �����뿴�����AllocateContainers
			containerSpec.Limits.CPU.LimitInShares = uint64(float64(exchanger.containerMaxCPUShares) * float64(executorContainer.CPUWeight) / 100.0)

			logJson, err := json.Marshal(executorContainer.LogConfig)
			if err != nil {
				logger.Error("failed-marshal-log", err)
				return executor.Container{}, err
			}

			metricsConfigJson, err := json.Marshal(executorContainer.MetricsConfig)
			if err != nil {
				logger.Error("failed-marshal-metrics-config", err)
				return executor.Container{}, err
			}

			resultJson, err := json.Marshal(executorContainer.RunResult)
			if err != nil {
				logger.Error("failed-marshal-run-result", err)
				return executor.Container{}, err
			}

			//Ȼ�����һЩ�嵥����
			containerSpec.Properties = garden.Properties{
				ContainerOwnerProperty:         exchanger.containerOwnerName,
				ContainerStateProperty:         string(executorContainer.State),
				ContainerAllocatedAtProperty:   fmt.Sprintf("%d", executorContainer.AllocatedAt),
				ContainerStartTimeoutProperty:  fmt.Sprintf("%d", executorContainer.StartTimeout),
				ContainerRootfsProperty:        executorContainer.RootFSPath,
				ContainerLogProperty:           string(logJson),
				ContainerMetricsConfigProperty: string(metricsConfigJson),
				ContainerResultProperty:        string(resultJson),
				ContainerMemoryMBProperty:      fmt.Sprintf("%d", executorContainer.MemoryMB),
				ContainerDiskMBProperty:        fmt.Sprintf("%d", executorContainer.DiskMB),
				ContainerCPUWeightProperty:     fmt.Sprintf("%d", executorContainer.CPUWeight),
			}

			for name, value := range executorContainer.Tags {
				containerSpec.Properties[tagPropertyPrefix+name] = value
			}

			for _, env := range executorContainer.Env {
				containerSpec.Env = append(containerSpec.Env, env.Name+"="+env.Value)
			}

			for _, securityRule := range executorContainer.EgressRules {
				if err := securityRule.Validate(); err != nil {
					logger.Error("invalid-security-rule", err, lager.Data{"security_group_rule": securityRule})
					return executor.Container{}, executor.ErrInvalidSecurityGroup
				}
			}

			logger.Debug("creating-garden-container")
			gardenContainer, err := gardenClient.Create(containerSpec)
			if err != nil {
				logger.Error("failed-creating-garden-container", err)
				return executor.Container{}, err
			}
			logger.Debug("succeeded-creating-garden-container")

			//���ö˿ں�����ӳ��˿�
			if executorContainer.Ports != nil {
				actualPortMappings := make([]executor.PortMapping, len(executorContainer.Ports))

				logger.Debug("setting-up-ports")
				for i, ports := range executorContainer.Ports {
					actualHostPort, actualContainerPort, err := gardenContainer.NetIn(uint32(ports.HostPort), uint32(ports.ContainerPort))
					if err != nil {
						logger.Error("failed-setting-up-ports", err)
						exchanger.destroyContainer(logger, gardenClient, gardenContainer)
						return executor.Container{}, err
					}

					actualPortMappings[i].ContainerPort = uint16(actualContainerPort)
					actualPortMappings[i].HostPort = uint16(actualHostPort)
				}
				logger.Debug("succeeded-setting-up-ports")

				executorContainer.Ports = actualPortMappings
			}

			//���ð�ȫ�� ��ʵ��������������iptables
			//https://github.com/cloudfoundry-incubator/executor/blob/229bbf2af858bc00d14320249a4c16d908435682/depot/gardenstore/exchanger.go#L379
			for _, securityRule := range executorContainer.EgressRules {
				netOutRule, err := securityGroupRuleToNetOutRule(securityRule)
				if err != nil {
					logger.Error("failed-to-build-net-out-rule", err, lager.Data{"security_group_rule": securityRule})
					return executor.Container{}, err
				}

				logger.Debug("setting-up-net-out")
				err = gardenContainer.NetOut(netOutRule)
				if err != nil {
					logger.Error("failed-setting-up-net-out", err, lager.Data{"net-out-rule": netOutRule})
					exchanger.destroyContainer(logger, gardenClient, gardenContainer)
					return executor.Container{}, err
				}
				logger.Debug("succeeded-setting-up-net-out")
			}

			logger.Debug("getting-garden-container-info")
			info, err := gardenContainer.Info()
			if err != nil {
				logger.Error("failed-getting-garden-container-info", err)

				gardenErr := gardenClient.Destroy(gardenContainer.Handle())
				if gardenErr != nil {
					logger.Error("failed-destroy-garden-container", gardenErr)
				}

				return executor.Container{}, err
			}
			logger.Debug("succeeded-getting-garden-container-info")

			//���������externalIp�ͻ��������һ��Ϊ��
			executorContainer.ExternalIP = info.ExternalIP

			return executorContainer, nil
		}

�����Ĳ�������ˣ��޷Ǿ��Ǵ���LRPs����ʵ��������֮��ģ���֮�����ߵ�garden��һ�㡣</br>

### Garden�������
�ȿ���������������</br>

		#����cgroup�豸Ŀ¼ �����豸��ϵͳ���ص�cgroup��ȥ
		mkdir /tmp/devices-cgroup
		mount -t cgroup -o $devices_subsytems none /tmp/devices-cgroup
   
		#����btrfs�ļ�ϵͳ��������
		backing_store=/var/vcap/data/garden/garden_graph_backing_store 
		graph_path=/var/vcap/data/garden/btrfs_graph
		mount_point=$graph_path
		loopback_device=$(losetup -f --show $backing_store)
		mkfs.btrfs --nodiscard $loopback_device
		mount -t btrfs $loopback_device $mount_point
   
		/var/vcap/packages/garden-linux/bin/garden-linux \
		-depot=/var/vcap/data/garden/depot \
		-snapshots="${snapshots_path}" \
		-graph=$graph_path \
		-bin=/var/vcap/packages/garden-linux/src/github.com/cloudfoundry-incubator/garden-linux/linux_backend/bin \
		-mtu=1500 \
		-disableQuotas=false \
		-listenNetwork=tcp \
		-listenAddr=0.0.0.0:7777 \
		-denyNetworks=0.0.0.0/0 \
		-allowNetworks= \
		-allowHostAccess=false \
		-debugAddr=0.0.0.0:17013 \
		-rootfs=/var/vcap/packages/busybox \
		-containerGraceTime=5m
	  
ϵͳ��rootfses:</br>

		mkdir -p $RUN_DIR
		chown -R vcap:vcap $RUN_DIR

		echo $$ > $PIDFILE

		# Setup rootfs
		ROOTFS_PACKAGE=/var/vcap/packages/rootfs_cflinuxfs2/
		ROOTFS_DIR=$ROOTFS_PACKAGE/rootfs
		if [ ! -d $ROOTFS_DIR ]; then
		  mkdir -p $ROOTFS_DIR
		  tar -pzxf $ROOTFS_PACKAGE/cflinuxfs2.tar.gz -C $ROOTFS_DIR
		fi

### ������</br>
Garden:

* create/delete containers
* apply resource limits to containers
* open and attach network ports to containers
* copy files into/out of containers
* run processes within containers, streaming back stdout and stderr data
* annotate containers with arbitrary metadata
* snapshot containers for down-timeless redeploys

�������һ�������²����ʱ���ܽ������գ������Ķ��ǻ�������Դ���뻹Ӧ��</br>

�����ɻ����Garden-Linux</br>

������ϰ�ߣ��ȿ�routes ������Ժ������Ŀ���garden�������Щrestful������</br>

		var Routes = rata.Routes{
			{Path: "/ping", Method: "GET", Name: Ping},
			{Path: "/capacity", Method: "GET", Name: Capacity},

			{Path: "/containers", Method: "GET", Name: List},
			{Path: "/containers", Method: "POST", Name: Create},

			{Path: "/containers/:handle/info", Method: "GET", Name: Info},
			{Path: "/containers/bulk_info", Method: "GET", Name: BulkInfo},
			{Path: "/containers/bulk_metrics", Method: "GET", Name: BulkMetrics},

			{Path: "/containers/:handle", Method: "DELETE", Name: Destroy},
			{Path: "/containers/:handle/stop", Method: "PUT", Name: Stop},

			{Path: "/containers/:handle/files", Method: "PUT", Name: StreamIn},
			{Path: "/containers/:handle/files", Method: "GET", Name: StreamOut},

			{Path: "/containers/:handle/limits/bandwidth", Method: "PUT", Name: LimitBandwidth},
			{Path: "/containers/:handle/limits/bandwidth", Method: "GET", Name: CurrentBandwidthLimits},

			{Path: "/containers/:handle/limits/cpu", Method: "PUT", Name: LimitCPU},
			{Path: "/containers/:handle/limits/cpu", Method: "GET", Name: CurrentCPULimits},

			{Path: "/containers/:handle/limits/disk", Method: "PUT", Name: LimitDisk},
			{Path: "/containers/:handle/limits/disk", Method: "GET", Name: CurrentDiskLimits},

			{Path: "/containers/:handle/limits/memory", Method: "PUT", Name: LimitMemory},
			{Path: "/containers/:handle/limits/memory", Method: "GET", Name: CurrentMemoryLimits},

			{Path: "/containers/:handle/net/in", Method: "POST", Name: NetIn},
			{Path: "/containers/:handle/net/out", Method: "POST", Name: NetOut},

			{Path: "/containers/:handle/processes/:pid/attaches/:streamid/stdout", Method: "GET", Name: Stdout},
			{Path: "/containers/:handle/processes/:pid/attaches/:streamid/stderr", Method: "GET", Name: Stderr},
			{Path: "/containers/:handle/processes", Method: "POST", Name: Run},
			{Path: "/containers/:handle/processes/:pid", Method: "GET", Name: Attach},

			{Path: "/containers/:handle/properties", Method: "GET", Name: Properties},
			{Path: "/containers/:handle/properties/:key", Method: "GET", Name: Property},
			{Path: "/containers/:handle/properties/:key", Method: "PUT", Name: SetProperty},
			{Path: "/containers/:handle/properties/:key", Method: "DELETE", Name: RemoveProperty},

			{Path: "/containers/:handle/metrics", Method: "GET", Name: Metrics},
		}

�����handle����֮ǰ��container��ID��c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6-fc4a83cd-5ed9-40fb-51bf-1679beb41bfe</br>

���ǿ������ִ��һ�£�/info ��ȡһ��������info</br>

		{
		  "State": "active", 
		  "Events": [ ], 
		  "HostIP": "10.254.0.2", 
		  "ContainerIP": "10.254.0.1", 
		  "ExternalIP": "10.244.16.138", 
		  "ContainerPath": "/var/vcap/data/garden/depot/vrishcl40k3", 
		  "ProcessIDs": [
			1, 
			2
		  ], 
		  "Properties": {
			"executor:allocated-at": "1441087199719774323", 
			"executor:cpu-weight": "1", 
			"executor:disk-mb": "1024", 
			"executor:log-config": "{"guid":"c181be6b-d89c-469c-b79c-4cab1e99b8de","index":0,"source_name":"CELL"}", 
			"executor:memory-mb": "256", 
			"executor:metrics-config": "{"guid":"c181be6b-d89c-469c-b79c-4cab1e99b8de","index":0}", 
			"executor:owner": "executor", 
			"executor:result": "{"failed":false,"failure_reason":"","stopped":false}", 
			"executor:rootfs": "docker:///tutum/tomcat#8.0", 
			"executor:start-timeout": "60", 
			"executor:state": "running", 
			"tag:domain": "cf-apps", 
			"tag:instance-guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
			"tag:lifecycle": "lrp", 
			"tag:process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
			"tag:process-index": "0"
		  }, 
		  "MappedPorts": [
			{
			  "HostPort": 60000, 
			  "ContainerPort": 8080
			}, 
			{
			  "HostPort": 60001, 
			  "ContainerPort": 2222
			}
		  ]
		}

���ˣ�������������������ô�����ģ�</br>
https://github.com/cloudfoundry-incubator/garden/blob/master/server/request_handling.go#L52</br>

		func (s *GardenServer) handleCreate(w http.ResponseWriter, r *http.Request) {
			var spec garden.ContainerSpec
			if !s.readRequest(&spec, w, r) {
				return
			}

			hLog := s.logger.Session("create", lager.Data{
				"request": containerDebugInfo{
					Handle:     spec.Handle,
					GraceTime:  spec.GraceTime,
					RootFSPath: spec.RootFSPath,
					BindMounts: spec.BindMounts,
					Network:    spec.Network,
					Privileged: spec.Privileged,
					Limits:     spec.Limits,
				},
			})

			if spec.GraceTime == 0 {
				spec.GraceTime = s.containerGraceTime
			}

			hLog.Debug("creating")

			//�ؼ������������ǰ��ķ��������ڹ���spec�嵥Ҳ����task����DesiredLsp���嵥
			container, err := s.backend.Create(spec)
			if err != nil {
				s.writeError(w, err, hLog)
				return
			}

			hLog.Info("created")

			s.bomberman.Strap(container)

			s.writeResponse(w, &struct{ Handle string }{
				Handle: container.Handle(),
			})
		}

Ȼ�󿴵����https://github.com/cloudfoundry-incubator/garden/blob/master/client/connection/connection.go#L126</br>

		func (c *connection) Create(spec garden.ContainerSpec) (string, error) {
			res := struct {
				Handle string `json:"handle"`
			}{}

			err := c.do(routes.Create, spec, &res, nil, nil)
			if err != nil {
				return "", err
			}

			return res.Handle, nil
		}

����,�����������json�Ľ����ˣ�����ʵ�ֵĺ����ţ�</br>

		func (c *connection) do(
			handler string,
			req, res interface{},
			params rata.Params,
			query url.Values,
		) error {
			var body io.Reader

			if req != nil {
				buf := new(bytes.Buffer)

				err := transport.WriteMessage(buf, req)
				if err != nil {
					return err
				}

				body = buf
			}

			contentType := ""
			if req != nil {
				contentType = "application/json"
			}

			response, err := c.hijacker.Stream(
				handler,
				body,
				params,
				query,
				contentType,
			)
			if err != nil {
				return err
			}

			defer response.Close()

			return json.NewDecoder(response).Decode(res)
		}

### ������garden-linux���ɣ����Ǹ��Ӵ�Ҳ������ϵͳ������ĵĲ�����

* garden-linux

��garden-server��garden-linux�������������ĺ����������������ʱ�򣬺�˼���ʼִ����Ӧ�Ĳ���</br>

����**garden-server**�������Ĺ��̣�</br>

1.��ʼ��docker��Graph������</br>

		dockerGraphDriver, err := graphdriver.New(*graphRoot, nil)
		dockerGraph, err := graph.NewGraph(*graphRoot, dockerGraphDriver)

2.Ȼ�����btrfs��ʽ�������ļ�ϵͳ</br>

		graphMountPoint := mountPoint(logger, *graphRoot)

3.����garden�Ļ�������</br>

		cake = &layercake.BtrfsCleaningCake{
			Cake:            cake,
			Runner:          runner,
			BtrfsMountPoint: graphMountPoint,
			RemoveAll:       os.RemoveAll,
			Logger:          logger.Session("btrfs-cleanup"),
		}
		
4.����repository_fetcher</br>

������4��������dockerRegistry,cake,map[registry.APIVersion]repository_fetcher.VersionedFetcher,repository_fetcher.EndpointPinger{}
�����Ǹ��ݲ�ͬ�汾��registry api�汾����fetcher���Լ���repository_fetcher��</br>

5.����uidNamespace,����uid gid��Χ</br>

		rootFSNamespacer := &rootfs_provider.UidNamespacer
		
6.����RootFsProvider���� rootfs_provider��</br>

		//docker��rootfs
		remoteRootFSProvider, err := rootfs_provider.NewDocker(fmt.Sprintf("docker-remote-%s", cake.DriverName()),
		repoFetcher, cake, rootfs_provider.SimpleVolumeCreator{}, rootFSNamespacer, clock.NewClock())

		//�Լ�warden��rootfs
		localRootFSProvider, err := rootfs_provider.NewDocker(fmt.Sprintf("docker-local-%s", cake.DriverName()),
		&repository_fetcher.Local{
			Cake:              cake,
			DefaultRootFSPath: *rootFSPath,
			IDProvider:        repository_fetcher.LayerIDProvider{},
		}, cake, rootfs_provider.SimpleVolumeCreator{}, rootFSNamespacer, clock.NewClock())
		
		rootFSProviders := map[string]rootfs_provider.RootFSProvider{
			"":       localRootFSProvider,
			"docker": remoteRootFSProvider,
		}
	
7.����externalIP��ʵ������local ip</br>

	ip, err := localip.LocalIP()

8.����quotaManager ����graphMountPoint��������btrfs��������</br>

		var quotaManager linux_container.QuotaManager = quota_manager.DisabledQuotaManager{}
		if !*disableQuotas {
			quotaManager = &quota_manager.BtrfsQuotaManager{
				Runner:     runner,
				MountPoint: graphMountPoint,
			}
		}

9.����һЩ������Դpool��iptables,mtu��subnetPool�ȣ�����������</br>

10.�������ϵ���Դ���䣬����linux_backed</br>

		backend := linux_backend.New(logger, pool, container_repository.New(), injector, systemInfo, *snapshotsPath, int(*maxContainers))
		err = backend.Setup()

11.������϶�û�д���ʼ����gardenServer</br>

		graceTime := *containerGraceTime
		gardenServer := server.New(*listenNetwork, *listenAddr, graceTime, backend, logger)
		

---------------------------------------------------------------------------------------------------
�ٷ��������漰ͼ��һ���������������̣�һ����gardenServer��θ�backed��˽���ͨ�ŵ�</br>
![github](http://github.com/wdxxs2z/PictureStore/diego/container creation.png "github")

����������ʵ����һ����wardenû��ʲô������������**AcquirePoolResources**��**AcquireSystemResources**����������</br>

��ȫ��Ϊ������docker����,����resource_pool��</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go</br>

ֱ��ȥ�������Acquire,��һ��garden���������������</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L244</br>

		(p *LinuxResourcePool) Acquire(spec garden.ContainerSpec) (linux_backend.LinuxContainerSpec, error)

�����и�������garden.ContainerSpec�����sPecֵ����֮ǰ���Ƕ���container��һЩ���������������Ͱ�ȫ��ȵ�</br>

1.����container id ,container path,depotPathһ��Ϊ��/var/vcap/data/garden/depot</br>

		containerID��id := <-p.containerIDs��
		containerPath := path.Join(p.depotPath, id)
		
2.��ʼ��ȡpoolResource</br>

		resources, err := p.acquirePoolResources(spec, id)
		https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L484

		func (p *LinuxResourcePool) acquirePoolResources(spec garden.ContainerSpec, id string) (*linux_backend.Resources, error) {
			//��ʵ��CELL��IP
			resources := linux_backend.NewResources(0, nil, "", nil, p.externalIP)
			
			//����spec�е�network����
			subnet, ip, err := parseNetworkSpec(spec.Network)
			if err != nil {
				return nil, fmt.Errorf("create container: invalid network spec: %v", err)
			}

			//����Privileged�ж��Ƿ���rootȨ�ޣ������uid�϶���0��һ����build�����ʱ�����ֵһ�㶼Ϊtrue��uidΪroot
			if err := p.acquireUID(resources, spec.Privileged); err != nil {
				return nil, err
			}

			//����subnet��ip����resources.Network
			//https://github.com/cloudfoundry-incubator/garden-linux/blob/59c89dc849e992f5a5f7531889c493cfd844bc4d/network/subnets/subnets.go#L69
			if resources.Network, err = p.subnetPool.Acquire(subnet, ip); err != nil {
				p.releasePoolResources(resources)
				return nil, err
			}

			return resources, nil
		}

3.����handleId,�����ǰû�з��䣬���ID���ó�handlerId</br>

		handle := getHandle(spec.Handle, id)

4.���ô������</br>

		var quota int64 = int64(spec.Limits.Disk.ByteHard)
		if quota == 0 {
			quota = math.MaxInt64
		}

5.����containerRootFSPath, rootFSEnv���ص���acquireSystemResources</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L268</br>

		containerRootFSPath, rootFSEnv, err := p.acquireSystemResources(id, handle, containerPath, spec.RootFSPath, resources, spec.BindMounts, quota, pLog)

**spec.BindMounts**�����ǳ�˵��docker volume</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/resource_pool/resource_pool.go#L524</br>

1).����containerPath</br>

		os.MkdirAll(containerPath, 0755)
		
2).����rootfsUrl dockerһ��Ϊdocker:\\\</br>
3).����rootfsProviders��docker��warden</br>

		provider, found := p.rootfsProviders[rootfsURL.Scheme]
		
4).���ݲ�ͬ��provider���ò�ͬ��rootfsPath</br>

		rootfsPath, rootFSEnvVars, err := provider.ProvideRootFS(pLog.Session("create-rootfs"), id, rootfsURL, resources.RootUID != 0, diskQuota)
		
dockerһ���ȥ���Լ���layer�������layer�Ѿ���garden���ˣ�Ҳ������btrfsĿ¼��</br>
�������ͨ��buildpack�����ȥ�����Լҵ�rootfs��/var/vcap/packages/rootfs_cflinuxfs2/rootfs</br>

5).Ϊ��ǰ��������һ�����ţ����������ʵ��Ϊ�˷���ͬһ��CELL�еĲ�ͬ����ͨ���õģ���Ϊʵ��CIDRΪ/30�Ļ��֣������2��IP����</br>

6).������������һ����wb��ͷ ����������ID</br>

7).�������һЩ�е��������������ˣ���create.sh����ű���ʼ�����û��������ȣ�Ȼ�󿴵�һ��������</br>

		err = p.writeBindMounts(containerPath, rootfsPath, bindMounts)
		ÿ��containerPath�¶���һ��libĿ¼�������м����ű�����ʵ������Ӧ�ú���Ϥ����CF��v2����Ҳ�У���������cgroup
		hook-parent-before-clone.sh ����Ͳ���������

6.�������ж�������ɺ󣬿�ʼ�ϲ���������</br>

		specEnv, err := process.NewEnv(spec.Env)
		spec.Env = rootFSEnv.Merge(specEnv).Array()

7.���շ���һ���ṹ�壺</br>

		return linux_backend.LinuxContainerSpec{
			ID:                  id,
			ContainerPath:       containerPath,
			ContainerRootFSPath: containerRootFSPath,
			Resources:           resources,
			Events:              []string{},
			Version:             p.currentContainerVersion,
			State:               linux_backend.StateBorn,

			ContainerSpec: spec,
		}, nil

������������Դ���־��Ѿ��������ˡ�

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
* ���ڿ��Խ�����۽��������ط���һ����garden����ι���docker����ģ�һ����garden����δ��������</br>

		��Ϊ֮ǰ����rootfsProvider��
		type RootFSProvider interface {
			Name() string
			ProvideRootFS(logger lager.Logger, id string, rootfs *url.URL, namespaced bool, quota int64) (mountpoint string, envvar process.Env, err error)
		}

		type Graph interface {
			layercake.Cake
		}

��������graph������о���dockerԴ�룬Ӧ��֪��docker��һ��graph driver��������layer���ɶѵ��ļ�ϵͳ�������������кܶ�ʵ�֣���aufs,btrfs�������Լҵ�DeviceMapper����ʵ��Ϊ�˴������ŵ�metadata</br>

ProvideRootFS������docker image</br>

1.Ĭ������һ��tag := "latest" </br>

2.fetch����</br>

		fetchedID, envvars, volumes, err := provider.repoFetcher.Fetch(logger, url, tag, quota)
		
�����֪�������fetch�����ؾ���ģ����Դ��������֣�</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go</br>

������ʹ��docker��ʱ����pull�����ʱ��һ���ʽ��some-repository-name:target �����ϸ�����˵��docker:///��some-repository-name:target</br>
gardenҲ�����⣺</br>

1).��ȡ�����metadata</br>

		imgID, endpoints, err := fetcher.fetchImageMetadata(request)
		
2).ͨ������endpoints����ȡimage,��������ʹ��dockerʱ�ῴ����</br>

		31fa814ba25a: Pulling image (latest) from training/webapp, endpoint: https://reg31fa814ba25a: Pulling dependent layers
		image, err := fetcher.fetchFromEndpoint(request, endpointURL, imgID, request.Logger)
		
����ȥ��https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go#L79</br>

һ��һ��image��ֺܶ�㣬���Ի�һ��һ��Ļ�ȡ���ڻ�ȡĳһ��֮ǰ���ȼ����һ���layer�Ƿ��Ѿ����������</br>

		var allLayers []*dockerLayer
		layer, err := fetcher.fetchLayer(request, endpointURL, history[i], remainingQuota, logger)
		allLayers = append(allLayers, layer)
		
��������fetchLayer:</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/4c869ef07d712cfe007c4ed1f81b576efa640c04/repository_fetcher/remote_v1.go#L109</br>

		func (fetcher *RemoteV1Fetcher) fetchLayer(request *FetchRequest, endpointURL string, layerID string, remaining int64, logger lager.Logger) (*dockerLayer, error)
		����Ƿ񱻻��棬����У���ֱ�ӷ������layer��û�еģ���ͨ�����������ȡ��
		fetcher.Cake.Get(layercake.DockerImageID(layerID))
		ÿ�����ض��Ὺ����ʱ��Ȼ��ͳ������������õ�ʱ�䣺took�������ῴ������״̬��һ����downloading��download

3.����imageId��containerID������garden�Լ���rootfs</br>

		provider.graph.Create(containerID, imageID)

https://github.com/cloudfoundry-incubator/garden-linux/blob/964c92719378f8ef0bdbe60726e2f9ab42b69850/layercake/docker.go#L24</br>

		func (d *Docker) Create(containerID ID, imageID ID) error {
			return d.Register(
				&image.Image{
					ID:     containerID.GraphID(),
					Parent: imageID.GraphID(),
				}, nil)
		}

���Կ�����ʵgarden�ڴ洢�Լ��ľ���ʱ��һ����containerId,Ҳ����ʵ��ID������һ���������¼һ��docker Image��ID</br>

4.�����volume������������graph���ļ�ϵͳ�ﴴ����һ��volume,����ٷ�ֻ˵��Ŀǰֻ�Ǽ򵥵�ʵ�ִ�������û�����κι���</br>

ͨ���鿴��־�����ǵ�֪��645c4570fd120f7ed5bed9277886af4797e1874e3d5b571257b8ffdc6596bf9eΪ����imageId,Ȼ�����ǻ��ῴ��һ��08on3iof61u�����Ƕ���
/var/vcap/data/garden/btrfs_graph/btrfs/subvolumes/���Ŀ¼�£����ݷ�����Դ�룬Ҳ��֪�������08on3iof61u����containerID.GraphID()����ʵҲ�Ǵ������rootPath�ĵط�
/var/vcap/data/garden/depot/08on3iof61u</br>

���ǻ��ܿ���һ��NamespacedLayerID��645c4570fd120f7ed5bed9277886af4797e1874e3d5b571257b8ffdc6596bf9e@0-4294967294-1,1-1-4294967293+0-4294967294-1,1-1-4294967293</br>
		
		func (n NamespacedLayerID) GraphID() string {
			return shaID(n.LayerID + "@" + n.CacheKey)
		}

��һ��֪ʶ���ڹ���ÿһ���ʱ�򣬶�������һlayer�Ĵ���json��size���ݣ������docker�����</br>
/var/vcap/data/garden/btrfs_graph/{layerId}</br>

		{
		  "id": "ff365bfa7ca61680fbbe4b27d3473d7b5d76adde64c199fb312dcd30c3302b0f", 
		  "parent": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
		  "created": "2015-07-26T17:15:16.767121317Z", 
		  "container": "3a0dd2e1601f9def91d517491ccb1ce20abc9f8ac720f22e1b2e533bbb5039db", 
		  "container_config": {
			"Hostname": "dd360632d03c", 
			"Domainname": "", 
			"User": "", 
			"AttachStdin": false, 
			"AttachStdout": false, 
			"AttachStderr": false, 
			"PortSpecs": null, 
			"ExposedPorts": null, 
			"Tty": false, 
			"OpenStdin": false, 
			"StdinOnce": false, 
			"Env": [
			  "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 
			  "HOME=/root"
			], 
			"Cmd": [
			  "/bin/sh", 
			  "-c", 
			  "#(nop) ENTRYPOINT ["/scripts/run.sh"]"
			], 
			"Image": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
			"Volumes": null, 
			"VolumeDriver": "", 
			"WorkingDir": "/root", 
			"Entrypoint": [
			  "/scripts/run.sh"
			], 
			"NetworkDisabled": false, 
			"MacAddress": "", 
			"OnBuild": [ ], 
			"Labels": { }
		  }, 
		  "docker_version": "1.6.2", 
		  "author": "Ferran Rodenas <frodenas@gmail.com>", 
		  "config": {
			"Hostname": "dd360632d03c", 
			"Domainname": "", 
			"User": "", 
			"AttachStdin": false, 
			"AttachStdout": false, 
			"AttachStderr": false, 
			"PortSpecs": null, 
			"ExposedPorts": null, 
			"Tty": false, 
			"OpenStdin": false, 
			"StdinOnce": false, 
			"Env": [
			  "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 
			  "HOME=/root"
			], 
			"Cmd": null, 
			"Image": "a827709e978385e0e2998703fbe17f934c1a6bc233c7eb98820140ed5e279c23", 
			"Volumes": null, 
			"VolumeDriver": "", 
			"WorkingDir": "/root", 
			"Entrypoint": [
			  "/scripts/run.sh"
			], 
			"NetworkDisabled": false, 
			"MacAddress": "", 
			"OnBuild": [ ], 
			"Labels": { }
		  }, 
		  "architecture": "amd64", 
		  "os": "linux", 
		  "Size": 0
		}

		laylersize:0

���������ʵ����һ�����ʣ�����btrfs��ôû�г��֣���ʵbtrfs�ڴ����͹��غú��������еľ��������ֻ��btrfs���ļ�ϵͳ�������������ôʵ�֣�����ͺ�
garden�޹��ˣ���Ϊ����������docker��ʵ�֣�copy on write ���ƣ�˵�������ã�������Ҳ�µ��ˣ��������������ʱ��ʹ�ã�</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/6b419ed1e7020930425adc05bc29602dc774eb16/linux_container/quota_manager/btrfs_quota_manager.go</br>
����Ͳ���ȥ�ˣ���ҪΪbtrfs�Ĵ������������ƺͻ�ȡbtrfs�������ľ�����Ϣ�ȡ�</br>


* ����������garden����ι�������ģ�</br>
����֪��docker�ڴ�������ʱ���������һ������deamon������ʱ����ʼ��һ��docker bridge,�ڶ������ڴ���������ʱ�����˵������ʱ������һ��veth pair��һ���������
һ��patch��docker�����ϣ�����ǽ�����һ�˵�container���䵽���е�PID��Ҳ����namespace�С�</br>
����garden�Ƚ��ر�����������ÿ��������ʱ�򴴽�veth��ͬʱ��Ϊÿ��������CIDR 30������һ��bridge������veth����һ��patch�����bridge��ȥ</br>
https://github.com/cloudfoundry-incubator/garden-linux/blob/master/network/configure.go#L55</br>

* Bridge:

		Veth interface {
			Create(hostIfcName, containerIfcName string) (*net.Interface, *net.Interface, error)
		}
		Bridge interface {
			Create(bridgeName string, ip net.IP, subnet *net.IPNet) (*net.Interface, error)
			Add(bridge, slave *net.Interface) error
		}
		
����ǰ˵һ��bridgeName�����ÿ��ʵ���µ�һ��bridge-name�ļ��</br>
/var/vcap/data/garden/depot/08on3iof61u��wb-08on3ioescs0</br>

		//name������������ip�����ŵ�ip��subnet������ һ����cidr /30,����ʵ����docker��libcontainer/netlink��
		func (Bridge) Create(name string, ip net.IP, subnet *net.IPNet) (intf *net.Interface, err error) {
			netlinkMu.Lock()
			defer netlinkMu.Unlock()

			if err := netlink.NetworkLinkAdd(name, "bridge"); err != nil && err.Error() != "file exists" {
				return nil, fmt.Errorf("devices: create bridge: %v", err)
			}

			if intf, err = net.InterfaceByName(name); err != nil {
				return nil, fmt.Errorf("devices: look up created bridge interface: %v", err)
			}

			if err = netlink.NetworkLinkAddIp(intf, ip, subnet); err != nil && err.Error() != "file exists" {
				return nil, fmt.Errorf("devices: add IP to bridge: %v", err)
			}
			return intf, nil
		}

* Veth:
hostIfcNameΪ���������������ƣ�containerIfcNameΪ����ʵ����</br>

		func (VethCreator) Create(hostIfcName, containerIfcName string) (host, container *net.Interface, err error) {
			netlinkMu.Lock()
			defer netlinkMu.Unlock()

			if err := netlink.NetworkCreateVethPair(hostIfcName, containerIfcName, 1); err != nil {
				return nil, nil, fmt.Errorf("devices: create veth pair: %v", err)
			}

			if host, err = net.InterfaceByName(hostIfcName); err != nil {
				return nil, nil, fmt.Errorf("devices: look up created host interface: %v", err)
			}

			if container, err = net.InterfaceByName(containerIfcName); err != nil {
				return nil, nil, fmt.Errorf("devices: look up created container interface: %v", err)
			}

			return host, container, nil
		}
		
Ȼ����ǽ�veth����һ�˺�bridge��������</br>

		c.configureHostIntf(cLog, host, bridge, config.Mtu)

����ڽ�container��ĩ��IP���뵽namespace��ȥ��</br>

		// move container end in to container
		if err = c.Link.SetNs(container, config.ContainerPid); err != nil {
			return &SetNsFailedError{err, container, config.ContainerPid}
		}

������������һ�¾����������������ʱ�������Ѿ��ϴ���һ��dockerӦ�ã�</br>

		w08on3iof61u-0 Link encap:Ethernet  HWaddr 6a:54:f3:2f:23:00  
				  inet6 addr: fe80::6854:f3ff:fe2f:2300/64 Scope:Link
				  UP BROADCAST RUNNING  MTU:1500  Metric:1
				  RX packets:12 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:19 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:1 
				  RX bytes:928 (928.0 B)  TX bytes:1486 (1.4 KB)

		w08on3iof621-0 Link encap:Ethernet  HWaddr 72:74:d0:b1:e4:d7  
				  inet6 addr: fe80::7074:d0ff:feb1:e4d7/64 Scope:Link
				  UP BROADCAST RUNNING  MTU:1500  Metric:1
				  RX packets:11 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:18 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:1 
				  RX bytes:801 (801.0 B)  TX bytes:1434 (1.4 KB)

		wb-08on3ioescs0 Link encap:Ethernet  HWaddr 6a:54:f3:2f:23:00  
				  inet addr:10.254.0.2  Bcast:0.0.0.0  Mask:255.255.255.252
				  inet6 addr: fe80::b4ad:1ff:fede:9357/64 Scope:Link
				  UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
				  RX packets:12 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:0 
				  RX bytes:760 (760.0 B)  TX bytes:928 (928.0 B)

		wb-08on3ioescv0 Link encap:Ethernet  HWaddr 72:74:d0:b1:e4:d7  
				  inet addr:10.254.0.6  Bcast:0.0.0.0  Mask:255.255.255.252
				  inet6 addr: fe80::6c45:aeff:fe38:7803/64 Scope:Link
				  UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
				  RX packets:11 errors:0 dropped:0 overruns:0 frame:0
				  TX packets:11 errors:0 dropped:0 overruns:0 carrier:0
				  collisions:0 txqueuelen:0 
				  RX bytes:647 (647.0 B)  TX bytes:876 (876.0 B)

root@5ccb272f-018c-4b31-a5a4-92c8c7626a4c:/tmp/devices-cgroup/instance-08jaqet7bv6/instance-08on3iof61u# brctl show </br>

		bridge name     bridge id               STP enabled     interfaces
		wb-08on3ioescs0         8000.6a54f32f2300       no              w08on3iof61u-0
		wb-08on3ioescv0         8000.7274d0b1e4d7       no              w08on3iof621-0

root@5ccb272f-018c-4b31-a5a4-92c8c7626a4c:/tmp/devices-cgroup/instance-08jaqet7bv6/instance-08on3iof61u# bridge li</br>

		4: w08on3iof61u-0 state UP : <BROADCAST,UP,LOWER_UP> mtu 1500 master wb-08on3ioescs0 state forwarding priority 32 cost 2 
		13: w08on3iof621-0 state UP : <BROADCAST,UP,LOWER_UP> mtu 1500 master wb-08on3ioescv0 state forwarding priority 32 cost 2
