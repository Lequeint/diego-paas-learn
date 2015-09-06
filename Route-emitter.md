# route-emitter ����(����)
���濪ʼ��route-emitter���ֿ�ʼ��ҪԴ�������������ڷ������⣬����ڸ� <br />

route-emitterһ����1��������
-----------------------------------
˵����<br />
��������nats�˵�·��ע��ͷ�ע����Ϣ��Ȼ��ͨ����Щ��Ϣע������·�ɱ�<br />

### Route-emitter�������
�ȿ���������������<br />

		/var/vcap/packages/route_emitter/bin/route-emitter \
		-consulCluster=http://127.0.0.1:8500 \
		-natsAddresses=10.244.0.6:4222 \
		-natsUsername=nats \
		-natsPassword=nats \
		-diegoAPIURL=http://:@receptor.service.cf.internal:8887 \
		-debugAddr=0.0.0.0:17009 \
		-syncInterval=60s \
		-logLevel=debug
		
ֱ�ӿ����ģ�watcher/watcher.go<br />

����watcher�Ķ�����Ҫ��������һ����<strong>DesiredLrp</strong>,һ����<strong>ActualLrp</strong><br />

//�¼����� ��tps�ﶨ������ƣ�������Щ�¼����Ǵ�nats��ȡ���¼�Դ��һ��<br />

		func (watcher *Watcher) handleEvent(logger lager.Logger, event receptor.Event) {
			switch event := event.(type) {
			//Ӧ���嵥�����¼�
			case receptor.DesiredLRPCreatedEvent:
				watcher.handleDesiredCreate(logger, event.DesiredLRPResponse)
			//Ӧ���嵥�仯�¼�
			case receptor.DesiredLRPChangedEvent:
				watcher.handleDesiredUpdate(logger, event.Before, event.After)
			//Ӧ���嵥�Ƴ��¼�����ʵ��������Ӧ�ñ�ɾ��
			case receptor.DesiredLRPRemovedEvent:
				watcher.handleDesiredDelete(logger, event.DesiredLRPResponse)
			//Ӧ������ʵ�������¼�
			case receptor.ActualLRPCreatedEvent:
				watcher.handleActualCreate(logger, event.ActualLRPResponse)
			//Ӧ������ʵ���仯�¼�
			case receptor.ActualLRPChangedEvent:
				watcher.handleActualUpdate(logger, event.Before, event.After)
			//Ӧ������ʵ�����Ƴ��¼�
			case receptor.ActualLRPRemovedEvent:
				watcher.handleActualDelete(logger, event.ActualLRPResponse)
			default:
				logger.Info("did-not-handle-unrecognizable-event", lager.Data{"event-type": event.EventType()})
			}
		}

--->��λ��func (watcher *Watcher) Run(signals <-chan os.Signal, ready chan<- struct{}) error <br />

���ȶ���������ͨ����<br />

		eventChan := make(chan receptor.Event)
		syncEndChan := make(chan syncEndEvent)
		
		var eventSource atomic.Value
		var stopEventSource int32
	
�����������Ĵ���飺<br />

		//ͬ���¼�
		for {
			select {
			case <-watcher.syncEvents.Sync:
				if syncing == false {
					logger := watcher.logger.Session("sync")
					logger.Info("starting")
					syncing = true

					if !startedEventSource {
						startedEventSource = true
						startEventSource() 
					}

					cachedEvents = make(map[string]receptor.Event)
					go watcher.sync(logger, syncEndChan)
				}

			case syncEnd := <-syncEndChan:
				watcher.completeSync(syncEnd, cachedEvents)
				cachedEvents = nil
				syncing = false
				syncEnd.logger.Info("complete")

			case <-watcher.syncEvents.Emit:
				logger := watcher.logger.Session("emit")
				watcher.emit(logger)

			case event := <-eventChan:
				if syncing {
					watcher.logger.Info("caching-event", lager.Data{
						"type": event.EventType(),
					})

					cachedEvents[event.Key()] = event
				} else {
					watcher.handleEvent(watcher.logger, event)
				}

			case <-signals:
				watcher.logger.Info("stopping")
				atomic.StoreInt32(&stopEventSource, 1)
				if es := eventSource.Load(); es != nil {
					err := es.(receptor.EventSource).Close()
					if err != nil {
						watcher.logger.Error("failed-closing-event-source", err)
					}
				}
				return nil
			}
		}
	
--->����startedEventSource = true startEventSource()�������Ǹ���������<br />
startEventSource := func(){}����ȥ��������ľ�����Ϣ��<br />
	
		����ִ����һ��goroutine,������
		���ϵĴ����������Ϣ��������洢������
		es, err = watcher.receptorClient.SubscribeToEvents()
		eventSource.Store(es)
		
		�ڲ���һ��ѭ������event����eventChanͨ����
		if event != nil {
			eventChan <- event
		}
		
		ִ����startEventSource()�󣬽���������¼����洢������
		cachedEvents = make(map[string]receptor.Event)
		���ִ��syncͬ������
		go watcher.sync(logger, syncEndChan)
		���뵽sync������ȥ��
		var runningActualLRPs []receptor.ActualLRPResponse
		var getActualLRPsErr error
		var desiredLRPs []receptor.DesiredLRPResponse
		var getDesiredLRPsErr error
		��������Կ���������desiredLRPs��ActualLRPs������running��
		����runnintActualLRPs
		go func() {
			defer wg.Done()

			logger.Debug("getting-actual-lrps")
			actualLRPResponses, err := watcher.receptorClient.ActualLRPs()
			if err != nil {
				logger.Error("failed-getting-actual-lrps", err)
				getActualLRPsErr = err
				return
			}
			logger.Debug("succeeded-getting-actual-lrps", lager.Data{"num-actual-responses": len(actualLRPResponses)})

			runningActualLRPs = make([]receptor.ActualLRPResponse, 0, len(actualLRPResponses))
			for _, actualLRPResponse := range actualLRPResponses {
				if actualLRPResponse.State == receptor.ActualLRPStateRunning {
					runningActualLRPs = append(runningActualLRPs, actualLRPResponse)
				}
			}
		}()
		
�����ԵĿ�������ͨ��ִ��һ������goroutine������ͨ��receptorClient.ActualLRPs()��ȡactualLRPs���̶��ж���state�Ƿ���running�����running״̬��actualLRP���뵽runningActualLRPs��<br />
	
--->�ټ�����desiredLRPs<br />

		go func() {
			defer wg.Done()

			logger.Debug("getting-desired-lrps")
			desiredLRPResponses, err := watcher.receptorClient.DesiredLRPs()
			if err != nil {
				logger.Error("failed-getting-desired-lrps", err)
				getDesiredLRPsErr = err
				return
			}
			logger.Debug("succeeded-getting-desired-lrps", lager.Data{"num-desired-responses": len(desiredLRPResponses)})

			desiredLRPs = make([]receptor.DesiredLRPResponse, 0, len(desiredLRPResponses))
			for _, desiredLRPResponse := range desiredLRPResponses {
				desiredLRPs = append(desiredLRPs, desiredLRPResponse)
			}
		}()
		
����Ҳ������ͨ��receptorClient.DesiredLRPs��ȡ���е�desiredLRPs��Ȼ��������뵽desiredLRPs��<br />
	
�������һ��newTable<br />

		newTable := routing_table.NewTempTable(
			routing_table.RoutesByRoutingKeyFromDesireds(desiredLRPs),
			routing_table.EndpointsByRoutingKeyFromActuals(runningActualLRPs),
		)
	
��������һ��desiredLRPs ��handler����־��������Ҫչ����һ��appʵ������չ������ private_key̫�����ҿ�����<br />
		{
		"timestamp": "1441088698.029039621", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-desired-update.starting", 
		"log_level": 1, 
		"data": {
		"after": {
		  "ports": [
			8080, 
			2222
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
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
		  }
		}, 
		"before": {
		  "ports": [
			8080, 
			2222
		  ], 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6", 
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
		  }
		}, 
		"session": "5.2695"
		}
		}

������Կ���������·�ɱ��ǰ��Աȣ����ֻ���޸�ʵ��������һ�㲻����ʲô�仯<br />
����һ�㻹�� �����������ݶ����<br />

	   route-emitter.watcher.handling-desired-update.emitting-messages 
	   route-emitter.watcher.handling-desired-update.complet
   
��һ�ᣬ�嵥����cells�ﻹ��Ҫһ���¼�������ʵ��Ҳ����actualLRPs,ע��۲�before��after<br />
   
		{
		"timestamp": "1441088706.596357346", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-actual-update.starting", 
		"log_level": 1, 
		"data": {
		"after": {
		  "address": "10.244.16.138", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
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
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"before": {
		  "address": "", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": null, 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"session": "5.2699"
		}
		}
   
��ʱ���Ǿ��ܿ���ʵ��������������ôӳ����ˣ���Ҳ����diego��������LRPs�ľ����<br />

		{
		"timestamp": "1441088706.596627951", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.handling-actual-update.emitting-messages", 
		"log_level": 0, 
		"data": {
		"after": {
		  "address": "10.244.16.138", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
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
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"before": {
		  "address": "", 
		  "cell-id": "cell_z1-0", 
		  "domain": "cf-apps", 
		  "evacuating": false, 
		  "index": 1, 
		  "instance-guid": "21811b3c-a37e-4277-5878-c646a7213bf0", 
		  "ports": null, 
		  "process-guid": "c181be6b-d89c-469c-b79c-4cab1e99b8de-6eac776b-f051-4cb0-9614-5a3954d1bfd6"
		}, 
		"messages": {
		  "RegistrationMessages": [
			{
			  "host": "10.244.16.138", 
			  "port": 60002, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
			}
		  ], 
		  "UnregistrationMessages": null
		}, 
		"session": "5.2699"
		}
		}
		
��һ��nats��ע����Ϣ��<br />

		{
		"timestamp": "1441088706.596804857", 
		"source": "route-emitter", 
		"message": "route-emitter.nats-emitter.emit", 
		"log_level": 0, 
		"data": {
		"message": {
		  "host": "10.244.16.138", 
		  "port": 60002, 
		  "uris": [
			"helloDocker.10.244.0.34.xip.io"
		  ], 
		  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
		  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
		}, 
		"session": "3", 
		"subject": "router.register"
		}
		}

ֱ������route-emitter.watcher.handling-actual-update.complete ���ʵ���������Ĵ����ɹ�<br />
	
--->���������絽watcher���֣�����·�ɱ�<br />
		{
		"timestamp": "1441088716.608674765", 
		"source": "route-emitter", 
		"message": "route-emitter.watcher.emit.emitting-messages", 
		"log_level": 0, 
		"data": {
		"messages": {
		  "RegistrationMessages": [
			{
			  "host": "10.244.16.138", 
			  "port": 60000, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"
			}, 
			{
			  "host": "10.244.16.138", 
			  "port": 60002, 
			  "uris": [
				"helloDocker.10.244.0.34.xip.io"
			  ], 
			  "app": "c181be6b-d89c-469c-b79c-4cab1e99b8de", 
			  "private_instance_id": "21811b3c-a37e-4277-5878-c646a7213bf0"
			}
		  ], 
		  "UnregistrationMessages": null
		}, 
		"session": "5.2701"
		}
		}

		{"timestamp":"1441088716.608885765","source":"route-emitter","message":"route-emitter.nats-emitter.emit","log_level":0,"data":{"message":{"host":"10.244.16.138","port":60000,"uris":["helloDocker.10.244.0.34.xip.io"],"app":"c181be6b-d89c-469c-b79c-4cab1e99b8de","private_instance_id":"fc4a83cd-5ed9-40fb-51bf-1679beb41bfe"},"session":"3","subject":"router.register"}}
		{"timestamp":"1441088716.609278917","source":"route-emitter","message":"route-emitter.nats-emitter.emit","log_level":0,"data":{"message":{"host":"10.244.16.138","port":60002,"uris":["helloDocker.10.244.0.34.xip.io"],"app":"c181be6b-d89c-469c-b79c-4cab1e99b8de","private_instance_id":"21811b3c-a37e-4277-5878-c646a7213bf0"},"session":"3","subject":"router.register"}}

�ٴο���watcher �Ѿ�����actuallrps�Ѿ����2��<br />

		{
			"timestamp": "1441088716.663050652", 
			"source": "route-emitter", 
			"message": "route-emitter.watcher.sync.succeeded-getting-actual-lrps", 
			"log_level": 0, 
			"data": {
				"num-actual-responses": 2, 
				"session": "5.2700"
			}
		}
	
��desiredlrps����1��˵������������<br />

		{
			"timestamp": "1441088716.664330006", 
			"source": "route-emitter", 
			"message": "route-emitter.watcher.sync.succeeded-getting-desired-lrps", 
			"log_level": 0, 
			"data": {
				"num-desired-responses": 1, 
				"session": "5.2700"
			}
		}
	
������˽�touteTable�뿴��routing_table��һ���ֵĴ���