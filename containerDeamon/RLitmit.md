# RLimit garden-linux ˵��
garden������ÿ��Ӧ�õ�ʱ�򶼻������м���һ���ػ�����wsh,���������warden����Ѵ��ڣ��������Ĳ�����������ͨ������ͨ�ŵģ�˵���˾��ǽ��̼�ͨ��socket

## Garden-linux ������ε��������е����ܣ�
container_daemon:Process -> container_daemon:proc_starter -> linux_container:running -> container_daemon:rlimits_manager </br>

����linux_container:running��</br>
���ȿ���wsh��һЩ�еĴ���������ָ���û��ͻ���������ʼ��wsh������Դ�������� </br>

		func (c *LinuxContainer) Run(spec garden.ProcessSpec, processIO garden.ProcessIO) (garden.Process, error) {
			wshPath := path.Join(c.ContainerPath, "bin", "wsh")
			sockPath := path.Join(c.ContainerPath, "run", "wshd.sock")

			if spec.User == "" {
				c.logger.Error("linux_container: Run:", errors.New("linux_container: Run: A User for the process to run as must be specified."))
				return nil, errors.New("A User for the process to run as must be specified.")
			}

			args := []string{"--socket", sockPath, "--user", spec.User}

			specEnv, err := process.NewEnv(spec.Env)
			if err != nil {
				return nil, err
			}

			procEnv, err := process.NewEnv(c.Env)
			if err != nil {
				return nil, err
			}

			processEnv := procEnv.Merge(specEnv)

			for _, envVar := range processEnv.Array() {
				args = append(args, "--env", envVar)
			}

			if spec.Dir != "" {
				args = append(args, "--dir", spec.Dir)
			}

			processID := c.processIDPool.Next()
			c.logger.Info("next pid", lager.Data{"pid": processID})

			if c.Version.Compare(MissingVersion) == 0 {
				pidfile := path.Join(c.ContainerPath, "processes", fmt.Sprintf("%d.pid", processID))
				args = append(args, "--pidfile", pidfile)
			}

			args = append(args, spec.Path)

			wsh := exec.Command(wshPath, append(args, spec.Args...)...)
			
			//�����ǶԸ��������������Ƶĵط�
			setRLimitsEnv(wsh, spec.Limits)

			return c.processTracker.Run(processID, wsh, processIO, spec.TTY, c.processSignaller())
		}
		
����container_daemon:rlimits_manager����rlimits����Щ����������</br>
		
		rLimitsMap := map[string]*rlimitEntry{
			"cpu":        &rlimitEntry{Id: RLIMIT_CPU, Max: RLIMIT_INFINITY},
			"fsize":      &rlimitEntry{Id: RLIMIT_FSIZE, Max: RLIMIT_INFINITY},
			"data":       &rlimitEntry{Id: RLIMIT_DATA, Max: RLIMIT_INFINITY},
			"stack":      &rlimitEntry{Id: RLIMIT_STACK, Max: RLIMIT_INFINITY},
			"core":       &rlimitEntry{Id: RLIMIT_CORE, Max: RLIMIT_INFINITY},
			"rss":        &rlimitEntry{Id: RLIMIT_RSS, Max: RLIMIT_INFINITY},
			"nproc":      &rlimitEntry{Id: RLIMIT_NPROC, Max: RLIMIT_INFINITY},
			"nofile":     &rlimitEntry{Id: RLIMIT_NOFILE, Max: maxNoFile},
			"memlock":    &rlimitEntry{Id: RLIMIT_MEMLOCK, Max: RLIMIT_INFINITY},
			"as":         &rlimitEntry{Id: RLIMIT_AS, Max: RLIMIT_INFINITY},
			"locks":      &rlimitEntry{Id: RLIMIT_LOCKS, Max: RLIMIT_INFINITY},
			"sigpending": &rlimitEntry{Id: RLIMIT_SIGPENDING, Max: RLIMIT_INFINITY},
			"msgqueue":   &rlimitEntry{Id: RLIMIT_MSGQUEUE, Max: RLIMIT_INFINITY},
			"nice":       &rlimitEntry{Id: RLIMIT_NICE, Max: RLIMIT_INFINITY},
			"rtprio":     &rlimitEntry{Id: RLIMIT_RTPRIO, Max: RLIMIT_INFINITY},
		}
		
Ϊ�˺���ĵ��ţ��ֱ�˵һ����д���ε�����:</br>

**RLIMIT_CPU:** ��������CPUʹ��ʱ�䣬��Ϊ��λ�������̴ﵽ�����ƣ��ں˽����䷢��SIGXCPU�źţ���һ�źŵ�Ĭ����Ϊ����ֹ���̵�ִ�С�Ȼ�������Բ�׽�źţ��������ɽ����Ʒ��ظ�������������̼����ķ�CPUʱ�䣬���Ļ���ÿ��һ�ε�Ƶ�ʸ��䷢��SIGXCPU�źţ�ֱ���ﵽӲ���ƣ���ʱ�������̷��� SIGKILL�ź���ֹ��ִ�С�</br>
**RLIMIT_FSIZE:** ���Դ������ļ�������ֽڳ��ȡ���������������ʱ������ý��̷���SIGXFSZ�źš�</br>
**RLIMIT_DATA:** �������ݶε����ֵ </br>
**RLIMIT_STACK:** ���Ľ��̶�ջ�����ֽ�Ϊ��λ Ĭ��Ϊ8M </br>
**RLIMIT_CORE:** �ں�ת���ļ�����󳤶� </br>
**RLIMIT_RSS:**  </br>
**RLIMIT_NPROC:** �û���ӵ�е��������� </br>
**RLIMIT_NOFILE:** ÿ�������ܴ򿪵�����ļ�����������ֵ���������EMFILE���� </br>
**RLIMIT_MEMLOCK:** ���̿��������ڴ��е�������������ֽ�Ϊ��λ </br>
**RLIMIT_AS:** ���̵�������ڴ�ռ䣬�ֽ�Ϊ��λ </br>
**RLIMIT_LOCKS:** ���̿ɽ������������޵����ֵ </br>
**RLIMIT_SIGPENDING:** �û���ӵ�е��������ź��� </br>
**RLIMIT_MSGQUEUE:** ���̿�ΪPOSIX��Ϣ���з��������ֽ��� </br>
**RLIMIT_NICE:** ���̿�ͨ��setpriority() �� nice()�������õ��������ֵ </br>
**RLIMIT_RTPRIO:** ���̿�ͨ��sched_setscheduler �� sched_setparam���õ����ʵʱ���ȼ� </br>

* Ĭ�����ã�</br>

		root@ubuntu:~# ulimit -a
		core file size          (blocks, -c) 0
		data seg size           (kbytes, -d) unlimited
		scheduling priority             (-e) 0
		file size               (blocks, -f) unlimited
		pending signals                 (-i) 11727
		max locked memory       (kbytes, -l) 64
		max memory size         (kbytes, -m) unlimited
		open files                      (-n) 1024
		pipe size            (512 bytes, -p) 8
		POSIX message queues     (bytes, -q) 819200
		real-time priority              (-r) 0
		stack size              (kbytes, -s) 8192
		cpu time               (seconds, -t) unlimited
		max user processes              (-u) 11727
		virtual memory          (kbytes, -v) unlimited
		file locks                      (-x) unlimited
		
* ϵͳ��Ӳ�Աȣ�</br>

		root@ubuntu:~# ulimit -c -n -s
		core file size          (blocks, -c) 0
		open files                      (-n) 1024
		stack size              (kbytes, -s) 8192
		
		root@ubuntu:~# ulimit -c -n -s -H
		core file size          (blocks, -c) unlimited
		open files                      (-n) 4096
		stack size              (kbytes, -s) unlimited
		
32λ��linuxϵͳ�Ĭ���û��ռ�Ϊ3G��Ĭ��ջΪ8M������3096/8=384���̣߳�Ҳ����˵Ĭ��ÿ�����̿��Դ���384���̣߳�����κ����ݶεȻ�Ҫռ��һЩ�ռ䣬���߳���һ���̣߳�����һ�����Դ���382���̡߳�</br>
Ϊ��ͻ���ڴ�����ƣ����������ַ���</br>
**1)** �� ulimit -s 1024 ��СĬ�ϵ�ջ��С</br>
**2)** ���� pthread_create ��ʱ���� pthread_attr_getstacksize ����һ����С��ջ��С</br>
Ҫע����ǣ���ʹ������Ҳ�޷�ͻ��**1024**���̵߳�Ӳ���ƣ��������±��� C ��,ulimit -s 1024������Եõ�3054���߳�</br>

��Linux�ں�2.2.x���������������޸ģ� </br>

		echo ��8192�� > /proc/sys/fs/file-max
		echo ��32768�� > /proc/sys/fs/inode-max 

������������ӵ�/etc/rc.c/rc.local�ļ��У���ʹϵͳÿ����������ʱ��������ֵ�� </br>
��Linux�ں�2.4.x����Ҫ�޸�ԭʼ�룬Ȼ�����±����ں˲���Ч���༭Linux�ں�ԭʼ���е� include/linux/fs.h�ļ����� NR_FILE ��8192��Ϊ 65536����NR_RESERVED_FILES ��10 ��Ϊ 128���༭fs/inode.c �ļ��� MAX_INODE ��16384��Ϊ262144�� </br>
һ������£������ļ����ȽϺ��������Ϊÿ4M�����ڴ�256������256M�ڴ�����Ϊ16384��������ʹ�õ�i�ڵ����ĿӦ���������ļ���Ŀ��3����4����

### PS:
LINUX�н��̵�������������㣺</br>

ÿ�����̵ľֲ���������LDT����Ϊһ�������Ķζ����ڣ���ȫ�ֶ�������GDT��Ҫ��һ������ָ������ε���ʼ��ַ����˵���öεĳ����Լ�����һЩ ����������֮�⣬ÿ�����̻���һ��TSS�ṹ(����״̬��)Ҳ��һ�������ԣ�ÿ�����̶�Ҫ��ȫ�ֶ�������GDT��ռ�����������ô��GDT�������ж�� �أ��μĴ���������GDT���±��λ�ο����13λ������GDT�п�����8192���������һЩϵͳ�Ŀ���(����GDT�еĵ�2��͵�3��ֱ������ں� �Ĵ���κ����ݶΣ���4��͵�5����Զ���ڵ�ǰ���̵Ĵ���κ����ݶΣ���1����Զ��0���ȵ�)���⣬����8180������ɹ�ʹ�ã�����������ϵͳ������ ����������4090��
</br>
���һ��warden�еĽ�����Դ����:</br>

		container_rlimits:
		  as: 4294967296 (4G)
		  nofile: 8192 (ÿ����������ܴ�8192���ļ�)
		  nproc: 512 (�û���ӵ�����512������)