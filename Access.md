# Access ����(����)
���濪ʼ��access�㿪ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

accessһ����2��������
-----------------------------------
* Receptor  �������CC-bridge�����Ĵ���tasks��Lrps������ǰ��dea�У���û������ͳ����н���֮˵�ģ������ṩ��restful API ��ͨ�� ���garden-linux�͸����齨֮�����ϵ ͬʱ����֤����BBS�˵�����ͬ��
* ssh-proxy һ��ssh�˵ĸ��ؾ�����

### ��һ���ἰһ��Desired��Actual������LRP�ؼ���
DesiredLRP���ŵ���һ���嵥�������մ����database�Ҳ��������֤�˺���Ӧ��ʵ��������˿�������������desiredLRP���Ƕ����</br>
ActualLRP��ά������ʵ����������DesiredLRP����һЩʵ��������Ϣ����ЩҲ���Ǵ����database���</br>

### Receptor�������
�ȿ���������������</br>

		/var/vcap/packages/receptor/bin/receptor ${etcd_sec_flags} \
		-address=0.0.0.0:8887 \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-taskHandlerAddress=10.244.16.6:1169 \
		-debugAddr=0.0.0.0:17014 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-consulCluster=http://127.0.0.1:8500 \
		-username= \
		-password= \
		-registerWithRouter=true \
		-domainNames=receptor.10.244.0.34.xip.io \
		-natsAddresses=10.244.0.6:4222 \
		-natsUsername=nats \
		-natsPassword=nats \
		-corsEnabled=false \
		-logLevel=debug
		
### ������
�Ȳ�����Դ�룬�ȿ�����ʵ��.����ע�⵽receptorʵ�����ṩ��һ���Ѻõ�restful api�����Ƿ��ʣ����������ܿ���ʲô��</br>
ͨ������http://receptor.10.244.0.34.xip.io/v1/desired_lrps </br>
���Կ�����ֻ��һ���嵥������˵�������Ӧ���ж���ʵ�������õ�rootfs����Դ���ƣ���ȫ��ȵȡ�</br>
<string>DesiredLRP</strong></br>

		[
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"domain": "cf-apps", 
		"rootfs": "docker:///tutum/tomcat#8.0", 
		"instances": 2, 
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
		"action": {
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
		"start_timeout": 60, 
		"disk_mb": 1024, 
		"memory_mb": 256, 
		"cpu_weight": 1, 
		"privileged": false, 
		"ports": [
		  8080, 
		  2222
		], 
		"routes": {
		  "cf-router": [
			{
			  "hostnames": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "port": 8080
			}
		  ], 
		  "diego-ssh": {
			"container_port": 2222, 
			"host_fingerprint": "b8:d0:bf:b8:1f:06:d0:1a:d2:7c:ea:91:b5:70:43:fc", 
			"private_key": ""
		  }
		}, 
		"log_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		"log_source": "CELL", 
		"metrics_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		"annotation": "1441088697.8285718", 
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
		], 
		"modification_tag": {
		  "epoch": "c69ce119-896b-4a3a-77be-3dc793908340", 
		  "index": 3
		}
		}
		]
	
������������ActualLRP���涨����ʲô��</br>
����ͨ������ http://receptor.10.244.0.34.xip.io/v1/actual_lrps</br>
��������Կ�����actualLRPֻ����Ӧ�õ�����ʵ���ڲ������ľ�����Ϣ������host������ַ�������˿ں�����ӳ��˿ڣ�����״̬�ȵȡ�</br>
<string>ActualLRP</strong></br>

		[
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"instance_guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
		"cell_id": "cell_z1-0", 
		"domain": "cf-apps", 
		"index": 0, 
		"address": "10.244.16.138", 
		"ports": [
		  {
			"container_port": 8080, 
			"host_port": 60000
		  }, 
		  {
			"container_port": 2222, 
			"host_port": 60001
		  }
		], 
		"state": "RUNNING", 
		"crash_count": 0, 
		"since": 1441087965748636200, 
		"evacuating": false, 
		"modification_tag": {
		  "epoch": "d292b1b4-fd3e-4df1-5ea3-733e14a616ae", 
		  "index": 27
		}
		}, 
		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"instance_guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		"cell_id": "cell_z1-0", 
		"domain": "cf-apps", 
		"index": 1, 
		"address": "10.244.16.138", 
		"ports": [
		  {
			"container_port": 8080, 
			"host_port": 60002
		  }, 
		  {
			"container_port": 2222, 
			"host_port": 60003
		  }
		], 
		"state": "RUNNING", 
		"crash_count": 0, 
		"since": 1441088706589340400, 
		"evacuating": false, 
		"modification_tag": {
		  "epoch": "c5e63fbb-eea9-4c1c-4a4a-1513ecfbb781", 
		  "index": 2
		}
		}
		]

�˽�����������LRPs������������������������Ҫ��Networking Information</br>
���µĹ����ǱȽ�cool�Ĺ��ܣ�������Ҫ��cells���rep�齨�н�-exportNetworkEnvVars ���ó�enable</br>
CF_INSTANCE_IP������ָ��ʵ�����������е�IP��</br>
CF_INSTANCE_PORT�� ����˿��趨��Ӧ��desiredLRP��ports�����е�һ��port��ӳ��</br>
CF_INSTANCE_ADDR�� $CF_INSTANCE_IP:$CF_INSTANCE_PORT</br>
CF_INSTANCE_PORTS�� [{"external":60413,"internal":8080},{"external":60414,"internal":2222}]</br>
	
���������ǻ��ܿ���������tasks����lrps��ִ�е�ʱ���ж�����ִ�У������ڱ����ʱ�����download,upload�ȶ���</br>
**RunAction**: ������������һ������</br>
**DownloadAction**: ��ȡһ��tgz����zip������ѹ����������</br>
**UploadAction**: �������������ϴ�һ���������ļ���ָ����fileserver,����һ��Ϊdroplet</br>
**ParallelAction**: ͬʱ���ж������</br>
**CodependentAction**: ���е�ִ�����еĶ�����ֻҪ��һ��ִ�гɹ������˳����������������̫������</br>
**SerialAction**: ����һ��˳��ִ�ж���</br>
**EmitProgressAction**: Ƕ��ʽ��ִ�ж���</br>
**TimeoutAction**: �������Ƕ��ʽ�Ķ�����һ��ʱ����û������򱨴�</br>
**TryAction**: �����ִ��Ƕ��ʽ�Ķ���ʱ�д����������</br>
	
���ڿ�ʼ��һ��Դ�룺</br>
	
���¶����˻����ĳ�����restful����ȵȡ�</br>

		const (
		// Tasks
		CreateTaskRoute = "CreateTask"
		TasksRoute      = "Tasks"
		GetTaskRoute    = "GetTask"
		DeleteTaskRoute = "DeleteTask"
		CancelTaskRoute = "CancelTask"

		// DesiredLRPs
		CreateDesiredLRPRoute = "CreateDesiredLRP"
		GetDesiredLRPRoute    = "GetDesiredLRP"
		UpdateDesiredLRPRoute = "UpdateDesiredLRP"
		DeleteDesiredLRPRoute = "DeleteDesiredLRP"
		DesiredLRPsRoute      = "DesiredLRPs"

		// ActualLRPs
		ActualLRPsRoute                         = "ActualLRPs"
		ActualLRPsByProcessGuidRoute            = "ActualLRPsByProcessGuid"
		ActualLRPByProcessGuidAndIndexRoute     = "ActualLRPByProcessGuidAndIndex"
		KillActualLRPByProcessGuidAndIndexRoute = "KillActualLRPByProcessGuidAndIndex"

		// Cells
		CellsRoute = "Cells"

		// Domains
		UpsertDomainRoute = "UpsertDomain"
		DomainsRoute      = "Domains"

		// Event Streaming
		EventStream = "EventStream"

		// Authentication Cookie
		GenerateCookie = "GenerateCookie"
		)

		var Routes = rata.Routes{
		// Tasks
		{Path: "/v1/tasks", Method: "POST", Name: CreateTaskRoute},
		{Path: "/v1/tasks", Method: "GET", Name: TasksRoute},
		{Path: "/v1/tasks/:task_guid", Method: "GET", Name: GetTaskRoute},
		{Path: "/v1/tasks/:task_guid", Method: "DELETE", Name: DeleteTaskRoute},
		{Path: "/v1/tasks/:task_guid/cancel", Method: "POST", Name: CancelTaskRoute},

		// DesiredLRPs
		{Path: "/v1/desired_lrps", Method: "GET", Name: DesiredLRPsRoute},
		{Path: "/v1/desired_lrps", Method: "POST", Name: CreateDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "GET", Name: GetDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "PUT", Name: UpdateDesiredLRPRoute},
		{Path: "/v1/desired_lrps/:process_guid", Method: "DELETE", Name: DeleteDesiredLRPRoute},

		// ActualLRPs
		{Path: "/v1/actual_lrps", Method: "GET", Name: ActualLRPsRoute},
		{Path: "/v1/actual_lrps/:process_guid", Method: "GET", Name: ActualLRPsByProcessGuidRoute},
		{Path: "/v1/actual_lrps/:process_guid/index/:index", Method: "GET", Name: ActualLRPByProcessGuidAndIndexRoute},
		{Path: "/v1/actual_lrps/:process_guid/index/:index", Method: "DELETE", Name: KillActualLRPByProcessGuidAndIndexRoute},

		// Cells
		{Path: "/v1/cells", Method: "GET", Name: CellsRoute},

		// Domains
		{Path: "/v1/domains/:domain", Method: "PUT", Name: UpsertDomainRoute},
		{Path: "/v1/domains", Method: "GET", Name: DomainsRoute},

		// Event Streaming
		{Path: "/v1/events", Method: "GET", Name: EventStream},

		// Authentication Cookie
		{Path: "/v1/auth_cookie", Method: "POST", Name: GenerateCookie},
		}

���������еĲ�������ͨ���ͻ�������������Ĵ����ǽ����˺ܶ಻ͬ����handler�����ǿ�����1����</br>
--->handlers/actual_lrp_handlers.go</br>
��Щhandler���漰��һ���Ƚϸ������ֵĿͻ��ˣ�bbs bbs.Client</br>
���ǿ����������func (h *ActualLRPHandler) GetAllByProcessGuid(w http.ResponseWriter, req *http.Request) ������ͬС��</br>
	
1.���Ȼ�ȡprocess_guid </br>
processGuid := req.FormValue(":process_guid")</br>

2.��ʼ��ȡ����actualLRPs</br>
actualLRPGroupsByIndex, err := h.bbs.ActualLRPGroupsByProcessGuid(processGuid)</br>
--->����ActualLRPGroupsByProcessGuid���濴һ�£�https://github.com/cloudfoundry-incubator/bbs/blob/373fe5f7af9b9bfdb311a6f88bf8ba8c06555d86/handlers/actual_lrp_handlers.go#L38</br>
		
		func (h *ActualLRPHandler) ActualLRPGroupsByProcessGuid(w http.ResponseWriter, req *http.Request) {
			logger := h.logger.Session("actual-lrp-groups-by-process-guid")

			request := &models.ActualLRPGroupsByProcessGuidRequest{}
			response := &models.ActualLRPGroupsResponse{}

			response.Error = parseRequest(logger, req, request)
			if response.Error == nil {
				response.ActualLrpGroups, response.Error = h.db.ActualLRPGroupsByProcessGuid(logger, request.ProcessGuid)
			}

			writeResponse(w, response)
		}
		
--->���ﻹ�ǿ�������������https://github.com/cloudfoundry-incubator/bbs/blob/cd77526f2067736f163a784c6d1a351162ddf855/db/etcd/actual_lrp_db.go#L70</br>
����һ����Ϥ�Ķ�����**etcd**,û�����ͨ��fetch etcd�����node����ȡ������Ԫ���ݵģ����ǽ�ȥ��һ�£�</br>

		func (db *ETCDDB) ActualLRPGroupsByProcessGuid(logger lager.Logger, processGuid string) ([]*models.ActualLRPGroup, *models.Error) {
			node, bbsErr := db.fetchRecursiveRaw(logger, ActualLRPProcessDir(processGuid))
			if bbsErr.Equal(models.ErrResourceNotFound) {
				return []*models.ActualLRPGroup{}, nil
			}
			if bbsErr != nil {
				return nil, bbsErr
			}
			if node.Nodes.Len() == 0 {
			return []*models.ActualLRPGroup{}, nil
			}

			return parseActualLRPGroups(logger, node, models.ActualLRPFilter{})
		}
		
--->�������ǽ����붨λ����etcd�ͺ������̶���һ��fetchRecursiveRaw</br>

		https://github.com/cloudfoundry-incubator/bbs/blob/a504a4e4bdfdeb1363a75a380aaa422bf8799f87/db/etcd/etcd_db.go
		func (db *ETCDDB) fetchRecursiveRaw(logger lager.Logger, key string) (*etcd.Node, *models.Error) {
			logger.Debug("fetching-recursive-from-etcd")
			response, err := db.client.Get(key, false, true)
			if err != nil {
				return nil, ErrorFromEtcdError(logger, err)
			}
			logger.Debug("succeeded-fetching-recursive-from-etcd", lager.Data{"num-nodes": response.Node.Nodes.Len()})
			return response.Node, nil
		}
		
�Ҿ��������¸��ˣ���Ϊdb.client.Get(key, false, true)������ͨ��process_guid�����������Ϣ</br>
	
��󷵻صľ�������ṹ�壺����Ϣ���Ǹ�������ǰ��չʾ�Ĳ�����:</br>
etcd��洢��·��Ϊ��</br>
/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6</br>
���ǽ�ȥ������</br>

		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/0
		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/1
		
����������������˵��������������ʵ����</br>

		/v1/actual/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6/0/instance
	
�������ʵ����ȥ��һ�ۣ����ָ����Ǹղŵõ�����һ����</br>

		{
		"process_guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
		"index": 0, 
		"domain": "cf-apps", 
		"instance_guid": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe", 
		"cell_id": "cell_z1-0", 
		"address": "10.244.16.138", 
		"ports": [
		{
		  "container_port": 8080, 
		  "host_port": 60000
		}, 
		{
		  "container_port": 2222, 
		  "host_port": 60001
		}
		], 
		"crash_count": 0, 
		"state": "RUNNING", 
		"since": 1441087965748636200, 
		"modification_tag": {
		"epoch": "d292b1b4-fd3e-4df1-5ea3-733e14a616ae", 
		"index": 27
		}
		}
	
DesiredLRP�������ģ�/v1/desired/c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6</br>
��������л��Ľṹ��</br>
--->/serialization/actual_lrps.go</br>
	
		func ActualLRPProtoToResponse(actualLRP *models.ActualLRP, evacuating bool) receptor.ActualLRPResponse {
		return receptor.ActualLRPResponse{
			ProcessGuid:     actualLRP.ProcessGuid,
			InstanceGuid:    actualLRP.InstanceGuid,
			CellID:          actualLRP.CellId,
			Domain:          actualLRP.Domain,
			Index:           int(actualLRP.Index),
			Address:         actualLRP.Address,
			Ports:           PortMappingFromProto(actualLRP.Ports),
			State:           actualLRPProtoStateToResponseState(actualLRP.State),
			PlacementError:  actualLRP.PlacementError,
			Since:           actualLRP.Since,
			CrashCount:      int(actualLRP.CrashCount),
			CrashReason:     actualLRP.CrashReason,
			Evacuating:      evacuating,
			ModificationTag: actualLRPProtoModificationTagToResponseModificationTag(actualLRP.ModificationTag),
		}
		}
	
### С������������ȫ��ǿ���etcd��ǳ����ף�

		CERT_DIR=/var/vcap/jobs/etcd/config/certs
		ca_cert_file=${CERT_DIR}/server-ca.crt
		server_cert_file=${CERT_DIR}/server.crt
		server_key_file=${CERT_DIR}/server.key
		client_cert_file=${CERT_DIR}/client.crt
		client_key_file=${CERT_DIR}/client.key
		
		etcdctl_sec_flags=" \
		-ca-file=${ca_cert_file} \
		-cert-file=${client_cert_file} \
		-key-file=${client_key_file}"
		
		etcd_sec_flags=" \
		--client-cert-auth \
		--trusted-ca-file ${ca_cert_file} \
		--cert-file ${server_cert_file} \
		--key-file ${server_key_file}"
		
		peer_ca_cert_file=${CERT_DIR}/peer-ca.crt
		peer_cert_file=${CERT_DIR}/peer.crt
		peer_key_file=${CERT_DIR}/peer.key

		etcd_peer_sec_flags=" \
		--peer-client-cert-auth \
		--peer-trusted-ca-file ${peer_ca_cert_file} \
		--peer-cert-file ${peer_cert_file} \
		--peer-key-file ${peer_key_file}"

		etcd_sec_flags="${etcd_sec_flags} ${etcd_peer_sec_flags}"
  
��ʱ������ִ�У�
/var/vcap/packages/etcd/etcdctl ${etcdctl_sec_flags} -C 'https://database-z1-0.etcd.service.cf.internal:4001' ls