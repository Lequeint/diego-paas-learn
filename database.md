# database ����(����)
���濪ʼ��database���ֿ�ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

databaseһ����2��������
-----------------------------------
* etcd �����database�Ĵ洢����
* bbs BBS������diego��䵱�ľ���etcd�������ṩ��ԭʼ���ݵĴ洢�����ṩ��һ��restful�ڲ�ʹ�õ�API

### bbs�������
�ȿ���������������</br>

		/var/vcap/packages/bbs/bin/bbs ${etcd_sec_flags} \
		-address=0.0.0.0:8889 \
		-auctioneerAddress=http://auctioneer.service.cf.internal:9016 \
		-debugAddr=0.0.0.0:17017 \
		-consulCluster=http://127.0.0.1:8500 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-logLevel=info
		
��Ϊ��etcd�Ľ�������漰�Ĳ���ҲԽ��Խ�ײ�

		var Routes = rata.Routes{
			// Domains
			{Path: "/v1/domains/list", Method: "POST", Name: DomainsRoute},
			{Path: "/v1/domains/upsert", Method: "POST", Name: UpsertDomainRoute},

			// Actual LRPs
			{Path: "/v1/actual_lrp_groups/list", Method: "POST", Name: ActualLRPGroupsRoute},
			{Path: "/v1/actual_lrp_groups/list_by_process_guid", Method: "POST", Name: ActualLRPGroupsByProcessGuidRoute},
			{Path: "/v1/actual_lrp_groups/get_by_process_guid_and_index", Method: "POST", Name: ActualLRPGroupByProcessGuidAndIndexRoute},

			// Actual LRP Lifecycle
			{Path: "/v1/actual_lrps/claim", Method: "POST", Name: ClaimActualLRPRoute},
			{Path: "/v1/actual_lrps/start", Method: "POST", Name: StartActualLRPRoute},
			{Path: "/v1/actual_lrps/crash", Method: "POST", Name: CrashActualLRPRoute},
			{Path: "/v1/actual_lrps/fail", Method: "POST", Name: FailActualLRPRoute},
			{Path: "/v1/actual_lrps/remove", Method: "POST", Name: RemoveActualLRPRoute},
			{Path: "/v1/actual_lrps/retire", Method: "POST", Name: RetireActualLRPRoute},

			// Evacuation
			{Path: "/v1/actual_lrps/remove_evacuating", Method: "POST", Name: RemoveEvacuatingActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_claimed", Method: "POST", Name: EvacuateClaimedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_crashed", Method: "POST", Name: EvacuateCrashedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_stopped", Method: "POST", Name: EvacuateStoppedActualLRPRoute},
			{Path: "/v1/actual_lrps/evacuate_running", Method: "POST", Name: EvacuateRunningActualLRPRoute},

			// Desired LRPs
			{Path: "/v1/desired_lrps/list", Method: "POST", Name: DesiredLRPsRoute},
			{Path: "/v1/desired_lrps/get_by_process_guid", Method: "POST", Name: DesiredLRPByProcessGuidRoute},

			// Desire LPR Lifecycle
			{Path: "/v1/desired_lrp/desire", Method: "POST", Name: DesireDesiredLRPRoute},
			{Path: "/v1/desired_lrp/update", Method: "POST", Name: UpdateDesiredLRPRoute},
			{Path: "/v1/desired_lrp/remove", Method: "POST", Name: RemoveDesiredLRPRoute},

			// LRP Convergence
			{Path: "/v1/lrps/converge", Method: "POST", Name: ConvergeLRPsRoute},

			// Tasks
			{Path: "/v1/tasks/list", Method: "POST", Name: TasksRoute},
			{Path: "/v1/tasks/get_by_task_guid", Method: "GET", Name: TaskByGuidRoute},

			// Task Lifecycle
			{Path: "/v1/tasks/desire", Method: "POST", Name: DesireTaskRoute},
			{Path: "/v1/tasks/start", Method: "POST", Name: StartTaskRoute},
			{Path: "/v1/tasks/cancel", Method: "POST", Name: CancelTaskRoute},
			{Path: "/v1/tasks/fail", Method: "POST", Name: FailTaskRoute},
			{Path: "/v1/tasks/complete", Method: "POST", Name: CompleteTaskRoute},
			{Path: "/v1/tasks/resolving", Method: "POST", Name: ResolvingTaskRoute},
			{Path: "/v1/tasks/delete", Method: "POST", Name: DeleteTaskRoute},

			// Task Convergence
			{Path: "/v1/tasks/converge", Method: "POST", Name: ConvergeTasksRoute},

			// Event Streaming
			{Path: "/v1/events", Method: "GET", Name: EventStreamRoute},
		}
		
��receptor�����Ѿ���������������Ҫ�߼�����Ҫ������Ŀ¼��</br>
handlers��db������db����һ�����ǽӿڣ�������ʵ����db/etcd����е����󶼻�������holdס</br>

etcd���node��洢���У�</br>

		/v1/desired
		/v1/actual
		/v1/domain
		/v1/task

����Ҫע�����������¼����͵ģ�����etcd��watcher��������</br>

		/db/etcd/event_db.go

		func (db *ETCDDB) WatchForActualLRPChanges(logger lager.Logger,
			created func(*models.ActualLRPGroup),
			changed func(*models.ActualLRPChange),
			deleted func(*models.ActualLRPGroup),
		) (chan<- bool, <-chan error)
		
		//������������actualLRP�ڵ��ǰ��仯��
		events, stop, err := db.watch(ActualLRPSchemaRoot) 
		
		//Ȼ����һ��goroutine:�������е�events
		case event.Node != nil && event.PrevNode == nil:�����ڵ�
		case event.Node != nil && event.PrevNode != nil:���½ڵ�
		case event.PrevNode != nil && event.Node == nil:ɾ���ڵ�

�����ActualLRP node�и�TTL�ĸ�����緢��ĳ���ڵ㼴�����ڣ����������������ͻ��������ִ����߼���WatchForDesiredLRPChangesû���������</br>

		evacuating := isEvacuatingActualLRPNode(event.PrevNode)
		actualLRPGroup := &models.ActualLRPGroup{}
		if evacuating {
			actualLRPGroup.Evacuating = &actualLRP
		} else {
			actualLRPGroup.Instance = &actualLRP
		}

���ﲻ�ǹ̶��ģ����µ�ʱ����ܼ�Ҫ����ǰ���ڵ㣬��Ҫ���Ǻ�̽ڵ㣺</br>

		evacuating := isEvacuatingActualLRPNode(event.Node)
		beforeGroup := &models.ActualLRPGroup{}
		afterGroup := &models.ActualLRPGroup{}
		if evacuating {
			afterGroup.Evacuating = &after
			beforeGroup.Evacuating = &before
		} else {
			afterGroup.Instance = &after
			beforeGroup.Instance = &before
		}

��һ�����isEvacuatingActualLRPNode������</br>
https://github.com/cloudfoundry-incubator/runtime-schema/blob/ba8fb9905c07f6f0598e4f9db06368374064ae67/bbs/lrp_bbs/actual_group_getters.go#L104</br>

		func isEvacuatingActualLRPNode(node storeadapter.StoreNode) bool {
			return path.Base(node.Key) == shared.ActualLRPEvacuatingKey
		}

		const ActualLRPEvacuatingKey = "evacuating"

��������Ͳ��ö�˵�ˣ��������ͱ����Ȱѽڵ����һ�顣