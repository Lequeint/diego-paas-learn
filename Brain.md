# Brains ����(����)
���濪ʼ��brain��ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

Brainsһ����3��������
-----------------------------------
* auctioneer ���ļ��㵥Ԫ������
* converger  ֱ�Ӹ�BBSͨ�ţ���Ϊ�����ߣ���˼���ƹ�tasks��lrps��״̬��Ϣ���ݴ�
* runtime_metrics_server ����tasks��lrps��ͳ����Ϣ

### auctioneer�������
�ȿ���������������</br>

		/var/vcap/packages/auctioneer/bin/auctioneer ${etcd_sec_flags} \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17001 \
		-listenAddr=0.0.0.0:9016 \
		-logLevel=info
		
### ������</br>
����**task**��**actualLrp**ʵ�������õ�**action**ȥִ����Ӧ�Ķ���������԰������Ϊһ��ִ�����Ĵ���
������Ҫ����һ�������**auction**�����actionͨ�����http������ͨѶ������auctioneer��cell Rep�Ĺ�ͨ������
�ṩ��ά�ֻ�������������ֻ��һ��auctioneer��ĳһʱ�����actions�����������ԡ�</br>

����ֻ�ÿ�һ���ط������ˣ�</br>
https://github.com/cloudfoundry-incubator/auctioneer/blob/4d3c25962b9a55a3fcce6c7962ad32b346346d52/auctionrunnerdelegate/auction_runner_delegate.go#L31</br>

		func (a *AuctionRunnerDelegate) FetchCellReps() (map[string]auctiontypes.CellRep, error) {
			cells, err := a.legacyBBS.Cells()
			cellReps := map[string]auctiontypes.CellRep{}
			if err != nil {
				return cellReps, err
			}

			for _, cell := range cells {
				cellReps[cell.CellID] = auction_http_client.New(a.client, cell.CellID, cell.RepAddress, a.logger.Session(cell.RepAddress))
			}

			return cellReps, nil
		}
		
���Կ���cell��һЩ������Ϣ��������rootfs_providers</br>

		[
		  {
			"cell_id": "cell_z1-0", 
			"zone": "z1", 
			"capacity": {
			  "memory_mb": 5968, 
			  "disk_mb": 79252, 
			  "containers": 256
			}, 
			"rootfs_providers": {
			  "docker": [ ], 
			  "preloaded": [
				"cflinuxfs2"
			  ]
			}
		  }
		]
		
��һ��auction_http_client���������������ǵ�����auction��ȿ�һ��action��·�ɹ���:</br>

		const (
			State   = "STATE"
			Perform = "PERFORM"

			Sim_Reset = "RESET"
		)

		var Routes = rata.Routes{
			{Path: "/state", Method: "GET", Name: State},
			{Path: "/work", Method: "POST", Name: Perform},

			{Path: "/sim/reset", Method: "POST", Name: Sim_Reset},
		}
		
Ȼ���ٶ�λ��https://github.com/cloudfoundry-incubator/auction/blob/master/communication/http/auction_http_client/auction_http_client.go#L28</br>

		func New(client *http.Client, repGuid string, address string, logger lager.Logger) *AuctionHTTPClient {
			return &AuctionHTTPClient{
				client:           client,
				repGuid:          repGuid,
				address:          address,
				requestGenerator: rata.NewRequestGenerator(address, routes.Routes),
				logger:           logger,
			}
		}
		
�������auction_http_client.go</br>
����������actionHttpClient������</br>

		State() (auctiontypes.CellState, error),
		Perform(work auctiontypes.Work) (auctiontypes.Work, error),
		Reset() error

��ʵ�����ڲ������͵����������ύ��������η��䵽cells�������漰��һЩ��Դ�����㷨��</br>
�����и�����˼�ĳƺ� cf���˹�����������˭Ӯ�ˣ�˭���õ�LRP��ִ��Ȩ������auction����rep����Ӧ��</br>
/auctionrunner/scheduler.go</br>
����ôһ��ע�ͣ�</br>
/*
Schedule takes in a set of job requests (LRP start auctions and task starts) and
assigns the work to available cells according to the diego scoring algorithm. The
scheduler is single-threaded.  It determines scheduling of jobs one at a time so
that each calculation reflects available resources correctly.  It commits the
work in batches at the end, for better network performance.  Schedule returns
AuctionResults, indicating the success or failure of each requested job.
*/</br>

������˼��˵��cf�����diego�������㷨������Щ�������lrp��������е�ĳ��cell���������ǵ��̵߳ģ���������һ�ε���jobs�ͽ�������ȷ���õ���Դ��Ϊ�˱���������磬��������ε��ύ
����ÿ������Ҫô�ɹ�Ҫôʧ�ܡ�</br>

�ڽ����㷨ǰ���ȿ�һ���ṹ��</br>
https://github.com/cloudfoundry-incubator/auction/blob/master/auctiontypes/types.go#L61</br>

		type LRPAuction struct {
			DesiredLRP *models.DesiredLRP
			Index      int
			AuctionRecord
		}

		type AuctionRecord struct {
			Winner   string
			Attempts int

			QueueTime    time.Time
			WaitDuration time.Duration

			PlacementError string
		}
		
DesiredLRP��</br>
https://github.com/cloudfoundry-incubator/bbs/blob/master/models/desired_lrp.go#L51</br>

�ٸ�����:</br>

		var lrpB = &models.DesiredLRP{
			ProcessGuid: "process-guid-b",
			Instances:   2,
			Domain:      domainB,
		}

���ˣ�Ԥ����������ˣ����ڽ����㷨���̣�</br>
https://github.com/cloudfoundry-incubator/auction/blob/master/auctionrunner/scheduler.go#L191</br>

ֻ����һ��LRP�ĵ����㷨��</br>

		func (s *Scheduler) scheduleLRPAuction(lrpAuction auctiontypes.LRPAuction) (auctiontypes.LRPAuction, error) {
			var winnerCell *Cell
			winnerScore := 1e20

			zones := accumulateZonesByInstances(s.zones, lrpAuction)

			filteredZones := filterZonesByRootFS(zones, lrpAuction.DesiredLRP.RootFs)

			if len(filteredZones) == 0 {
				return auctiontypes.LRPAuction{}, auctiontypes.ErrorCellMismatch
			}

			sortedZones := sortZonesByInstances(filteredZones)

			for zoneIndex, lrpByZone := range sortedZones {
				for _, cell := range lrpByZone.zone {
					score, err := cell.ScoreForLRPAuction(lrpAuction)
					if err != nil {
						continue
					}

					if score < winnerScore {
						winnerScore = score
						winnerCell = cell
					}
				}

				if zoneIndex+1 < len(sortedZones) &&
					lrpByZone.instances == sortedZones[zoneIndex+1].instances {
					continue
				}

				if winnerCell != nil {
					break
				}
			}

			if winnerCell == nil {
				return auctiontypes.LRPAuction{}, auctiontypes.ErrorInsufficientResources
			}

			err := winnerCell.ReserveLRP(lrpAuction)
			if err != nil {
				return auctiontypes.LRPAuction{}, err
			}

			lrpAuction.Winner = winnerCell.Guid
			return lrpAuction, nil
		}

### ����
1.���ȸ���һ������ķ�����1e20 û���ܳ�������</br>

2.����accumulateZonesByInstances���������https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L22</br>
		
		func accumulateZonesByInstances(zones map[string]Zone, lrpAuction auctiontypes.LRPAuction) []lrpByZone {
			lrpZones := []lrpByZone{}

			for _, zone := range zones {
				instances := 0
				for _, cell := range zone {
					for _, lrp := range cell.state.LRPs {
						if lrp.ProcessGuid == lrpAuction.DesiredLRP.ProcessGuid {
							instances++
						}
					}
				}
				lrpZones = append(lrpZones, lrpByZone{zone, instances})
			}
			return lrpZones
		}
		
ͳ��lrpAuction��DesiredLRP�еĽ���ID�Ƿ�������zone��ÿ��cell�У�����У���ͳ�Ƴ�ÿ��zone�д�˽��̵ĸ���</br>

3.filterZonesByRootFS��https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L45</br>
		
		func filterZonesByRootFS(zones []lrpByZone, rootFS string) []lrpByZone {
			filteredZones := []lrpByZone{}

			for _, lrpZone := range zones {
				cells := lrpZone.zone.FilterCells(rootFS)
				if len(cells) > 0 {
					filteredZone := lrpByZone{
						zone:      Zone(cells),
						instances: lrpZone.instances,
					}
					filteredZones = append(filteredZones, filteredZone)
				}
			}
			return filteredZones
		}
		
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/scheduler.go#L15</br>

		func (z *Zone) FilterCells(rootFS string) []*Cell {
			var cells = make([]*Cell, 0, len(*z))

			for _, cell := range *z {
				if cell.MatchRootFS(rootFS) {
					cells = append(cells, cell)
				}
			}

			return cells
		}
		
ƥ��rootfs��һ����docker��diegoϵͳ�Դ���rootfs</br>

4.sortedZones := sortZonesByInstances(filteredZones) ����zones�����ʵ�������Ķ���������</br>

https://github.com/cloudfoundry-incubator/auction/blob/29173639425e010af0e0943ee6c1105a11498062/auctionrunner/zone_sorter.go#L39

		func sortZonesByInstances(zones []lrpByZone) []lrpByZone {
			sorter := zoneSorterByInstances{zones: zones}
			sort.Sort(sorter)
			return sorter.zones
		}
		
5.������������Щzones��Ȼ��ͨ��lrpAuction����ķ�����</br>
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/cell.go#L28</br>
		
		func (c *Cell) ScoreForLRPAuction(lrpAuction auctiontypes.LRPAuction) (float64, error) {
			err := c.canHandleLRPAuction(lrpAuction)
			if err != nil {
				return 0, err
			}

			numberOfInstancesWithMatchingProcessGuid := 0
			for _, lrp := range c.state.LRPs {
				if lrp.ProcessGuid == lrpAuction.DesiredLRP.ProcessGuid {
					numberOfInstancesWithMatchingProcessGuid++
				}
			}

			remainingResources := c.state.AvailableResources
			remainingResources.MemoryMB -= int(lrpAuction.DesiredLRP.MemoryMb)
			remainingResources.DiskMB -= int(lrpAuction.DesiredLRP.DiskMb)
			remainingResources.Containers -= 1

			resourceScore := c.computeScore(remainingResources, numberOfInstancesWithMatchingProcessGuid)

			return resourceScore, nil
		}
		
����DesiredLRP��Ԥ����ڴ�ʹ��̷��䣬c.state.AvailableResources����һ��������Դ</br>
Ȼ��Ϳ�ʼ��ֵ��MemoryMB��DiskMB��Containers</br>
**Ȼ������㷨���ģ�**computeScore</br>
https://github.com/cloudfoundry-incubator/auction/blob/fcf9393a3a76b2883ebe0eadf32cf0a06fb75195/auctionrunner/cell.go#L159</br>

		func (c *Cell) computeScore(remainingResources auctiontypes.Resources, numInstances int) float64 {
			fractionUsedMemory := 1.0 - float64(remainingResources.MemoryMB)/float64(c.state.TotalResources.MemoryMB)
			fractionUsedDisk := 1.0 - float64(remainingResources.DiskMB)/float64(c.state.TotalResources.DiskMB)
			fractionUsedContainers := 1.0 - float64(remainingResources.Containers)/float64(c.state.TotalResources.Containers)

			resourceScore := (fractionUsedMemory + fractionUsedDisk + fractionUsedContainers) / 3.0
			resourceScore += float64(numInstances)

			return resourceScore
		}
		
6.���һ�ж�OK�����LRP��action�洢����</br>

		func (c *Cell) ReserveLRP(lrpAuction auctiontypes.LRPAuction) error {
			err := c.canHandleLRPAuction(lrpAuction)
			if err != nil {
				return err
			}

			c.state.LRPs = append(c.state.LRPs, auctiontypes.LRP{
				ProcessGuid: lrpAuction.DesiredLRP.ProcessGuid,
				Index:       lrpAuction.Index,
				MemoryMB:    int(lrpAuction.DesiredLRP.MemoryMb),
				DiskMB:      int(lrpAuction.DesiredLRP.DiskMb),
			})

			c.state.AvailableResources.MemoryMB -= int(lrpAuction.DesiredLRP.MemoryMb)
			c.state.AvailableResources.DiskMB -= int(lrpAuction.DesiredLRP.DiskMb)
			c.state.AvailableResources.Containers -= 1

			c.workToCommit.LRPs = append(c.workToCommit.LRPs, lrpAuction)

			return nil
		}

�㷨���漰��һ��ģ����Կ�ܣ�http://onsi.github.io/ginkgo/ ����BDD-style���Կ�����еĴ��붼��</br>
simulation�����Ȥ�Ŀ���ȥ����������һЩ���Ա���</br>

### converger�������
�ȿ���������������</br>

		/var/vcap/packages/converger/bin/converger ${etcd_sec_flags} \
		-etcdCluster="https://etcd.service.cf.internal:4001" \
		-bbsAddress=http://bbs.service.cf.internal:8889 \
		-consulCluster=http://127.0.0.1:8500 \
		-receptorTaskHandlerURL=http://receptor.service.cf.internal:1169 \
		-debugAddr=0.0.0.0:17002 \
		-convergeRepeatInterval=30s \
		-kickTaskDuration=30s \
		-expireCompletedTaskDuration=120s \
		-expirePendingTaskDuration=1800s \
		-logLevel=debug
		
˵������������Ϊ�����ߣ��ٷ�����4������˵����</br>

* ά��BBS������ȷ�������������ܣ��ر��ᵽ���������ǵ��ݵģ���˼���ǣ�����д�����Ҳ����Ӱ�쵽��״̬</br>
* ʹ������������ȷ��bbs�������һ���ԣ�����Tasks��LRPs���ݴ���</br>
* ������LRPs��ʱ����������Ҫȷ������˵Э�̣�DesiredLRP��ActualLRP��״̬��</br>
������Ҫע�⣬���ʵ����ʧ��auction���ᱻ���ͣ�����ζ�����᳢��ִ�ж�ʧ�Ķ���
��������һ����ʱ��ʵ������ô������ᷢ��һ��ֹͣ��Ϣ��cell������rep�����
* ���⣬�������������۲�һЩǱ�ڵĶ�ʧ����Ϣ������һ��һֱ����pending״̬��������ʱ�����auction�������п�����Զ���ᱻ���͸�auctioneer������Ҫ�ɵ�����</br>

���������˵�ӳ���ǻ�û�г����깤���ȿ���������Ҫ���룺</br>
converger_process/converger_process.go</br>
����run������������</br>

		convergeTimer := c.clock.NewTimer(c.convergeRepeatInterval)
		cellDisappeared := make(chan services_bbs.CellEvent)
		Ȼ��goroutine����һ����������������һ��ͨ����
		case event := <-events:
			switch event.EventType() {
			case services_bbs.CellDisappeared:
				c.logger.Info("received-cell-disappeared-event", lager.Data{"cell-id": event.CellIDs()})
				select {
				case cellDisappeared <- event:
				case <-done:
					return
				}
			}
			
����ǹ۲�cellsͻȻ��ʧ�ˣ�Ҳ���Ǵ�bbs�У�Ȼ��Ϳ�ʼ����</br>

		case <-cellDisappeared:
					c.converge()
			
�������converge����</br>

		func (c *ConvergerProcess) converge() {
			wg := sync.WaitGroup{}

			wg.Add(1)
			go func() {
				defer wg.Done()
				c.bbsClient.ConvergeTasks(
					c.kickTaskDuration,
					c.expirePendingTaskDuration,
					c.expireCompletedTaskDuration,
				)
			}()

			wg.Add(1)
			go func() {
				defer wg.Done()
				c.bbsClient.ConvergeLRPs()
			}()

			wg.Wait()
		}

���Կ���һ����Tasks��һ����LRPs����Ϊcells���ˣ�����Ҫ���¾ۺ�</br>

https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/client.go#L484</br>

		func (c *client) ConvergeTasks(kickTaskDuration, expirePendingTaskDuration, expireCompletedTaskDuration time.Duration) error {
			request := &models.ConvergeTasksRequest{
				KickTaskDuration:            kickTaskDuration.Nanoseconds(),
				ExpirePendingTaskDuration:   expirePendingTaskDuration.Nanoseconds(),
				ExpireCompletedTaskDuration: expireCompletedTaskDuration.Nanoseconds(),
			}
			response := models.ConvergeTasksResponse{}
			route := ConvergeTasksRoute
			err := c.doRequest(route, nil, nil, request, &response)
			if err != nil {
				return err
			}
			return response.Error.ToError()
		}

�����漰������·�ɣ�</br>

		ConvergeLRPsRoute = "ConvergeLRPs"
		ConvergeTasksRoute = "ConvergeTasks"
		{Path: "/v1/lrps/converge", Method: "POST", Name: ConvergeLRPsRoute},
		{Path: "/v1/tasks/converge", Method: "POST", Name: ConvergeTasksRoute},

https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/handlers/task_handlers.go#L169</br>

		func (h *TaskHandler) ConvergeTasks(w http.ResponseWriter, req *http.Request) {
			var err error
			logger := h.logger.Session("converge-tasks")

			request := &models.ConvergeTasksRequest{}
			response := &models.ConvergeTasksResponse{}

			err = parseRequest(logger, req, request)
			if err == nil {
				h.db.ConvergeTasks(
					logger,
					time.Duration(request.KickTaskDuration),
					time.Duration(request.ExpirePendingTaskDuration),
					time.Duration(request.ExpireCompletedTaskDuration),
				)
			}

			response.Error = models.ConvertError(err)
			writeResponse(w, response)
		}

֮������bbs/db/etcd/task_convergence.go ���ղ�������etcd</br>
������ܸ��ӣ���Ϊ�漰��һЩ<strong>CAS</strong>ԭ��</br>
https://github.com/cloudfoundry-incubator/bbs/blob/d643994d154d7f297da48a887f0df857a5899780/db/etcd/task_convergence.go#L27</br>

1.����cells:</br>

		cellSet, modelErr := cellsLoader.Cells()

2.��ɵ�task�Ľṹ</br>

		tasksToComplete := []*models.Task{}
		scheduleForCompletion := func(task *models.Task) {
			if task.CompletionCallbackUrl == "" {
				return
			}
			tasksToComplete = append(tasksToComplete, task)
		}
		
3.Ȼ���tasksCAS Ҳ����һ�����жϣ���Ҫ��CAS������Ҳ�����ؽ�����</br>

		tasksToCAS := []compareAndSwappableTask{}
		scheduleForCASByIndex := func(index uint64, newTask *models.Task) {
			tasksToCAS = append(tasksToCAS, compareAndSwappableTask{
				OldIndex: index,
				NewTask:  newTask,
			})
		}
		
4.����tasks nodes�ڵ��״̬��</br>
case models.Task_Pending</br>
�������������һ���ǣ�����������Ҫ�����Ϊʧ�ܣ�Ҳ���������޵�ʱ��û������,�����ˣ�����CAS���޸�����
�ڶ������������һֱ������auctionʱ��pending����������񽻸�auction
��ѡ��֮ǰ��һ�����Ҫע�⣺��һ�������ǳ���������״̬��ʱ��</br>

		shouldKickTask := db.durationSinceTaskUpdated(task) >= kickTaskDuration

		shouldMarkAsFailed := db.durationSinceTaskCreated(task) >= expirePendingTaskDuration

		if shouldMarkAsFailed {
			logError(task, "failed-to-start-in-time")
			db.markTaskFailed(task, "not started within time limit")
			scheduleForCASByIndex(node.ModifiedIndex, task)
			tasksKicked++
		} else if shouldKickTask {
			logger.Info("requesting-auction-for-pending-task", lager.Data{"task-guid": task.TaskGuid})
			tasksToAuction = append(tasksToAuction, task)
			tasksKicked++
		}

		case models.Task_Running:
		//��Ϊ��cells��ʧ��������ҪǨ�ƣ�����ֻ��һ�����
		cellIsAlive := cellSet.HasCellID(task.CellId)
		if !cellIsAlive {
			logError(task, "cell-disappeared")
			db.markTaskFailed(task, "cell disappeared before completion")
			scheduleForCASByIndex(node.ModifiedIndex, task)
			tasksKicked++
		}
		
һ������£�task���cells�Ѿ���Ч��˵��cellû�д���ʼCAS���޸ĵ������е�����node����</br>

case models.Task_Completed:
����һЩ�Ѿ���ɵ����񣬵���û���ü�����ģ�Ҳ���ǹ��ڵģ��ڵ㽫�ᱻɾ��
������������һ�ֳ�������״̬�������task�ŵ�scheduleForCompletion�ﴦ��</br>

		shouldDeleteTask := db.durationSinceTaskFirstCompleted(task) >= expireCompletedTaskDuration
		if shouldDeleteTask {
			logError(task, "failed-to-start-resolving-in-time")
			keysToDelete = append(keysToDelete, node.Key)
		} else if shouldKickTask {
			logger.Info("kicking-completed-task", lager.Data{"task-guid": task.TaskGuid})
			scheduleForCompletion(task)
			tasksKicked++
		}

case models.Task_Resolving:</br>
���һ����������Ǵ��ڷ��磬��ǰһ��״̬��ͬ���ǣ���û�б���ɹ������ھ��߽׶Σ�</br>
Ҳ�Ƿ����������һ������Ҫ��ɾ�����ڶ�������Ҫ������scheduleForCompletion,
������������֮ǰ����Ϊ���taskû����ɻ������м�״̬���Ƚ���Completed�Ȼ���ٽ����node���и��»���˵����</br>

		shouldDeleteTask := db.durationSinceTaskFirstCompleted(task) >= expireCompletedTaskDuration
		if shouldDeleteTask {
			logError(task, "failed-to-resolve-in-time")
			keysToDelete = append(keysToDelete, node.Key)
		} else if shouldKickTask {
			logger.Info("demoting-resolving-to-completed", lager.Data{"task-guid": task.TaskGuid})
			demoted := demoteToCompleted(task)
			scheduleForCASByIndex(node.ModifiedIndex, demoted)
			scheduleForCompletion(demoted)
			tasksKicked++
		}
		
����https://github.com/cloudfoundry-incubator/bbs/blob/master/db/etcd/task_convergence.go#L213</br>

		accumulateZonesByInstances//�������������completed��״̬
		httpsfunc demoteToCompleted(task *models.Task) *models.Task {
			task.State = models.Task_Completed
			return task
		}

��ʱ�����е�״̬��¼���Ѿ�������ɣ�</br>
running������ύ��auctionner�ͻ��˴���</br>
db.auctioneerClient.RequestTaskAuctions(tasksToAuction)</br>

����tasktoCAS������ֱ���������ˣ������tasksToCAS��scheduleForCASByIndex</br>

		tasksKickedCounter.Add(tasksKicked)
		err := db.batchCompareAndSwapTasks(tasksToCAS, logger)
		
���������cf��֯ר��Ϊ��д��һ���̳߳صĹ���workpool,��Ҫ��task��node�ڵ��ʱ����£�Ȼ�����taskToCAS.OldIndex����task�ĶԱ�</br>

		for _, taskToCAS := range tasksToCAS {
			task := taskToCAS.NewTask
			task.UpdatedAt = db.clock.Now().UnixNano()
			value, err := db.serializeModel(logger, task)
			if err != nil {
				logger.Error("failed-to-marshal", err, lager.Data{
					"task-guid": task.TaskGuid,
				})
				continue
			}

			index := taskToCAS.OldIndex
			works = append(works, func() {
				_, err := db.client.CompareAndSwap(TaskSchemaPathByGuid(task.TaskGuid), value, NO_TTL, index)
				if err != nil {
					logger.Error("failed-to-compare-and-swap", err, lager.Data{
						"task-guid": task.TaskGuid,
					})
				}
			})
		}

��һ�����CompareAndSwap����ʵ��һ��ԭ�Ӳ���</br>

		func (sc *storeClient) CompareAndSwap(key string, payload []byte, ttl uint64, prevIndex uint64) (*etcd.Response, error) {
			return sc.client.CompareAndSwap(key, string(payload), ttl, "", prevIndex)
		}
		
����go-etcd��ᷢ�ֻ�У֮ǰ��prevValue���бȽϣ�����Ͳ������ˣ�</br>

		func (c *Client) RawCompareAndSwap(key string, value string, ttl uint64,
			prevValue string, prevIndex uint64) (*RawResponse, error) {
			if prevValue == "" && prevIndex == 0 {
				return nil, fmt.Errorf("You must give either prevValue or prevIndex.")
			}

			options := Options{}
			if prevValue != "" {
				options["prevValue"] = prevValue
			}
			if prevIndex != 0 {
				options["prevIndex"] = prevIndex
			}

			raw, err := c.put(key, value, ttl, options)

			if err != nil {
				return nil, err
			}

			return raw, err
		}

Ȼ�����ύ��ɵ�tasks</br>

		for _, task := range tasksToComplete {
			db.taskCompletionClient.Submit(db, task)
		}
		tasksPrunedCounter.Add(uint64(len(keysToDelete)))

����ʣ����Ҫ�������Ч������</br>

		db.batchDeleteTasks(keysToDelete, logger)



